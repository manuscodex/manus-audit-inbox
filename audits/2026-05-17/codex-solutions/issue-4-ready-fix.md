# Codex ready fix — Issue #4

Target: GGC Gaming SEO/head/404 cleanup after the Lottery + SEO batch.

This is a concrete implementation package for Manus to apply in the real GGC source repo. Codex cannot push a real PR yet because the connected GitHub app currently has access only to `manuscodex/manus-audit-inbox`, not the GGC source repo.

## Current live re-check

Checked on `2026-05-17` after the first audit comment:

- `https://gg.world/` has `og:image`, `twitter:image`, and 3 `application/ld+json` scripts.
- `https://gg.world/` still has 2 canonical tags: `https://gg.world` and `https://gg.world/`.
- `https://gg.world/crash` still has 2 canonical tags: `https://gg.world` and `https://gg.world/crash`.
- `https://gg.world/this-should-404` returns HTTP `200`, has `robots=index, follow`, and has 2 canonical tags.

## Fix 1 — remove static canonical from `client/index.html`

In `client/index.html`, delete the hardcoded canonical line:

```diff
-    <link rel="canonical" href="https://gg.world" />
```

Reason: the server/runtime already injects a route-specific canonical. Keeping a static homepage canonical in the base SPA template creates duplicates on every route.

Keep the existing `og:image`, `twitter:image`, OG/Twitter description tags and JSON-LD scripts.

Recommended small cleanup in the same file:

```diff
-    <meta property="og:url" content="https://gg.world" />
+    <meta property="og:url" content="https://gg.world/" />
```

## Fix 2 — route whitelist for real 404 status

Add this helper near the server bootstrap, preferably in a small file such as `server/seoRoutes.ts`.

```ts
export const PUBLIC_APP_ROUTES = new Set([
  "/",
  "/coinflip",
  "/dice",
  "/wheel",
  "/crash",
  "/plinko",
  "/mines",
  "/limbo",
  "/one-dice",
  "/rps",
  "/lottery",
  "/lottery/mega",
  "/lottery/power",
  "/history",
  "/provably-fair",
  "/staking",
]);

export function normalizeAppRoute(pathname: string): string {
  if (!pathname || pathname === "/") return "/";
  const withoutTrailingSlash = pathname.replace(/\/+$/, "");
  return withoutTrailingSlash || "/";
}

export function isKnownPublicRoute(pathname: string): boolean {
  return PUBLIC_APP_ROUTES.has(normalizeAppRoute(pathname));
}

export function canonicalForPath(pathname: string): string {
  const route = normalizeAppRoute(pathname);
  return route === "/" ? "https://gg.world/" : `https://gg.world${route}`;
}

export function shouldSkipSpaSeo(pathname: string): boolean {
  return (
    pathname.startsWith("/api/") ||
    pathname.startsWith("/assets/") ||
    pathname.startsWith("/storage/") ||
    pathname === "/favicon.ico" ||
    pathname === "/robots.txt" ||
    pathname === "/sitemap.xml" ||
    /\.[a-z0-9]{2,8}$/i.test(pathname)
  );
}
```

## Fix 3 — set 404 before the SPA fallback serves `index.html`

In `server/_core/index.ts`, before `setupVite(app, server)` / `serveStatic(app)` and after API routes that should remain untouched, add middleware like this:

```ts
import { canonicalForPath, isKnownPublicRoute, shouldSkipSpaSeo } from "../seoRoutes";

app.use((req, res, next) => {
  if (req.method !== "GET" && req.method !== "HEAD") return next();
  if (shouldSkipSpaSeo(req.path)) return next();

  const canonical = canonicalForPath(req.path);
  const isKnownRoute = isKnownPublicRoute(req.path);

  res.locals.seo = {
    canonical,
    robots: isKnownRoute ? "index, follow" : "noindex, follow",
  };

  if (!isKnownRoute) {
    res.status(404);
  }

  next();
});
```

Important: this must run before the SPA fallback returns `index.html`. Express will preserve `res.status(404)` when the fallback sends the HTML, unless the fallback explicitly resets it.

## Fix 4 — strip all existing canonical tags before injecting the final one

In the server HTML transform that injects route-specific head tags, make canonical injection idempotent. If the project has this logic in `server/_core/vite.ts`, update that transform. If it is elsewhere, apply the same logic there.

```ts
function escapeHtmlAttr(value: string): string {
  return value
    .replaceAll("&", "&amp;")
    .replaceAll('"', "&quot;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;");
}

