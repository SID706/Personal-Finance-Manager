# Sandbox / Base44 notes

Single-file Flask app (`app.py`) with server-rendered Jinja templates in
`templates/`, assets in `static/`, and MongoDB. There is no test suite and no
linter config in this repo.

## How it runs in the sandbox

- `docker-compose.base44.yml` is the sandbox runbook. `web` runs the **Flask dev
  server with the reloader** from the bind-mounted source on container port
  5000, published as host port 3000 (the preview). `mongo` is a plain `mongo:6.0`
  service with a persistent volume.
- Python dependencies are installed at container start from `requirements.txt`
  (wheels cached in the `pip_cache` volume), so a requirements change only needs
  `docker compose -f docker-compose.base44.yml restart web` — no image rebuild.
- `create_indexes()` runs at **import time** and drops/recreates indexes on every
  start and every reload. Harmless for dev data, but never rely on custom
  indexes surviving a restart.
- `.env.base44-defaults` holds non-production placeholders (local `MONGO_URI`,
  `ADMIN_PASSWORD=admin-dev-password`, `ADMIN_SETUP_CODE=dev-setup-code`). It is
  listed **first** in `env_file:`; `/run/base44/app.env` (dashboard Secrets) is
  listed last so a real value always wins. Keep new secrets out of Compose
  `environment:` for the same reason.
- Do **not** set `FLASK_ENV=production` here: `app.py` then sets
  `SESSION_COOKIE_SECURE=True`, which breaks login over the preview origin.
- No sandbox-only code branches are needed — the app needed no
  `BASE44_PREVIEW_MODE` overrides.

## Non-obvious app facts

- `templates/` and `static/` had been dropped from the repository (they only
  survived inside `Finance-Manager-main.zip`, removed in commit `aa67292`).
  They were restored from that archive at commit `cbc1e10`; keep them in sync
  with `app.py` route names when editing.
- Email is optional: `RESEND_API_KEY` unset simply disables sending; the
  register/login flows do not depend on it.
- `/initialize-admin` only works while no admin exists; it needs
  `ADMIN_SETUP_CODE`. Admin-only actions (clear logs, toggle admin) additionally
  verify `ADMIN_PASSWORD`.
- Rate limiting uses in-memory storage (`memory://`), so repeated scripted
  hits on `/register` (10/hour) or the global 200/day limit reset on a `web`
  restart.

## Verifying it works

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/            # 200 landing page
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000/dashboard   # 302 -> /login when signed out
docker compose -f docker-compose.base44.yml ps                            # both services healthy
```

For an end-to-end check, register at `/register` (email + 8-char password),
which redirects to `/dashboard`; add a transaction at `/transactions/add` and
confirm it appears on `/dashboard` and `/transactions` and in
`/api/reports/summary`.
