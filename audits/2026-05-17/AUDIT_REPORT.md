# [Audit] GGC Gaming — Lottery Follow-up + SEO + OG Image (2026-05-17)

## Summary

This batch covers 3 feature additions, 2 bug fixes, and comprehensive SEO improvements for the GGC Gaming platform at https://gg.world.

**Preview:** https://gg.world
**Domains:** gg.world, www.gg.world, ggcgaming-hhpbegje.manus.space

---

## Changes

### Features

1. **Lottery Settlement Heartbeat Worker** — `POST /api/scheduled/lotterySettle` auto-imports official Mega Millions & Powerball results, matches against player orders, calculates payouts, submits on-chain settlement. Precise schedule documented: +5min, +30min, +1h, +24h after each draw.

2. **Crash + Lottery Tabs in History Page** — `/history` now includes Crash bets (auto/manual cashout, multiplier, win/loss) and Lottery orders (game name, line count, USD prize, payout status) alongside all existing games.

3. **Crash Tile on Home Page** — Game grid now includes Crash card with Flame icon, "Up to 100x" payout, link to `/crash`.

### Bug Fixes

4. **Polish Text Removed** — Lottery card on home page had Polish text; changed to English.

5. **Official Lottery Names** — All labels changed from "US Mega Jackpot"/"US Power Jackpot" to "USA Mega Millions"/"USA Powerball". Special ball labels: "Gold ball" → "Mega Ball", "Power ball" → "Powerball".

### SEO

6. **Meta Tags** — Added meta description, OG tags (type, title, description, url, site_name, image), Twitter Card tags (summary_large_image + image), canonical URL, robots meta.

7. **JSON-LD Structured Data** — WebSite schema, Organization schema, ItemList with all 10 games.

8. **OG Image** — Generated branded 2560x1440 social preview (GGC coin, "Play. Earn. Own.", game icons).

9. **Sitemap** — `sitemap.xml` with 16 public routes. `robots.txt` updated with sitemap reference + `/admin369` disallow.

10. **Lottery Settlement Schedules** — Documented precise cron expressions for both DST windows (references/lottery-schedules.md). Registration pending deploy.

---

## Changed Files

| File | Type | Description |
|------|------|-------------|
| `client/index.html` | SEO | Meta description, OG/Twitter tags, canonical, robots, JSON-LD (3 schemas), og:image |
| `client/public/robots.txt` | SEO | Sitemap reference, /admin369 disallow |
| `client/public/sitemap.xml` | SEO | 16 public routes |
| `client/src/pages/History.tsx` | Feature | Crash + Lottery normalization, tabs, queries |
| `client/src/pages/Home.tsx` | Feature+Fix | Crash tile, English text fix, Lottery copy update |
| `client/src/pages/Lottery.tsx` | Fix | Official names, Mega Ball/Powerball labels |
| `client/src/pages/admin/AdminLottery.tsx` | Fix | Official names in pricing/claims/result import |
| `server/_core/index.ts` | Feature | Mounted lottery settlement handler |
| `server/lotterySettleHandler.ts` | Feature | Heartbeat handler for auto settlement |
| `server/routers.ts` | Feature | `lottery.orders.recent` public route |
| `shared/lottery.ts` | Fix | lotteryLabels → official names |
| `references/lottery-schedules.md` | Docs | Precise cron schedule documentation |

---

## Test Results

```
npx tsc --noEmit → 0 errors
npx vitest run → 8 test files, 43 tests passed, 0 failed
npx vite build → ✓ built in 26.11s
```

---

## SEO Checklist

| Item | Status | Details |
|------|--------|---------|
| `<title>` | ✅ | "GGC Gaming" |
| `<meta description>` | ✅ | "GGC Gaming — Provably fair on-chain games on BNB Chain..." |
| `<link rel="canonical">` | ✅ | https://gg.world |
| `hreflang` | N/A | Single-language (English) |
| `<html lang>` | ✅ | `lang="en"` |
| `<meta robots>` | ✅ | `index, follow` |
| `sitemap.xml` | ✅ | 16 URLs |
| `robots.txt` | ✅ | Allow all, disallow /admin369, sitemap ref |
| 404 handling | ✅ | NotFound component with catch-all route |
| 301 redirects | N/A | No redirect rules needed |
| OG tags | ✅ | type, title, description, url, site_name, image (2560x1440) |
| Twitter Card | ✅ | summary_large_image + title + description + image |
| JSON-LD | ✅ | WebSite + Organization + ItemList (10 games) |
| Visible content after hydration | ✅ | All pages render after React hydration |

---

## Contracts

| Contract | Address | Network |
|----------|---------|---------|
| LotteryVault | `0x0b9369c4a1c2e430e9731970EFe5dF3B9890efd9` | BSC Mainnet |

Configs: usMegaJackpot + usPowerJackpot enabled. Payout vault: 200,000 GGC.

---

## Known Issues / Open Items

1. Lottery heartbeat schedules documented but not yet registered (requires deploy first)
2. Chunk size warnings for wagmi/metamask-sdk bundles >500 kB (pre-existing)
3. No SSR/pre-rendering for SEO crawlability (SPA only)

---

## Codex Review Request

Please audit the following:
- [ ] SEO meta tags correctness and completeness
- [ ] JSON-LD schema validity
- [ ] OG image renders correctly on social platforms
- [ ] Lottery settlement handler logic (idempotency, error handling)
- [ ] History page normalization for Crash + Lottery data
- [ ] Official lottery naming consistency across all files
- [ ] Cron schedule accuracy for Mega Millions and Powerball draw times
- [ ] Security: no sensitive data exposed in client-side code
