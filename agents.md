# Notes for Claude Code

Working log for autonomous/assisted work on this repo. See `plan.md` for the
full project plan and phase task list.

## Operating rules
- Default to the IBKR **paper-trading account** unless explicitly told to enable live trading.
- Never commit API keys, IBKR credentials, or `.env` — use `.env.example` as the template.
- Prioritize the risk-management layer (Phase 3) before any strategy optimization.
- Build and test IB Gateway re-authentication/keep-alive logic under simulated disconnects before relying on it.

## Strategy change log
Document every strategy change here with its backtest justification, to avoid
silent overfitting via constant tweaking. Format:

```
### YYYY-MM-DD — <change summary>
- What changed:
- Why (backtest/paper-trading evidence):
- Result:
```

(No strategy changes yet — Phase 2 not started.)

## Phase progress
Tracked via GitHub Issues (one per phase, labeled `plan`). See repo Issues tab
for current status of each phase.