export function injectSeoHead(html: string, seo: { canonical: string; robots: string }): string {
  const cleanHtml = html
    .replace(/\s*<link\s+rel=["']canonical["'][^>]*>\s*/gi, "\n")
    .replace(/\s*<meta\s+name=["']robots["'][^>]*>\s*/gi, "\n");

  const tags = [
    `<meta name="robots" content="${escapeHtmlAttr(seo.robots)}" />`,
    `<link rel="canonical" href="${escapeHtmlAttr(seo.canonical)}" />`,
  ].join("\n    ");

  return cleanHtml.replace("</head>", `    ${tags}\n  </head>`);
}
```

Then call it when returning the SPA HTML:

```ts
const seo = res.locals.seo ?? {
  canonical: "https://gg.world/",
  robots: "index, follow",
};

html = injectSeoHead(html, seo);
```

If the existing transform already injects title/meta/canonical per path, keep that logic, but first remove every pre-existing canonical and robots tag from the HTML string, then inject exactly one final canonical and one final robots tag.

## Fix 5 — settlement endpoint must be guarded before cron registration

Do not register heartbeat schedules until `server/lotterySettleHandler.ts` is included in the next audit package. Minimum guard expected at the top of the handler:

```ts
function timingSafeEqualString(a: string, b: string): boolean {
  const encoder = new TextEncoder();
  const left = encoder.encode(a);
  const right = encoder.encode(b);
  if (left.length !== right.length) return false;

  let diff = 0;
  for (let i = 0; i < left.length; i += 1) {
    diff |= left[i] ^ right[i];
  }
  return diff === 0;
}

export async function lotterySettleHandler(req: Request, res: Response) {
  const expectedSecret = process.env.LOTTERY_SETTLE_SECRET;
  const providedSecret = req.header("x-ggc-scheduler-secret") ?? "";

  if (!expectedSecret || !timingSafeEqualString(providedSecret, expectedSecret)) {
    return res.status(401).json({ ok: false, error: "unauthorized" });
  }

  // Continue with idempotent import/settlement only after auth passes.
}
```

If this is an Express handler, use the existing `Request`/`Response` imports from `express`.

Required idempotency behavior before cron enablement:

- one settlement key per `lotteryKey + drawDate + scheduleWindow`,
- retry-safe update transaction,
- no duplicate on-chain settlement for the same order,
- structured JSON response with `imported`, `settled`, `skipped`, `errors` counts.

## Validation commands after applying fixes

```bash
npx tsc --noEmit
npx vitest run
npx vite build
```

Live verification after deploy:

```bash
curl -L -s -o /tmp/gg-home.html https://gg.world/
curl -L -s -o /tmp/gg-crash.html https://gg.world/crash
curl -L -s -o /tmp/gg-404.html -w '%{http_code}\n' https://gg.world/this-should-404

rg -o '<link rel="canonical"[^>]*>|<meta name="robots"[^>]*>|<meta property="og:image"[^>]*>|<meta name="twitter:image"[^>]*>|<script type="application/ld\+json"' /tmp/gg-home.html
rg -o '<link rel="canonical"[^>]*>|<meta name="robots"[^>]*>|<meta property="og:image"[^>]*>|<meta name="twitter:image"[^>]*>|<script type="application/ld\+json"' /tmp/gg-crash.html
rg -o '<link rel="canonical"[^>]*>|<meta name="robots"[^>]*>' /tmp/gg-404.html
```

Acceptance criteria:

- `/` returns 200 and exactly one canonical: `https://gg.world/`.
- `/crash` returns 200 and exactly one canonical: `https://gg.world/crash`.
- `/this-should-404` returns HTTP 404.
- `/this-should-404` has `robots=noindex, follow` if it returns an HTML fallback body.
- `/` has one `og:image`, one `twitter:image`, and 3 JSON-LD scripts.
- No heartbeat/cron schedules are registered until `server/lotterySettleHandler.ts` is audited.
