# Lottery Settlement Schedules

## Draw Times (UTC)

| Lottery | Days | Draw Time (ET) | Draw Time (UTC) |
|---------|------|---------------|-----------------|
| USA Mega Millions | Tue, Fri | 23:00 ET | 03:00 UTC (Wed, Sat) |
| USA Powerball | Mon, Wed, Sat | 22:59 ET | 02:59 UTC (Tue, Thu, Sun) |

Note: During US Daylight Saving Time (Mar-Nov), ET = UTC-4. During Standard Time (Nov-Mar), ET = UTC-5.
The cron times below use UTC-4 (summer) offsets. Adjust in November and March.

## Settlement Check Windows

For each draw, check results at:
- **+5 min** after draw time (first attempt, results often available immediately)
- **+30 min** after draw time (fallback if first attempt had no result)
- **+1 hour** after draw time (second fallback)
- **+24 hours** after draw time (final safety net)

## Cron Expressions (6-field, UTC)

### Mega Millions (draws Wed 03:00 UTC, Sat 03:00 UTC)

| Check | Cron Expression | Description |
|-------|----------------|-------------|
| +5min | `0 5 3 * * 3,6` | Wed+Sat at 03:05 UTC |
| +30min | `0 30 3 * * 3,6` | Wed+Sat at 03:30 UTC |
| +1h | `0 0 4 * * 3,6` | Wed+Sat at 04:00 UTC |
| +24h | `0 0 3 * * 4,0` | Thu+Sun at 03:00 UTC |

### Powerball (draws Tue 02:59 UTC, Thu 02:59 UTC, Sun 02:59 UTC)

| Check | Cron Expression | Description |
|-------|----------------|-------------|
| +5min | `0 4 3 * * 2,4,0` | Tue+Thu+Sun at 03:04 UTC |
| +30min | `0 29 3 * * 2,4,0` | Tue+Thu+Sun at 03:29 UTC |
| +1h | `0 59 3 * * 2,4,0` | Tue+Thu+Sun at 03:59 UTC |
| +24h | `0 59 2 * * 3,5,1` | Wed+Fri+Mon at 02:59 UTC |

### Monthly Schedule Verification

| Check | Cron Expression | Description |
|-------|----------------|-------------|
| Monthly | `0 0 12 1 * *` | 1st of each month at 12:00 UTC |

## Registration Commands (run after deploy)

```bash
# Mega Millions +5min
manus-heartbeat create \
  --name lottery-mega-5min \
  --cron "0 5 3 * * 3,6" \
  --path /api/scheduled/lotterySettle \
  --description "Mega Millions settlement check +5min after draw"

# Mega Millions +30min
manus-heartbeat create \
  --name lottery-mega-30min \
  --cron "0 30 3 * * 3,6" \
  --path /api/scheduled/lotterySettle \
  --description "Mega Millions settlement check +30min after draw"

# Mega Millions +1h
manus-heartbeat create \
  --name lottery-mega-1h \
  --cron "0 0 4 * * 3,6" \
  --path /api/scheduled/lotterySettle \
  --description "Mega Millions settlement check +1h after draw"

# Mega Millions +24h
manus-heartbeat create \
  --name lottery-mega-24h \
  --cron "0 0 3 * * 4,0" \
  --path /api/scheduled/lotterySettle \
  --description "Mega Millions settlement check +24h after draw"

# Powerball +5min
manus-heartbeat create \
  --name lottery-power-5min \
  --cron "0 4 3 * * 2,4,0" \
  --path /api/scheduled/lotterySettle \
  --description "Powerball settlement check +5min after draw"

# Powerball +30min
manus-heartbeat create \
  --name lottery-power-30min \
  --cron "0 29 3 * * 2,4,0" \
  --path /api/scheduled/lotterySettle \
  --description "Powerball settlement check +30min after draw"

# Powerball +1h
manus-heartbeat create \
  --name lottery-power-1h \
  --cron "0 59 3 * * 2,4,0" \
  --path /api/scheduled/lotterySettle \
  --description "Powerball settlement check +1h after draw"

# Powerball +24h
manus-heartbeat create \
  --name lottery-power-24h \
  --cron "0 59 2 * * 3,5,1" \
  --path /api/scheduled/lotterySettle \
  --description "Powerball settlement check +24h after draw"

# Monthly schedule verification (AGENT cron - checks official sites for schedule changes)
# This one uses AGENT cron because it needs to browse official lottery websites
```

## DST Adjustment Notes

In November (when US clocks fall back to EST = UTC-5):
- Mega Millions draws at 04:00 UTC (not 03:00)
- Powerball draws at 03:59 UTC (not 02:59)

In March (when US clocks spring forward to EDT = UTC-4):
- Mega Millions draws at 03:00 UTC
- Powerball draws at 02:59 UTC

**Action required**: Update cron expressions twice per year at DST transitions, or add a second set of crons covering both windows and let the handler's idempotency handle the no-op calls.

## Alternative: Dual-window approach (no DST maintenance)

Register crons for BOTH possible draw times. The handler is idempotent — if no draws are pending, it returns immediately with `drawsProcessed: 0`. This avoids manual DST updates.

```bash
# Mega: cover both 03:00 UTC (summer) and 04:00 UTC (winter)
# +5min checks at both 03:05 and 04:05 on Wed+Sat
manus-heartbeat create --name lottery-mega-5min-summer --cron "0 5 3 * * 3,6" --path /api/scheduled/lotterySettle --description "Mega +5min (summer)"
manus-heartbeat create --name lottery-mega-5min-winter --cron "0 5 4 * * 3,6" --path /api/scheduled/lotterySettle --description "Mega +5min (winter)"
# ... repeat for +30min, +1h, +24h
```
