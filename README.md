# GAGS Conflict Monitor — Launch Guide

**Go from zero to live in under 30 minutes.**

---

## What You're Deploying

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Vercel     │ ──► │   Railway    │ ──► │  Supabase    │
│  (Frontend)  │     │  (API +      │     │  (Postgres)  │
│  Next.js     │     │   Worker)    │     │              │
│  FREE        │     │   $5/mo      │     │  FREE        │
└─────────────┘     └──────────────┘     └──────────────┘
     ▲                     │
     │              SSE live push
     └──── polls /api/events ────┘
```

**Total cost: ~$5/month.** Everything else is free tier.

---

## Step 1: Create Database (5 minutes)

### Supabase (recommended)

1. Go to **https://supabase.com** → Sign up → **New Project**
2. Name: `gags-conflict-monitor`
3. Set a strong database password → **copy it somewhere safe**
4. Region: pick closest to you (Middle East / Singapore)
5. Wait ~2 minutes for provisioning
6. Go to **SQL Editor** (left sidebar)
7. Paste the ENTIRE contents of `sql/schema.sql` → click **Run**
8. You should see "Success" — tables created + sample data seeded
9. Go to **Settings → Database** → copy the **Connection string (URI)**
   - It looks like: `postgresql://postgres.xxxx:PASSWORD@xxxx.supabase.co:5432/postgres`

✅ Database ready with 16 sample events already loaded.

---

## Step 2: Deploy Backend (10 minutes)

### Railway (recommended — stays awake, unlike Render free)

1. Push this repo to GitHub:
   ```bash
   cd gags-monitor
   git init
   git add .
   git commit -m "initial commit"
   gh repo create gags-conflict-monitor --private --push
   ```

2. Go to **https://railway.app** → Sign up with GitHub

3. **New Project → Deploy from GitHub Repo** → select `gags-conflict-monitor`

4. Railway will detect the repo. Click on the service → **Settings**:
   - **Root Directory**: `backend`
   - **Build Command**: `pip install -r requirements.txt`
   - **Start Command**: `gunicorn main:app -w 2 -k uvicorn.workers.UvicornWorker -b 0.0.0.0:$PORT --timeout 120`

5. Go to **Variables** tab → add:
   ```
   DATABASE_URL = <your Supabase connection string from Step 1>
   ALLOWED_ORIGINS = *
   POLLING_INTERVAL = 120
   LOG_LEVEL = INFO
   ```

6. Go to **Settings → Networking** → **Generate Domain**
   - You'll get something like: `gags-api-production.up.railway.app`

7. Wait for deploy (~2 minutes). Test it:
   ```
   https://YOUR-RAILWAY-DOMAIN/api/health
   https://YOUR-RAILWAY-DOMAIN/api/events
   https://YOUR-RAILWAY-DOMAIN/api/health/ingestion
   ```

✅ Backend live. Worker is polling 12 RSS feeds + GDELT every 2 minutes.

---

## Step 3: Deploy Frontend (5 minutes)

### Vercel

1. Go to **https://vercel.com** → Sign up with GitHub

2. **Import Project** → select `gags-conflict-monitor`

3. Configure:
   - **Root Directory**: `frontend`
   - **Framework Preset**: Next.js (auto-detected)

4. **Environment Variables**:
   ```
   NEXT_PUBLIC_API_URL = https://YOUR-RAILWAY-DOMAIN
   ```

5. Click **Deploy**

6. You'll get a URL like: `gags-conflict-monitor.vercel.app`

✅ Frontend live. MapLibre GL map + live SSE feed working.

---

## Step 4: Lock Down CORS (2 minutes)

Go back to Railway → Variables → update:
```
ALLOWED_ORIGINS = https://gags-conflict-monitor.vercel.app
```

Redeploy. Now only your frontend can call the API.

---

## Step 5: Set Up Monitoring (5 minutes)

### UptimeRobot (free)

1. Go to **https://uptimerobot.com** → Sign up
2. Add monitors:
   - **Health**: `https://YOUR-RAILWAY-DOMAIN/api/health` (every 5 min)
   - **Ingestion**: `https://YOUR-RAILWAY-DOMAIN/api/health/ingestion` (every 5 min)
3. Set alert contacts: your email

### Telegram Alerts (optional)

