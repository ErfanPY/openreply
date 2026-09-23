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

```powershell
cd d:\dev\hobby\openreply
docker compose up -d
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
| `DATABASE_URL` | `postgresql://postgres:postgres@localhost:5432/openreply` |
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

## 2. HTTPS tunnel (ngrok)

```powershell
ngrok http 3000
```

Copy the `https://....ngrok-free.app` URL into `NEXTAUTH_URL` in `.env`, then restart the dev server.

If ngrok is not installed: [ngrok download](https://ngrok.com/download) or use [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/).

## 3. Run processes

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

## 4. Sign in

1. Open `http://localhost:3000` (or your ngrok URL).
2. Enter an email listed in `ALLOWED_EMAILS`.
3. Click the magic link from Resend (requires verified domain/sender).

## 5. Connect Zernio (manual — API key not in repo)

1. Sign in as workspace **owner/admin** → **Settings**.
2. Paste your **new** Zernio API key (unrestricted read/write, **Inbox access**).
3. Select the Zernio profile that owns `@yourgrandfatherishere`.
4. Import Instagram account id `6ab389f38d284ffb213303af`.
5. OpenReply auto-registers its signed webhook to `NEXTAUTH_URL` — do not replace it manually in Zernio.
6. Do **not** create matching automations in Zernio (avoids duplicate replies).

If webhooks stop after an ngrok URL change: update `NEXTAUTH_URL`, restart dev + worker, re-save Zernio connection in Settings.

## 6. Test inbound DM

1. Confirm `/api/health` shows `worker.healthy: true`.
2. From a **different** Instagram account, DM `@yourgrandfatherishere` (plain text or share a post/reel).
3. Verify in OpenReply **Inbox** and optionally in Postgres `WebhookEvent`.
4. If nothing arrives:
   - `NEXTAUTH_URL` must match ngrok URL exactly
   - Re-save Zernio connection in Settings
   - Check ngrok request inspector for POSTs to OpenReply webhook routes

## VPS deployment (later)

| Component | Suggested host |
| --- | --- |
| Web (`next start` or `npm run dev`) | Vercel, or VPS + nginx + PM2 |
| Worker (`npm run worker`) | Always-on VPS (Railway, Render, existing VPS) |
| Postgres + Redis | Managed (Neon + Upstash) or Docker on VPS |

Update `NEXTAUTH_URL` to production domain and re-save Zernio connection so webhooks target the new URL.

## Future phases (not this setup)

- Text-file collector (second Zernio webhook → JSONL)
- Telegram bot for shared Instagram links
- NotebookLM pipeline (see `d:\dev\hobby\record-to-notebooklm`)
