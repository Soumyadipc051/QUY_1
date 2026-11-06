# MX Options Engine: multi-broker NIFTY options automation (Django + Celery)

Users sign up, connect **their own** broker (Angel One SmartAPI, Zerodha Kite Connect, ICICI Direct Breeze), create a run
(iron fly / short straddle), and a Celery worker executes it with per-user risk limits. Paper mode by default.

```
Browser (Tailwind + HTMX)  ->  Caddy (HTTPS)  ->  Django/gunicorn  ->  Postgres
                                                   Celery worker  ->  broker adapter (brokers/*.py) -> broker API
                                                   Celery beat    ->  dispatch() every minute in market hours
```

## Run locally
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env     # fill DJANGO_SECRET_KEY and FERNET_KEY, set DJANGO_DEBUG=1, remove DATABASE_URL/REDIS_URL lines
export $(grep -v '^#' .env | xargs)
python manage.py makemigrations core && python manage.py migrate
python manage.py createsuperuser
python manage.py runserver
# second terminals (need Redis): celery -A config worker -l info   |   celery -A config beat -l info
```

## Deploy on a public domain (any VPS, e.g. 2 vCPU / 4 GB)
1. Point your domain's A record at the server; open ports 80/443.
2. `git clone <your repo> && cd mxplatform && cp .env.example .env` and set every value (strong secrets, `DOMAIN`, `ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS`).
3. `docker compose run --rm web python manage.py makemigrations core` (commit the generated migration), then `docker compose up -d --build`.
4. `docker compose exec web python manage.py createsuperuser`.
Caddy obtains and renews the HTTPS certificate automatically.

## Push to GitHub
```bash
git init && git add . && git commit -m "MX Options Engine"
git branch -M main && git remote add origin https://github.com/<you>/mxplatform.git && git push -u origin main
```
`.env` is git-ignored. **Never commit keys.** CI (`.github/workflows/ci.yml`) runs checks and tests on every push.

## Broker notes (verify each against the vendor docs: I could not test against live APIs)
| Broker | Auth | Notes |
|---|---|---|
| Angel One | api key + client code + MPIN + TOTP secret | Fully automatic daily login. Storing a TOTP secret is high-trust: tell users plainly. |
| Zerodha Kite | api key + secret | Daily login from dashboard. Set your Kite app redirect URL to `https://yourdomain/kite/callback/`. Kite Connect is a paid API per app. |
| ICICI Breeze | api key + secret | Daily session token pasted by the user. No VIX feed (ATM IV used) and no 3m/15m bars (5m/30m used). Response field names must be verified. |
| Kotak | not implemented | The NEFT/RTGS link is a payments API. Trading needs Kotak Neo: implement `brokers/kotak.py`. |

## Going live: read this
- `ALLOW_LIVE_TRADING=0` by default. Paper-trade for weeks first; check fills, margin and expiry-day behaviour.
- **Regulation (India):** running automated order placement for the public, holding users' credentials, or charging for signals can fall under SEBI rules (retail algo framework, broker-approved/registered algos with strategy IDs, RA/IA registration). Get legal advice before opening this to strangers or charging money. Most brokers' API terms also restrict one API app serving many clients.
- Security checklist: HTTPS only, strong `FERNET_KEY` backup (losing it makes stored credentials unreadable), DB backups, login rate limiting (add `django-axes`), 2FA for admins, minimal logging of broker responses, a privacy policy and risk disclaimer.

## Scope of this version
Ported: neutral regime gate (ADX15, PCR, VIX), iron fly + short straddle ranking, SL/target/15:10 square-off, flatten, per-user isolation.
Not yet ported from the single-user script: bull put / bear call spreads, Supertrend exits, overnight NRML fly, short-leg roll, websocket feed, daily-loss limit across strategies.