1. Message **@BotFather** on Telegram → `/newbot` → get token
2. Message your bot once, then visit:
   `https://api.telegram.org/bot<TOKEN>/getUpdates`
   to get your chat ID
3. Add to Railway variables:
   ```
   TELEGRAM_BOT_TOKEN = <your token>
   TELEGRAM_CHAT_ID = <your chat id>
   ```

---

## Step 6: Share with Your Test Group

Send them this link:
```
https://gags-conflict-monitor.vercel.app
```

They'll see:
- Disclaimer modal on first visit
- Interactive map with MapLibre GL (drag, zoom, click markers)
- Global/UAE toggle
- Live event feed with Official/Unofficial badges
- Real-time updates via SSE
- Detail drawer with source classification

---

## Custom Domain (Optional)

### Vercel:
1. Go to your project → **Settings → Domains**
2. Add: `monitor.yourdomain.com`
3. Add CNAME record at your DNS: `cname.vercel-dns.com`

### Railway:
1. Go to **Settings → Networking → Custom Domain**
2. Add: `api.yourdomain.com`
3. Add CNAME record at your DNS

Then update:
- Railway: `ALLOWED_ORIGINS = https://monitor.yourdomain.com`
- Vercel: `NEXT_PUBLIC_API_URL = https://api.yourdomain.com`

---

## How the Ingestion Worker Knows What to Track

The worker polls **12 RSS feeds + GDELT every 120 seconds**:

| Source | Type | Classification |
|--------|------|---------------|
| WAM (UAE) | Government | 🟢 OFFICIAL |
| Saudi Press Agency | Government | 🟢 OFFICIAL |
| KUNA (Kuwait) | Government | 🟢 OFFICIAL |
| BNA (Bahrain) | Government | 🟢 OFFICIAL |
| IRNA (Iran) | Government | 🟢 OFFICIAL |
| Reuters | Wire service | 🟣 UNOFFICIAL |
| BBC World | Media | 🟣 UNOFFICIAL |
| Al Jazeera | Media | 🟣 UNOFFICIAL |
| The National | Media | 🟣 UNOFFICIAL |
| Al Arabiya | Media | 🟣 UNOFFICIAL |
| Gulf News | Media | 🟣 UNOFFICIAL |
| Khaleej Times | Media | 🟣 UNOFFICIAL |
| GDELT | Aggregator | 🟣 UNOFFICIAL |

**Relevance filter**: Articles must match ≥2 keywords from the Iran-Gulf conflict lexicon (60+ terms covering actors, locations, military, diplomatic, and casualty language).

**Dedup**: SHA-256 hash of URL prevents duplicate ingestion.

---

## Architecture Decisions for Production

| Decision | Why |
|----------|-----|
| Railway not Render | Render free tier sleeps after 15 min — kills the ingestion worker |
| Supabase not local PG | Managed backups, connection pooling, dashboard for free |
| SSE not WebSocket | Simpler, HTTP-native, works through CDNs/proxies |
| Append-only events | Events never deleted, only revised — full audit trail |
| In-process worker | Single container = simpler deploy. Split to Celery if >50k events/day |
| Rate limiting (30/min) | Prevents abuse without blocking legitimate testers |
| GZip middleware | 60-70% smaller API responses |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| No events loading | Check `NEXT_PUBLIC_API_URL` in Vercel matches Railway domain |
| CORS errors | Update `ALLOWED_ORIGINS` in Railway to match your Vercel URL |
| Worker not polling | Check Railway logs. Verify `DATABASE_URL` is correct |
| Map not rendering | MapLibre needs HTTPS. Vercel provides this automatically |
| SSE disconnecting | Normal — frontend auto-reconnects. Check Railway logs for errors |
| "Rate limit exceeded" | You're hitting >30 requests/min from one IP. Normal browsing won't trigger this |

---

## What's Next (Production Hardening Roadmap)

1. **Redis pub/sub** — Replace SSE polling with Redis for >100 concurrent users
2. **Semantic dedup** — TF-IDF similarity to catch rephrased duplicates
3. **Manual override API** — Password-protected endpoint for corrections
4. **Backup/restore** — Supabase handles this, but test recovery
5. **Custom domain + Cloudflare** — DDoS protection + caching
6. **Mobile PWA** — Add manifest.json for install-to-homescreen
# Conflict-Monitor
