# Setup

Tracks Phase 0 from `plan.md`. Items marked **[owner action]** require the
repo owner directly — an assistant cannot open financial accounts, accept
broker terms, or install GUI software on your behalf.

## Repo / environment
- [x] Git repo initialized, `plan.md`, `agents.md`, `.env.example` committed
- [x] `requirements.txt` added (ib_insync, pandas, backtrader, vectorbt, python-telegram-bot, python-dotenv)
- [ ] Python 3.12 venv — this dev machine currently has Python 3.14.6 and no
      3.12 available (no pyenv installed). Install Python 3.12 (e.g. via
      `pyenv install 3.12` or python.org) before creating the venv:
      ```
      python3.12 -m venv .venv
      source .venv/bin/activate
      pip install -r requirements.txt
      ```

## IBKR account **[owner action]**
- [ ] Open or confirm an IBKR Canada account (TFSA or non-registered)
- [ ] Enable API access: Account Settings → Settings → API → Enable ActiveX and Socket Clients
- [ ] Set up an IBKR **paper-trading account** (provided natively by IBKR alongside your live account)

## IB Gateway **[owner action]**
- [ ] Download IB Gateway (headless-capable, lighter than full TWS): https://www.interactivebrokers.com/en/trading/ibgateway-stable.php
- [ ] Install on this dev machine first and confirm paper-trading login works
      before deploying to any VM (Phase 5)
- [ ] Note the paper-trading socket port (default `4002`) and set it in `.env`
      (see `.env.example`)

## Once the above is done
Copy `.env.example` to `.env` and fill in `IBKR_ACCOUNT_ID` and any Telegram
credentials. `.env` is gitignored and must never be committed.
