# Render standby deployment

A second, independent copy of checkout-server on Render — usable if the VPS
goes down. Its database is the **Neon Postgres project that already holds a
synced copy of the VPS's MariaDB**, so the standby starts with real order
history, brands, and settings already in place — not an empty database.

## Architecture

| | VPS (primary) | Render (standby) |
|---|---|---|
| App | systemd, `checkout.service` | Render Web Service (`checkout-web`) |
| Background tasks | systemd, `checkout-worker.service` | Render Background Worker (`checkout-worker`) |
| Database | MariaDB, local to the VPS | **Neon Postgres** — the project the VPS syncs into |
| Redis | local to the VPS | Render Key Value (`checkout-redis`) |
| Data | real production orders | synced copy, via the existing VPS → Neon sync |

The VPS stays the system of record. Render connects to that same Neon
database with a normal connection string — that's why `render.yaml`
provisions no database of its own.

**Why the app runs on Postgres here:** the VPS runs MariaDB, Neon runs
Postgres — the app supports both via `DB_DIALECT` / `DB_URL` in `config.py`.
The VPS and staging keep running MySQL unchanged (zero `.env` changes
there); anything deployed against Neon just sets `DB_URL` and the right
driver is picked automatically.

## One-time setup

1. This repo is already on GitHub. Push the branch with these changes if
   you haven't already.
2. In the Render dashboard: **New +** → **Blueprint** → connect the repo
   (pick the right branch if prompted). Render reads `render.yaml` at the
   repo root and shows the services it's about to create (`checkout-redis`,
   `checkout-web`, `checkout-worker`). Click **Apply**.
3. Render prompts for every `sync: false` variable during this flow:

   | Variable | What to paste |
   |---|---|
   | `DB_URL` | Neon's **direct** connection string — see below. Same value on `checkout-web` and `checkout-worker`. |
   | `ADMIN_USERNAME` / `ADMIN_PASSWORD` | admin dashboard login for the standby |
   | `VIEWER_USERNAME` / `VIEWER_PASSWORD` | optional read-only login (leave blank to disable) |

### Which Neon connection string

Neon's dashboard offers two; use the **direct** (non-`-pooler`) one:

```
postgres://user:pass@ep-xxx-123456.us-east-2.aws.neon.tech/neondb?sslmode=require
```

- The pooled (`-pooler`) endpoint runs PgBouncer in *transaction* mode,
  and asyncpg's prepared statements don't survive that — you'd get
  intermittent "prepared statement does not exist" errors. The direct
  endpoint's connection limit is comfortably above this app's pool sizes
  (10 connections + overflow per service).
- Keep the `?sslmode=require` — `config.py` translates it to asyncpg's
  `ssl=` automatically. (Without that translation the engine crashes at
  startup: asyncpg's `connect()` has no `sslmode` parameter, and SQLAlchemy
  passes URL query params straight through as connect kwargs.)

## Required manual steps after the first deploy

1. **Update `BASE_URL`** on `checkout-web` to whatever `.onrender.com` URL
   Render assigned it (shown at the top of the service page), or a custom
   domain if you attach one.
2. **Add whichever payment-processor credentials you actually need** for
   standby use — Shopify, Stripe, Helcim, Shippo, BTCPay, WPay, pymtz,
   NowPayments, Resend (email), etc. Every one defaults to blank/disabled
   in `config.py`, exactly like an unconfigured `.env` on any other
   environment — the app runs fine with all of them off; each feature just
   stays disabled until you add real values. Copy whichever ones you need
   from the VPS's `.env` into the matching `checkout-web` environment
   variable.

## Verifying it worked

```
curl https://<your-service>.onrender.com/health
# {"status":"ok","environment":"production"}
```

Then log into `/peps-admin-2026/login` with the `ADMIN_USERNAME` /
`ADMIN_PASSWORD` you set. The dashboard should show your **real order
history** (from Neon) — an empty dashboard means `DB_URL` isn't wired up.
Note the very first request can take a few seconds: Neon's compute
auto-suspends after inactivity and needs a moment to wake (`pool_pre_ping`
in `database.py` already handles stale pooled connections after a wake-up).

## Failover: actually using this during an outage

1. Point DNS/Cloudflare at the Render URL (or share the URL directly with
   stores).
2. Consider **pausing the VPS → Neon sync** while serving from Neon, so a
   half-dead VPS can't overwrite newer Neon data with stale snapshots.
3. **Failing back:** orders taken during the outage exist in Neon only —
   the VPS's MariaDB never saw them. Before resuming normal VPS operation,
   move that window of orders Neon → MariaDB (manually or with a small
   script), or those orders will be missing from the VPS's view.

## Cost-saving option

The stack (2 Starter services + a Starter Key Value) costs roughly $20/mo
to keep running warm. If that's not worth it for a standby, you can
suspend `checkout-web`/`checkout-worker` from the Render dashboard when not
in use — but a suspended stack needs manual resuming before it can take
traffic, which slows down a real failover.

## Redeploying after code changes

Render auto-deploys `checkout-web` and `checkout-worker` on every push to
the connected GitHub branch by default. `checkout-redis` never needs
redeploying for app code changes.

## Schema notes

Table creation happens automatically on `checkout-web` startup
(`Base.metadata.create_all` in `main.py`, dialect-agnostic), so any table
missing from the synced Neon copy gets created on first boot. The old
incremental scripts under `migrations/` use MySQL-specific raw SQL and are
**not** run against Neon — the VPS remains where schema migrations are
applied, and the sync carries them over.
