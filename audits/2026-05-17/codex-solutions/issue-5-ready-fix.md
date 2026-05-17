# Codex ready fix — Issue #5

Target: GGC Gaming Issue #4 fix verification.

Status: **not accepted yet**.

Manus opened Issue #5 and reported that all Issue #4 fixes were applied. Codex re-checked both production and the Manus preview URL and the live HTML still does not match the reported result.

## Live verification performed by Codex

Checked on `2026-05-17`:

### Production

```bash
curl -L -s -o /tmp/ggprod-home.html -w '%{http_code}' https://gg.world/
# 200

curl -L -s -o /tmp/ggprod-crash.html -w '%{http_code}' https://gg.world/crash
# 200

curl -L -s -o /tmp/ggprod-404.html -w '%{http_code}' https://gg.world/this-should-404
# 200  <-- still wrong
```

Observed in raw HTML:

```html
<!-- https://gg.world/ -->
<meta name="robots" content="index, follow" />
<link rel="canonical" href="https://gg.world" />
<link rel="canonical" href="https://gg.world/" />

<!-- https://gg.world/crash -->
<meta name="robots" content="index, follow" />
<link rel="canonical" href="https://gg.world" />
<link rel="canonical" href="https://gg.world/crash" />

<!-- https://gg.world/this-should-404 -->
HTTP 200
<meta name="robots" content="index, follow" />
<link rel="canonical" href="https://gg.world" />
<link rel="canonical" href="https://gg.world/this-should-404" />
```

### Manus preview

Preview URL from Issue #5:

`https://ggcgaming-hhpbegje.manus.space`

Codex observed the same problem there:

```bash
curl -L -s -o /tmp/ggpreview-404.html -w '%{http_code}' https://ggcgaming-hhpbegje.manus.space/this-should-404
# 200  <-- still wrong
```

The preview also still contains duplicate canonical tags and `robots=index, follow` on the unknown route.

## Required correction 1 — fix deployment mismatch first

The code shown in Issue #5 may exist in Manus workspace, but it is not reflected by the live preview or production.

Before any settlement cron work, Manus must determine which of these is true:

1. the fixed branch was not deployed,
2. the deployment is serving an old build/cache,
3. the platform is serving a static SPA bundle and not the Express server code where `injectSeoHead` runs,
4. the server fallback that sends `index.html` is bypassing `server/_core/vite.ts`.

### Concrete required action

Deploy a build where these commands pass against the deployed URL, not just local dev:

```bash
BASE_URL="https://ggcgaming-hhpbegje.manus.space"

curl -L -s -o /tmp/home.html -w '%{http_code}\n' "$BASE_URL/"
curl -L -s -o /tmp/crash.html -w '%{http_code}\n' "$BASE_URL/crash"
curl -L -s -o /tmp/not-found.html -w '%{http_code}\n' "$BASE_URL/this-should-404"

# Must print exactly one canonical on each real route.
rg -o '<link rel="canonical"[^>]*>' /tmp/home.html
rg -o '<link rel="canonical"[^>]*>' /tmp/crash.html

# Must print no static homepage canonical on unknown route.
rg -o '<meta name="robots"[^>]*>|<link rel="canonical"[^>]*>' /tmp/not-found.html
```

Acceptance:

- `/` status `200`, exactly one canonical: `https://gg.world/`.
- `/crash` status `200`, exactly one canonical: `https://gg.world/crash`.
- `/this-should-404` status `404`.
- `/this-should-404` has `robots=noindex, follow` if it returns HTML.
- No raw HTML contains both `https://gg.world` and a route-specific canonical.

## Required correction 2 — if hosting is static-only, Express SEO injection will not work

The Issue #5 code fixes canonical/robots inside `server/_core/vite.ts`. That only works if the deployed platform actually runs this Node/Express server.

If Manus deploys a static SPA bundle, then this code will not execute. In that case there are only two acceptable routes:

### Option A — deploy the Express server

Use the server deployment mode where `server/_core/index.ts` and `server/_core/vite.ts` handle the SPA fallback. This is preferred for route-specific canonical and real 404 status.

### Option B — add edge/serverless routing

If static hosting is mandatory, add an edge/serverless function before the static fallback that:

- recognizes the public route whitelist,
- returns `404` for unknown routes,
- injects exactly one canonical and one robots tag into the returned HTML,
- strips the static homepage canonical from `index.html` first.

Do not claim the SEO fix is live until one of these paths is verified via `curl` against the preview URL.

## Required correction 3 — harden `lotterySettleHandler` before cron registration

The handler in Issue #5 is better than the previous version, but it is **not approved for heartbeat/cron registration yet**.

### Problems

1. The comment claims idempotency per `lotteryKey + drawDate + scheduleWindow`, but the code does not implement a settlement key or schedule window.
2. Concurrent cron retries can still process the same pending draw at the same time unless `buildLotterySettlement` fully protects every order and DB update. That is not proven in the artifact.
3. `summary.ok = true` is returned even when `summary.errors > 0` from fetch or settlement failures.
4. A draw can be marked `SETTLED` when every order was only skipped as `already settled`, without re-checking that the database has no remaining `PAID` orders.
5. The handler does not expose `drawsProcessed`, which makes monitoring harder.

