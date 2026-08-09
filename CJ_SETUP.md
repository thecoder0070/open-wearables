# CJ OpenWearables Setup

This fork is the OpenWearables backend that feeds the CJ Fitness app.

**Live apps (two, cross-linked as tabs):**

- https://cj-fitness-2026.vercel.app — the **classic Health Tracker** (source
  `../health guide`, repo `thecoder0070/health-tracker`): real Strava/WHOOP/
  Garmin history, Runs/Heat/Trends/Wearables pages, serverless /api. Restored
  2026-07-27 via Vercel Instant Rollback to deployment `figcpxcmb` (d413ca1) —
  note: while the rollback state is active, new production deploys of this
  project do NOT auto-promote until the rollback is removed in the Vercel UI.
- https://cj-fitness-dashboard.vercel.app — the **OpenWearables dashboard**
  (Vercel project `cj-fitness-dashboard`; source `../cj-fitness-2026`, its own
  git repo): reads the `cj_*` Supabase tables seeded from the local
  OpenWearables backend. Its header tabs link to the classic app.

## Current state (2026-07-27)

- Backend runs **locally without Docker** (no Docker/node/brew on this Mac):
  Python 3.13 via `uv sync`, Postgres 16 from the `pgserver` pip package on
  `127.0.0.1:5433`. See `.localdev/README.md` for start/stop and all scripts.
- Admin: `calvin@cjfitness.app` / password in `backend/config/.env`
  (`.test` TLDs are rejected by backend email validation, so not `@local.test`).
- API key ("cj-fitness") and CJ user exist; ids in `.localdev/`.
- CJ user is seeded with the `active_athlete` preset (120 workouts, 30 sleeps,
  ~215k samples, daily scores), dated over the last 180 days.
- Supabase project "Personal-Health-Relationship-credit Card"
  (`nwnidtzcooerwdasgcpa`) holds the app's read model in `cj_*` tables
  (daily activity, sleep, workouts, health scores, daily series, sync meta) —
  RLS: public SELECT only. Sync via `.localdev/push_to_supabase.py`
  (temp anon write policies applied/dropped around each run).
- The same Supabase project also has older real Whoop/Garmin/Strava history
  (`whoop_*`, `garmin_*`, `health_*` tables) from previous direct integrations —
  untouched; a future OpenWearables ingest could replace those bespoke syncs.

## Local smoke test

Docker Desktop is installed (2026-07-27); the stack runs the documented way:

```bash
cd "/Users/calvinpatterson/Documents/Coding/open-wearables"
docker compose up -d
```

- API docs: http://localhost:8000/docs
- Developer portal: http://localhost:3000 (admin `calvin@cjfitness.app`)
- Celery worker/beat/Redis run, so backfills and scheduled syncs work.
- Data was migrated from the earlier no-Docker setup (same user id + API key);
  see `.localdev/README.md` for details and the legacy fallback.

## Garmin login via OpenWearables (built 2026-07-27; waiting on Garmin approval)

The OAuth plumbing is verified working end-to-end on our side (OAuth2 + PKCE
S256, state stored server-side; placeholder creds in `backend/config/.env`):

- Authorize: `GET /api/v1/oauth/garmin/authorize?user_id=<uuid>` → returns the
  `connect.garmin.com/oauth2Confirm` URL to send the user to.
- Callback: `/api/v1/oauth/garmin/callback` (validates state, exchanges code at
  `diauth.garmin.com`, stores the connection, fetches Garmin user id +
  permissions, and auto-dispatches a historical backfill via Celery —
  `HISTORICAL_SYNC_ON_CONNECT=true`).
- Live updates arrive by webhook: `POST /api/v1/garmin/webhooks/ping` and
  `POST /api/v1/garmin/webhooks/push`.
- Backfill monitoring: `/api/v1/providers/garmin/users/{user_id}/backfill/status`
  (+ cancel / retry).

### To activate (user steps)

1. Apply at developer.garmin.com → **Garmin Connect Developer Program**
   (request the Health API; approval typically takes days–weeks).
2. Once approved, create the app in Garmin's portal. Scopes/permissions are
   configured there, not in our env (`GARMIN_DEFAULT_SCOPE` stays empty).
3. Register these URLs in the Garmin portal (must be the PUBLIC backend URL —
   deploy the backend first, e.g. Oracle free VM or Railway):
   - OAuth redirect: `https://<backend>/api/v1/oauth/garmin/callback`
   - Ping notification URL: `https://<backend>/api/v1/garmin/webhooks/ping`
   - Push notification URL: `https://<backend>/api/v1/garmin/webhooks/push`
4. Replace `GARMIN_CLIENT_ID` / `GARMIN_CLIENT_SECRET` placeholders in
   `backend/config/.env` (values = consumer key/secret from the portal) and
   set `API_BASE_URL` to the public URL; restart the stack.
5. Connect from the portal (Users → CJ → connect Garmin) or the Health
   Tracker's Wearables page once its env vars point at the public backend.

## Production path (still to do)

Railway is the fastest hosted path for this repo. Deploy the Railway template or
this fork, then set:

- Backend `API_BASE_URL` to the public backend URL.
- Backend `FRONTEND_URL` and `CORS_ORIGINS` to the public frontend URL plus `https://cj-fitness-2026.vercel.app`.
- Frontend `VITE_API_URL` to the public backend URL.

Then fill provider OAuth credentials (Garmin/Whoop/Strava/Oura…) in the backend
env, connect providers, backfill history, and point the sync at the hosted
instance instead of localhost so real data replaces the demo seed.
