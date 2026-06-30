# Deploying dograh (AI Calling engine) in Dokploy

This fork is configured for **external** Postgres, Redis, and S3 (RustFS) — the
bundled `postgres`/`redis`/`minio`/`cloudflared` services were removed so you can
use Dokploy-managed databases (with backups) and your existing RustFS bucket.

Images are pinned to **`1.37.0`** (the `:latest` UI image is broken). Override
with `DOGRAH_TAG` if needed.

## 1. Prerequisites (in the Chati Dokploy project)

Create two **Dokploy database services** (so you get backup/restore):
- **`calling-postgres`** (Postgres) — note user/password
- **`calling-redis`** (Redis) — note password

Have ready: your **RustFS** bucket (`aicalling`) + access key/secret + endpoint
(`https://s3.chati.ai`).

## 2. Create the Compose app
- Dokploy → Chati project → **Create Service → Compose**
- Source: GitHub `devcrunchy/dograh`, branch `main`, compose file `docker-compose.yaml`

## 3. Environment variables (Dokploy env panel)

```
# Database & Redis (point at the Dokploy DB services — internal hostnames)
DATABASE_URL=postgresql+asyncpg://<pg-user>:<pg-pass>@calling-postgres:5432/dograh
REDIS_URL=redis://:<redis-pass>@calling-redis:6379

# Storage — RustFS (S3-compatible)
ENABLE_AWS_S3=true
AWS_ACCESS_KEY_ID=<rustfs access key>
AWS_SECRET_ACCESS_KEY=<rustfs secret>
S3_BUCKET=aicalling
S3_ENDPOINT_URL=https://s3.chati.ai
S3_REGION=us-east-1
S3_SIGNATURE_VERSION=s3v4
S3_ADDRESSING_STYLE=path

# Auth (REQUIRED — strong random)
OSS_JWT_SECRET=<long random string>

# WebRTC (needed for browser "web call"; set when you add coturn)
TURN_HOST=<server public IP/domain>
TURN_SECRET=<random>

ENABLE_TELEMETRY=false
```

Notes:
- `DATABASE_URL` MUST use `postgresql+asyncpg://` and a **dedicated `dograh`
  database** (create it on calling-postgres) — never the billing DB.
- Model keys (Gemini/Sarvam) and telephony keys (Plivo/Exotel) are added **in the
  dograh UI** after deploy, not here.

## 4. WebRTC / coturn (browser web calls)
By default only `api` + `ui` run. For browser "web call" you also need coturn —
deploy with the profile:
```
COMPOSE_PROFILES=local-turn
```
(That starts `coturn` + `dograh-init`; it does NOT start nginx — Dokploy/Traefik
handles TLS.) Open coturn ports: 3478, 5349, and UDP 49152–49200.

## 5. Domains (Dokploy → Domains)
- `ui` (port **3010**) → e.g. `calling.chati.ai`
- `api` (port **8000**) → e.g. `api-calling.chati.ai`

## 6. Deploy → then
1. Open the `ui` domain → **sign up** (first user = admin).
2. **Models** → add Gemini/Sarvam key → set a bot to Odia.
3. **Web Call** → test the Odia bot.
4. Later: **Telephony** → add Plivo/Exotel for real phone calls.

## Backups
Enable scheduled backups on **`calling-postgres`** in Dokploy (target your RustFS
`aicalling`/backups bucket). Postgres is the source of truth; Redis is just cache.
