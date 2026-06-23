# RogerAI broker - DigitalOcean deploy

The broker is the only public component. Deploy it cheap on DO App Platform, point
`broker.rogerai.fyi` at it, and reuse your existing managed Postgres. Clients (`rogerai`)
need nothing public - they dial out.

## What you need
- The `rogerai.fyi` domain (bought ✓) + Cloudflare DNS (you manage).
- An existing DO managed **Postgres** (we just add `rogerai_`-prefixed tables - no new DB needed).
- DO App Platform **Basic** plan (cheapest, ~$5/mo) - same tier as your other backends.

## DO project
All RogerAI resources live under the **"RogerAI" DO project** (created in the console). Assign
the App + the managed Postgres to it: `doctl projects list` for the id, then
`doctl projects resources assign <project-id> --resource=do:app:<app-id> --resource=do:dbaas:<db-id>`
(or move them in the console). Keeps billing/resources grouped.

## Steps
1. **Push the repo** (GitHub). App Platform builds from the `Dockerfile` (or set it as the
   build target). The broker binary is tiny + static.
2. **Create the App** (DO console or `doctl apps create --spec .do/app.yaml`):
   - Source: this repo, Dockerfile build.
   - HTTP port: App Platform sets `$PORT`; the broker binds `0.0.0.0:$PORT` automatically.
   - Env vars:
     - `DATABASE_URL = <your DO managed-Postgres connection string>?sslmode=require`
     - (optional) `--fee` / `--seed-credits` via the run command if you want non-defaults.
   - Instance: Basic XS is plenty to start (the broker is I/O-bound; inference runs on providers).
3. **Domain**: in DO add the domain `broker.rogerai.fyi`; in **Cloudflare DNS** add a CNAME
   `broker` → the App Platform hostname (proxied is fine; it's plain HTTPS). DO provisions TLS.
4. **Verify**: `curl https://broker.rogerai.fyi/health` → `ok`. The Postgres tables
   (`rogerai_wallet`, `rogerai_earnings`, `rogerai_receipts`) auto-create on first boot.
5. **Point the CLI at it**: `rogerai config set broker https://broker.rogerai.fyi`. Then
   `rogerai share` (provider) and `rogerai search/use` (consumer) work over the public broker.

## Database provisioning (least-privilege - one-time, as admin)
The app's DB user gets ONLY its own schema; no database-level CREATE, no `public`. Run once
with the admin URL (`DATABASE_ADMIN_URL`):
```sql
CREATE SCHEMA IF NOT EXISTS rogerai AUTHORIZATION rogerai_user;  -- user owns its schema
GRANT CONNECT ON DATABASE rogerai_db TO rogerai_user;
GRANT USAGE, CREATE ON SCHEMA rogerai TO rogerai_user;
REVOKE CREATE ON SCHEMA public FROM rogerai_user;               -- can't litter public
ALTER ROLE rogerai_user SET search_path = rogerai;
```
The broker (as `rogerai_user`) then auto-creates `rogerai.wallet/earnings/receipts` on boot.
Verified against the live DO cluster. (This box must be in the DB's DO "trusted sources" to run
admin SQL from here; the deployed App reaches it over DO's private network.)

## Reuse the existing Postgres (no new spend)
The schema is namespaced (`rogerai_*`), so it coexists in any existing database. If you'd rather
isolate it, create a `rogerai` schema or a separate DB user - only `DATABASE_URL` changes.

## Staying modular (swap the DB anytime)
Persistence lives behind `internal/store.Store` (methods: BalanceOf, Settle, EarningsOf,
AddCredits, Close). Today: `Mem` (no DATABASE_URL) and `Postgres` (pgx). To move to another
managed Postgres → just change `DATABASE_URL`. To move to a different engine (e.g. SQLite for a
self-hoster, or a managed MySQL) → add one file implementing `Store` and select it in the
broker's `main`. No other code touches the DB. This keeps brokers cheap to run and easy to fork
(see BROKER-SPEC.md - anyone can host one).

## Scaling later
- Multiple broker instances behind App Platform: move the per-node tunnel/job state to Redis
  (or sticky routing) so any instance can reach any node. P2 work; one instance is fine to start.
- Egress: all inference traffic transits the broker - watch bandwidth; upgrade the instance or
  add the P2 direct-path upgrade when volume grows.
