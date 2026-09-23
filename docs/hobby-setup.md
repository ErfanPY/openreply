# Hobby deployment: OpenReply + Zernio (@yourgrandfatherishere)

Personal deployment overlay for [ErfanPY/openreply](https://github.com/ErfanPY/openreply). Forked from [diwenne/openreply](https://github.com/diwenne/openreply).

**GitHub org note:** `hobby-projects` org creation was not available via CLI on this account. Repo lives at **https://github.com/ErfanPY/openreply** until an org is created manually at [github.com/organizations/plan](https://github.com/organizations/plan).

## Target Instagram account

| Field | Value |
| --- | --- |
| Handle | `@yourgrandfatherishere` |
| Zernio account id | `6ab389f38d284ffb213303af` |
| Provider | Zernio (no Meta app review) |

**Security:** If a Zernio API key was ever pasted in chat or committed, rotate it in the [Zernio dashboard](https://zernio.com/dashboard) before connecting. Store the new key only in OpenReply **Settings** (encrypted in Postgres) — never in git or `.env`.

## Local prerequisites (Windows)

| Requirement | Purpose |
| --- | --- |
| Node.js 20+ | OpenReply (Next.js) |
| Docker Desktop | PostgreSQL 16 + Redis 7 (`docker compose up -d`) |
| ngrok or Cloudflare Tunnel | Public HTTPS for Zernio webhooks + magic-link OAuth |
| [Resend](https://resend.com) | Magic-link login (`RESEND_API_KEY`, verified `EMAIL_FROM`) |
| Zernio with **Inbox access** | Unrestricted read/write API key |

Working directory: `d:\dev\hobby\openreply`

## 1. Infrastructure

**Preferred:** Docker Desktop + `docker compose up -d` (Postgres 16 + Redis 7).

**Fallback (no Docker):** On this machine Docker was not installed. Redis and PostgreSQL were installed via Scoop instead:

```powershell
scoop install redis postgresql
pg_ctl -D "$env:USERPROFILE\scoop\persist\postgresql\data" -l "$env:USERPROFILE\scoop\persist\postgresql\logfile" start
Start-Process redis-server
psql -U postgres -c "CREATE DATABASE openreply;"
```

Use `DATABASE_URL=postgresql://postgres@localhost:5432/openreply` (Scoop Postgres uses trust auth, no password).

```powershell
cd d:\dev\hobby\openreply
npm install
copy .env.example .env
```

Generate secrets (PowerShell):

```powershell
# NEXTAUTH_SECRET / CRON_SECRET (run twice)
[Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }))

# ENCRYPTION_KEY (64 hex chars)
-join ((1..32 | ForEach-Object { '{0:x2}' -f (Get-Random -Maximum 256) }))
```

Set in `.env`:

| Variable | Local value |
| --- | --- |
| `DATABASE_URL` | `postgresql://postgres@localhost:5432/openreply` (Scoop) or `postgresql://postgres:postgres@localhost:5432/openreply` (Docker) |
| `REDIS_URL` | `redis://localhost:6379` |
| `NEXTAUTH_SECRET` | generated |
| `CRON_SECRET` | generated |
| `ENCRYPTION_KEY` | generated (same on web + worker) |
| `RESEND_API_KEY` | from Resend dashboard |
| `EMAIL_FROM` | verified sender, e.g. `OpenReply <login@yourdomain.com>` |
| `ALLOWED_EMAILS` | your email only (comma-separated) |
| `NEXTAUTH_URL` | ngrok HTTPS URL (set after tunnel starts) |

Do **not** set Meta vars for Zernio-only setup.

```powershell
npm run db:generate
npm run db:migrate
```

## 2. Resend (magic-link login) — user action required

Set in `.env` (never commit real values):

| Variable | Source |
| --- | --- |
| `RESEND_API_KEY` | [Resend dashboard](https://resend.com/api-keys) |
| `EMAIL_FROM` | Verified domain sender, e.g. `OpenReply <login@yourdomain.com>` |
| `ALLOWED_EMAILS` | Your email (gh returned no public email for ErfanPY — set manually) |

Without a valid Resend key and verified sender, magic-link login will fail.

**Current local state (check `.env`):** `RESEND_API_KEY` may already be set, but `EMAIL_FROM` and `ALLOWED_EMAILS` must be real values (not `login@example.com` / `your@email.com`). `NEXTAUTH_URL` must be your ngrok HTTPS URL before Zernio webhooks will work.

## 3. HTTPS tunnel (ngrok) — required for inbound DMs

Zernio sends webhooks to `NEXTAUTH_URL/api/zernio/webhook/{workspaceId}`. If `NEXTAUTH_URL` is `http://localhost:3000`, **Instagram DMs will never arrive** — Zernio cannot reach your machine.

### Step A — ngrok account (one time)

1. Sign up: https://dashboard.ngrok.com/signup
2. Copy authtoken: https://dashboard.ngrok.com/get-started/your-authtoken
3. Run:
   ```powershell
   ngrok config add-authtoken YOUR_TOKEN_HERE
   ```

### Step B — start tunnel (every session)

Terminal 3 (keep running):

```powershell
ngrok http 3000
```

Copy the **Forwarding** HTTPS URL, e.g. `https://abc123.ngrok-free.app` (no trailing slash).

### Step C — update `.env` and restart

```env
NEXTAUTH_URL=https://abc123.ngrok-free.app
```

Restart **both** processes:

```powershell
# Terminal 1
npm run dev

# Terminal 2
npm run worker
```

### Step D — re-register Zernio webhook (critical)

OpenReply registered the webhook when `NEXTAUTH_URL` was still localhost. After changing the URL:

1. Open OpenReply at your **ngrok URL** (not localhost) — e.g. `https://abc123.ngrok-free.app`
2. Sign in → **Settings** → Zernio section
3. **Re-save your profile selection** (same profile as before) — this updates the webhook URL in Zernio
4. Confirm `webhookReady` or check ngrok inspector (http://127.0.0.1:4040) for POSTs to `/api/zernio/webhook/...`

### Step E — test again

DM `@yourgrandfatherishere` from another Instagram account → check OpenReply **Inbox**.

**Note:** Free ngrok URLs change every restart. Each time ngrok restarts, repeat steps B–D.

Alternative: [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) for a stable URL.

## 4. Run processes

Terminal 1:

```powershell
cd d:\dev\hobby\openreply
npm run dev
```

Terminal 2:

```powershell
cd d:\dev\hobby\openreply
npm run worker
```

Health check: `GET http://localhost:3000/api/health` — `worker.healthy` must be `true`.

## 5. Sign in

1. Open `http://localhost:3000` (or your ngrok URL).
2. Enter an email listed in `ALLOWED_EMAILS`.
3. Click the magic link from Resend (requires verified domain/sender).

## 6. Connect Zernio (manual — API key not in repo)

1. Sign in as workspace **owner/admin** → **Settings**.
2. Paste your **new** Zernio API key (unrestricted read/write, **Inbox access**).
3. Select the Zernio profile that owns `@yourgrandfatherishere`.
4. Import Instagram account id `6ab389f38d284ffb213303af`.
5. OpenReply auto-registers its signed webhook to `NEXTAUTH_URL` — do not replace it manually in Zernio.
6. Do **not** create matching automations in Zernio (avoids duplicate replies).

If webhooks stop after an ngrok URL change: update `NEXTAUTH_URL`, restart dev + worker, re-save Zernio connection in Settings.

## 7. Test inbound DM

1. Confirm `/api/health` shows `worker.healthy: true`.
2. From a **different** Instagram account, DM `@yourgrandfatherishere` (plain text or share a post/reel).
3. Verify in OpenReply **Inbox** and optionally in Postgres `WebhookEvent`.
4. If nothing arrives:
   - `NEXTAUTH_URL` must match ngrok URL exactly
   - Re-save Zernio connection in Settings
   - Check ngrok request inspector for POSTs to OpenReply webhook routes

## VPS deployment (130.185.76.124)

SSH: `ssh vps` (uses `C:/Users/lenovo/Downloads/private-key-file.pem` as `root`).

```powershell
cd d:\dev\hobby\openreply
$env:OPENREPLY_PUBLIC_URL = "http://130.185.76.124:3200"
.\infra\deploy-vps.ps1
```

Stack: Docker Compose (`infra/docker-compose.vps.yml`) — postgres, redis, web, worker, cron.

After deploy: sign in at the public URL, **re-save Zernio profile** in Settings, test inbound DM.

**Zernio + Iran:** webhooks may require HTTPS and may not reach all regions — test at `http://130.185.76.124:3200/api/health` and re-save Zernio in Settings after deploy (fresh DB).


## Future phases (not this setup)

- Text-file collector (second Zernio webhook → JSONL)
- Telegram bot for shared Instagram links
- NotebookLM pipeline (see `d:\dev\hobby\record-to-notebooklm`)