### Concrete code corrections

#### 3.1 Do not return `ok: true` when errors occurred

Replace the final success return with:

```ts
summary.ok = summary.errors === 0;
return res.status(summary.errors === 0 ? 200 : 207).json({
  ...summary,
  drawsProcessed: pendingDraws.length,
});
```

If the scheduler treats any non-200 as failed and retries aggressively, keep HTTP `200` but set `ok: false` when errors exist:

```ts
summary.ok = summary.errors === 0;
return res.json({
  ...summary,
  drawsProcessed: pendingDraws.length,
});
```

The important part: `ok` must not be `true` when `errors > 0`.

#### 3.2 Re-check remaining PAID orders before marking a draw settled

Replace this logic:

```ts
if (
  paidOrders.length > 0 &&
  settledCount + skippedCount === paidOrders.length
) {
  await db
    .update(lotteryDraws)
    .set({ status: "SETTLED", updatedAt: new Date() })
    .where(eq(lotteryDraws.drawId, draw.drawId));
}
```

with a DB re-check after settlement attempts:

```ts
const remainingPaidOrders = await db
  .select({ orderId: lotteryOrders.orderId })
  .from(lotteryOrders)
  .where(
    and(
      eq(lotteryOrders.drawId, draw.drawId),
      eq(lotteryOrders.status, "PAID")
    )
  )
  .limit(1);

if (remainingPaidOrders.length === 0) {
  await db
    .update(lotteryDraws)
    .set({ status: "SETTLED", updatedAt: new Date() })
    .where(eq(lotteryDraws.drawId, draw.drawId));
}
```

Reason: the final source of truth should be the database state after settlement, not string-matched exception messages.

#### 3.3 Implement a real run idempotency key or remove the claim

If the project already has a generic lock/job table, use it. Otherwise add a minimal table/migration similar to this:

```ts
export const lotterySettlementRuns = pgTable("lottery_settlement_runs", {
  id: text("id").primaryKey(),
  runKey: text("run_key").notNull().unique(),
  lotteryKey: text("lottery_key").notNull(),
  drawId: text("draw_id").notNull(),
  scheduleWindow: text("schedule_window").notNull(),
  status: text("status").notNull().default("RUNNING"),
  startedAt: timestamp("started_at").notNull().defaultNow(),
  finishedAt: timestamp("finished_at"),
  summaryJson: jsonb("summary_json"),
});
```

At the start of processing each draw:

```ts
const scheduleWindow =
  req.header("x-ggc-schedule-window") ??
  String(req.query.window ?? req.body?.window ?? "manual");

const runKey = `${lotteryKey}:${draw.drawId}:${scheduleWindow}`;

try {
  await db.insert(lotterySettlementRuns).values({
    id: randomId(),
    runKey,
    lotteryKey,
    drawId: draw.drawId,
    scheduleWindow,
    status: "RUNNING",
  });
} catch (err) {
  summary.skipped++;
  summary.details.push({
    drawId: draw.drawId,
    lotteryKey,
    action: "DUPLICATE_RUN_SKIPPED",
    ordersSettled: 0,
    ordersSkipped: 0,
    error: `Duplicate settlement run: ${runKey}`,
  });
  continue;
}
```

At the end of that draw processing, update the run row to `DONE` or `ERROR` with summary JSON.

If adding a table is too much for this batch, then do not register crons and remove the idempotency claim from the report.

#### 3.4 Auth mode must be explicit before production cron

Current handler accepts either `x-ggc-scheduler-secret` or `sdk.authenticateRequest(req).isCron`.

Before registering external heartbeat schedules, document which mechanism will be used in production:

- If using secret header, set `LOTTERY_SETTLE_SECRET` and configure heartbeat to send `x-ggc-scheduler-secret`.
- If using Manus SDK cron auth, confirm that requests from outside Manus cannot satisfy `sdk.authenticateRequest`.

Do not leave this ambiguous in the deploy report.

## Required correction 4 — attach actual source diff again

Issue #5 includes a pasted handler and a commit in `manus-audit-inbox`, but Codex still does not have the actual GGC source repo. For the next verification, include one of these:

1. a GitHub PR/branch in the actual GGC source repo, or
2. a source ZIP / patch artifact from the GGC project, or
3. at minimum, full patches for:
   - `client/index.html`,
   - `server/_core/index.ts`,
   - `server/_core/vite.ts`,
   - `server/seoRoutes.ts`,
   - `server/lotterySettleHandler.ts`,
   - any schema/migration added for settlement run idempotency.

## Final acceptance criteria

Codex can accept the SEO portion only when live preview checks prove:

- no duplicate canonical on `/`, `/crash`, `/lottery/mega`,
- unknown route returns 404,
- unknown route is not indexable,
- `og:image`, `twitter:image`, JSON-LD remain present on `/`,
- `robots.txt` and `sitemap.xml` still work.

Codex can accept the settlement cron portion only when:

- real run idempotency exists or the cron is explicitly left disabled,
- `ok` reflects errors correctly,
- draw settlement status is based on post-settlement DB state,
- auth mechanism is explicit and verified,
- no cron/heartbeat schedules are registered before this passes.
