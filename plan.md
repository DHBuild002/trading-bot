# Autonomous Market Bot — Plan for Claude Code

**Broker:** Interactive Brokers Canada (stocks, ETFs, commodities via futures)
**Budget:** ~$500/month trading capital, near-zero infra cost
**Owner:** DB002

**Why IBKR over other Canadian brokers:** RBC GoSmart and Questrade have no public trading API — both are manual/app-only or, at best, offer read-only AI access with a mandatory approval gate before any order fires. That rules out true autonomy. IBKR is the only Canada-accessible broker with a full programmatic API (via `ib_insync`/`ibapi`) covering stocks, ETFs, options, futures, and commodities, plus TFSA/RRSP support so registered-account tax treatment is preserved.

---

## PART 1 — Investor Pitch (audience: Claude as VC)

### The opportunity
A self-hosted, low-cost bot that trades crypto on Coinbase using rule-based signals, with a disciplined risk layer and full backtest/paper-trade validation before any capital goes live. This is pitched honestly: the edge comes from *process discipline* (fees, sizing, drawdown control), not from a secret predictive model. Any framing that promises consistent alpha should be treated as a red flag — I'm avoiding that here.

### Recommended tool stack

| Layer | Tool | Why |
|---|---|---|
| Language | Python 3.12 | Best library support for trading/data |
| Broker connectivity | **ib_insync** (wraps IBKR's native API) + **IB Gateway** (headless, lighter than full TWS) | ib_insync is the standard, well-documented Python interface to IBKR; IB Gateway runs without a GUI, suited to an always-on VM |
| Bot framework | Custom Python service built on ib_insync (no equivalent of Freqtrade exists for IBKR) | IBKR's asset breadth (stocks/ETFs/futures) doesn't map to a crypto-bot framework; a slim custom event loop around ib_insync is the standard approach here |
| Backtesting | **backtrader** or **vectorbt**, fed by IBKR historical data or a free source (Yahoo Finance/Stooq) | Both handle equities/futures well; vectorbt for faster vectorized research, backtrader if you want built-in broker-simulation realism (commissions, slippage) |
| Data storage | SQLite | Free, zero-admin, sufficient at this scale |
| Hosting | **Oracle Cloud Always Free** ARM VM (4 OCPU/24GB, genuinely free forever) | Beats Azure free tier, which is free for 12 months only, then billed. Azure remains useful for your learning goals but isn't the cost-optimal host here. IB Gateway runs headless on this VM via Xvfb + a keep-alive/re-login script (IBKR sessions require periodic re-authentication — build this in early) |
| Alerting | Telegram bot (free) | Custom integration via python-telegram-bot; trade notifications + remote status/kill-switch commands |
| CI / health checks | GitHub Actions (free minutes) | Periodic uptime/heartbeat checks, no extra hosting cost |
| Version control | GitHub | Already your workflow |

**Estimated infra cost: $0–10/month.** IBKR charges no platform fee for the API itself; commissions are low (IBKR Pro: typically fractions of a cent per share, low flat futures fees). The one caveat: real-time Level 1 market data for Canadian/US equities may require a small monthly exchange data subscription (often a few dollars, sometimes waived at low activity) — confirm current IBKR data-fee terms before going live, since these change periodically.

**Market hours matter now.** Unlike crypto's 24/7 market, equities/commodities trade on exchange hours (with pre/post-market extensions on some). The bot's scheduling logic needs to account for this — no signals fire, and no orders route, outside market hours.

### On "accuracy" — the honest version
Single-indicator or simple technical-signal strategies on liquid equities typically land around **50–55% directional accuracy** — close to a coin flip, and if anything equities markets are more efficient and more heavily arbitraged by institutions than crypto, so the bar for a genuine retail edge is higher, not lower. Any small edge is easily erased by fees, slippage, and bid-ask spread if the strategy overtrades. This isn't a flaw specific to this build; it's the baseline reality of retail algo trading, and I'm not going to sell you a number that implies otherwise.

What *does* move the needle:
- **Risk management** (fixed fractional position sizing, hard stop-losses, max daily loss limits) matters more than prediction accuracy.
- **Backtest-to-live degradation is real** — Sharpe ratios and win rates commonly fall by 30–50%+ out of sample. The plan below forces a mandatory paper-trading gate before any real capital is committed, specifically to surface this before it costs money.
- Success should be measured by **risk-adjusted return (Sharpe/Sortino) and max drawdown**, not raw % gain claims.

### Growth expectations
I'm deliberately not giving you a monthly return projection — any specific number here would be a guess dressed up as a forecast, and that's the opposite of what a good pitch should do. A more honest framing:

- **Phase 1 goal:** the bot should reliably beat "do nothing" after fees, in backtest and paper trading, before touching real money.
- **Phase 2 goal:** live capital starts small (e.g., $50–100 of the $500 monthly allocation) with the rest held back until the strategy proves itself over a live sample of at least 4–8 weeks.
- Treat any return above capital preservation + fee coverage in year one as a good outcome. This is a research and infrastructure project first, a return-generating system second.

### Risks (stated plainly)
- Market risk: equities and commodities can still produce large drawdowns, particularly if the strategy touches leveraged instruments like futures.
- Overfitting: strategies tuned on historical data often fail live — mitigated by the paper-trading gate.
- Operational risk: IBKR sessions require periodic re-authentication and the Gateway process can disconnect — a bad disconnect during an open position is a real failure mode, mitigated by kill-switches, max-loss circuit breakers, and a "flatten all positions on connectivity loss" safeguard (built into the task list below).
- Leverage risk: futures (used for commodities exposure) are margined instruments — losses can exceed the capital allocated to a trade if position sizing isn't strictly enforced.
- This is not financial advice, and I'm not a financial advisor — the plan below is an engineering and risk-control framework, not a return guarantee.

### The ask
Approve the build below: a paper-trading-first, fee-aware, risk-gated bot on free/near-free infrastructure, with a hard rule that no strategy trades live capital until it has passed backtest + a live paper-trading window.

---

## PART 2 — Actionable Task List (for Claude Code)

### Phase 0 — Environment & repo setup
- [ ] Init Git repo, `plan.md`, `agents.md`, `.env.example` (no secrets committed)
- [ ] Set up Python 3.12 venv, `requirements.txt` (ib_insync, pandas, backtrader or vectorbt, python-telegram-bot)
- [ ] Open/confirm IBKR Canada account (TFSA or non-registered, per your goals) and enable API access in Account Settings
- [ ] Install IB Gateway (headless) on dev machine first; confirm paper-trading login works before touching the VM
- [ ] Set up an **IBKR paper-trading account** (IBKR provides this natively — use it as the Phase 4 gate instead of a custom dry-run mode)

### Phase 1 — Data pipeline
- [ ] Pull historical OHLCV via ib_insync's `reqHistoricalData`, or a free source (Stooq/Yahoo Finance) for faster iteration during research
- [ ] Store data locally (SQLite/Parquet); write a refresh script for daily updates, respecting exchange calendars (no data pulls expected on weekends/holidays)
- [ ] Validate data integrity (gaps, timezone consistency, corporate actions like splits/dividends if trading individual stocks)

### Phase 2 — Backtesting framework
- [ ] Implement 2–3 baseline strategies (e.g., EMA cross, RSI mean-reversion, Bollinger breakout) on a small liquid universe (start with a handful of large-cap TSX/S&P names, or a commodities ETF/futures proxy) as a comparison floor
- [ ] Run backtests over multiple market regimes (bull, bear, chop — at minimum 2021–2024 range)
- [ ] Record metrics: win rate, Sharpe, max drawdown, fee-and-spread-adjusted return, trade frequency
- [ ] Reject any strategy that doesn't beat buy-and-hold on a risk-adjusted basis after costs

### Phase 3 — Risk management layer (build before going further)
- [ ] Fixed-fractional position sizing (e.g., risk ≤1–2% of allocated capital per trade)
- [ ] Hard stop-loss per trade + max daily loss circuit breaker
- [ ] Max concurrent open positions cap
- [ ] If using futures for commodities exposure: explicit margin/leverage caps, separate and stricter from equity position sizing
- [ ] Kill-switch: auto-halt trading on API errors, abnormal volatility, or drawdown threshold breach
- [ ] "Flatten on disconnect" safeguard: if IB Gateway loses connection while positions are open, trigger an alert and (per your risk tolerance) either hold or auto-flatten

### Phase 4 — Paper trading (mandatory gate)
- [ ] Deploy best-performing strategy against your **IBKR paper-trading account** (native IBKR feature — no custom dry-run mode needed)
- [ ] Run for minimum 4–8 weeks, across normal market hours, before any live capital
- [ ] Compare live paper results against backtest expectations; document degradation
- [ ] Set explicit go/no-go criteria (e.g., Sharpe > X, max drawdown < Y%) before Phase 6

### Phase 5 — Deployment infrastructure
- [ ] Provision Oracle Cloud Always Free ARM VM
- [ ] Install IB Gateway headless (Xvfb + a supervised process) on the VM; build a re-login/keep-alive script (IBKR sessions expire and need periodic re-authentication — this is the main new operational wrinkle vs. crypto)
- [ ] Deploy the custom bot service via Docker Compose; configure to survive reboots (systemd/Docker restart policy)
- [ ] Set up Telegram bot for trade notifications and remote start/stop/status commands
- [ ] Configure GitHub Actions heartbeat check (pings bot's status endpoint every N hours, alerts on failure) — include a Gateway-connectivity check specifically

### Phase 6 — Controlled live trading
- [ ] Allocate a small initial slice of the $500/month (e.g., $50–100) to live trading
- [ ] Keep paper trading running in parallel as a control group for comparison
- [ ] Weekly review of live vs. paper vs. backtest performance
- [ ] Scale allocation only after sustained (8–12 week) live performance meeting go/no-go criteria

### Phase 7 — Monitoring & iteration
- [ ] Build a simple dashboard (Freqtrade UI or lightweight custom view) for P&L, drawdown, open positions
- [ ] Monthly strategy review; log all parameter changes with rationale (avoid silent overfitting via constant tweaking)
- [ ] Document every strategy change and its backtest justification in `agents.md`

---

## Notes for Claude Code
- Default to IBKR **paper-trading account** unless explicitly told to enable live trading.
- Never commit API keys or IBKR credentials; use `.env` + `.gitignore`.
- Prioritize the risk-management layer (Phase 3) before any strategy optimization — it matters more than signal accuracy.
- Build the IB Gateway re-authentication/keep-alive logic early and test it under simulated disconnects — this is the most IBKR-specific operational risk and the most likely source of a silent failure if skipped.
