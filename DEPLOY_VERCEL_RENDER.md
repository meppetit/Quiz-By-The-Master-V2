# Deploy MEP Quiz — Vercel (frontend) + Render (backend) + Supabase (database)

This is the recommended setup for the event. Vercel serves the participant app from a global CDN so the QR-scan rush never touches your server for page loads, Render runs the API, and Supabase holds the data.

Read this top to bottom once. Total time: about 30 minutes.

```
Phone (QR scan)
      |
      v
Vercel  ──/api/*──>  Render (FastAPI)  ──>  Supabase Postgres
(React static)          Mumbai/Singapore        Mumbai
```

**The single most important choice:** put Render in the region closest to your Supabase project. Every database query costs one network round-trip. Same region = ~2 ms. Different continent = ~450 ms per query, and the quiz feels sluggish no matter what else you do.

---

# Part 0 — Before you start

You need:

1. Your code on **GitHub**. In Emergent, use the "Save to GitHub" / push button, or download the code and push it yourself:
   ```bash
   git init && git add . && git commit -m "MEP Quiz"
   git remote add origin https://github.com/<you>/mep-quiz.git
   git push -u origin main
   ```
2. Your **Supabase pooled connection string** (see `SUPABASE_SETUP.md`). It looks like:
   ```
   postgresql://postgres.arfywtfdaovlrzjhqmdq:YOURPASSWORD@aws-0-ap-south-1.pooler.supabase.com:6543/postgres
   ```
   Use the **pooler** host and port **6543**. The direct `db.<ref>.supabase.co` host is IPv6-only and will not connect from Render.
3. Two secrets you generate now and keep:
   ```bash
   openssl rand -hex 32     # -> JWT_SECRET
   ```
   and a strong admin password of your choosing.

Repo layout that matters:

```
/backend        FastAPI app (server.py, db.py, models.py, requirements.txt)
/frontend       React app (package.json, src/, yarn.lock)
/deploy         Docker/compose files (not needed for this guide)
```

---

# Part 1 — Backend on Render

## 1.1 Create the service

1. Go to **https://dashboard.render.com** → **New +** → **Web Service**.
2. Connect your GitHub repo and pick it.
3. Fill in:

| Field | Value |
|---|---|
| Name | `mep-quiz-api` |
| Language | `Python 3` |
| **Region** | **Singapore** (closest Render region to Supabase Mumbai) |
| Branch | `main` |
| **Root Directory** | `backend` |
| Build Command | `pip install -r requirements.txt` || Start Command | `gunicorn server:app -k uvicorn.workers.UvicornWorker -w 4 -b 0.0.0.0:$PORT --timeout 60` |
| Instance Type | **Starter** or higher — *not* Free (Free instances sleep and cold-start ~50 s, fatal on event day) |

> `$PORT` is supplied by Render. Do not hardcode 8001.

## 1.2 Backend environment variables

Render → your service → **Environment** → **Add Environment Variable**, one row each:

| Key | Value | Notes |
|---|---|---|
| `DATABASE_URL` | `postgresql://postgres.<ref>:<password>@aws-0-ap-south-1.pooler.supabase.com:6543/postgres` | Pooler host, port 6543. Paste it raw — the app converts it for the async driver, enables SSL and disables prepared-statement caching for poolers automatically. |
| `JWT_SECRET` | output of `openssl rand -hex 32` | Signs admin sessions. Changing it logs admins out. |
| `ADMIN_USERNAME` | `admin` | Your admin login |
| `ADMIN_PASSWORD` | a long unique password | Do **not** reuse the preview one (`mepquiz2026`) |
| `CORS_ORIGINS` | `https://mep-quiz.vercel.app` | Your exact Vercel URL, no trailing slash. Fill this in after Part 2, then redeploy. |
| `PYTHON_VERSION` | `3.11.9` | Pins the runtime |

Notes:
- If your password contains `$`, `#` or `@`, paste it into Render's field as-is (Render does not do shell expansion). Only `.env` files need quoting.
- Multiple frontends? `CORS_ORIGINS` accepts a comma-separated list: `https://a.vercel.app,https://quiz.yourdomain.com`.

## 1.3 Health check

Render → **Settings** → **Health Check Path**: `/api/health`

It should return:
```json
{"status":"ok","database":"postgresql","question_sets":20}
```

## 1.4 Deploy and verify

Click **Deploy**. When it goes live, note the URL (e.g. `https://mep-quiz-api.onrender.com`) and test:

```bash
curl https://mep-quiz-api.onrender.com/api/health
curl -X POST https://mep-quiz-api.onrender.com/api/admin/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"YOUR_ADMIN_PASSWORD"}'
```

The first call proves the database connection; the second proves your admin credentials. On first boot the app creates its own tables and, if the database is empty, seeds 20 sets.

## 1.5 Scaling for the burst

Render → **Settings** → **Scaling**:

| Expected crowd | Instances | Instance type |
|---|---|---|
| up to 300 | 1 | Starter |
| 300–1500 | 2 | Standard |
| 1500–5000 | 3–4 | Standard |

Each instance already runs 4 worker processes (the `-w 4` in the start command). Scale up the morning of the event; scale back down after.

---

# Part 2 — Frontend on Vercel

## 2.1 Import the project

1. Go to **https://vercel.com/new** and import the same GitHub repo.
2. Configure:

| Field | Value |
|---|---|
| Framework Preset | **Create React App** |
| **Root Directory** | `frontend` |
| Build Command | `yarn build` |
| Output Directory | `build` |
| Install Command | `yarn install` |

## 2.2 Frontend environment variable

Vercel → **Settings** → **Environment Variables**:

| Key | Value | Environments |
|---|---|---|
| `REACT_APP_BACKEND_URL` | `https://mep-quiz-api.onrender.com` | Production, Preview, Development |

Critical details:
- **No trailing slash.** The app appends `/api/...` itself; `.../onrender.com/` would produce `//api/...`.
- It must start with `https://`.
- Create React App bakes this value in **at build time**. If you change it, you must **redeploy** (Deployments → ⋯ → Redeploy) — restarting is not enough.

## 2.3 SPA routing

The app uses client-side routes (`/quiz`, `/completion`, `/admin`, `/admin/live`). Without a rewrite, refreshing `/admin` returns a 404. Add `frontend/vercel.json` (already included in this repo):

```json
{
  "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
}
```

## 2.4 Deploy and close the CORS loop

1. Deploy. Note your URL, e.g. `https://mep-quiz.vercel.app`.
2. Go back to Render → `CORS_ORIGINS` → set it to that exact URL → save (Render redeploys automatically).
3. Open the Vercel URL on your phone and register a test participant. If the button spins and fails, it is almost always `CORS_ORIGINS` or a trailing slash in `REACT_APP_BACKEND_URL`.

## 2.5 Custom domain (optional)

Vercel → **Domains** → add `quiz.yourdomain.com` and follow the DNS instructions. Then update Render's `CORS_ORIGINS` to the new domain (keep the `.vercel.app` one too if you like).

---

# Part 3 — Full environment variable reference

**Render (backend)**
```
DATABASE_URL=postgresql://postgres.<ref>:<password>@aws-0-ap-south-1.pooler.supabase.com:6543/postgres
JWT_SECRET=<openssl rand -hex 32>
ADMIN_USERNAME=admin
ADMIN_PASSWORD=<long unique password>
CORS_ORIGINS=https://mep-quiz.vercel.app
PYTHON_VERSION=3.11.9
```

**Vercel (frontend)**
```
REACT_APP_BACKEND_URL=https://mep-quiz-api.onrender.com
```

**Never** put `DATABASE_URL`, `JWT_SECRET` or `ADMIN_PASSWORD` in the frontend. Anything named `REACT_APP_*` is shipped to the browser in plain text.

---

# Part 4 — Day-before checklist

1. `curl https://<render-url>/api/health` → `question_sets: 20`.
2. Open `https://<vercel-url>/admin/login`, sign in with your production admin password.
3. **Set health** tab → all 20 sets **READY**. If a set is flagged, fix it on the **Questions** tab.
4. Do one complete run on a real phone over mobile data, not office wifi.
5. Clear your test data:
   ```bash
   TOKEN=$(curl -s -X POST https://<render-url>/api/admin/login \
     -H "Content-Type: application/json" \
     -d '{"username":"admin","password":"YOUR_PASSWORD"}' | python3 -c "import sys,json;print(json.load(sys.stdin)['access_token'])")
   curl -X POST https://<render-url>/api/admin/reset-attempts -H "Authorization: Bearer $TOKEN"
   ```
6. Scale Render to the instance count you planned.
7. Open the Supabase dashboard once so the project is warm (free projects idle out).
8. Generate the QR code pointing at `https://<your-domain>/` — the bare root, not `/quiz`.
9. Open `https://<your-domain>/admin/live` on the projector laptop and press **Fullscreen**.

---

# Part 5 — During the event

- Projector: `/admin/live` (refreshes every 5 seconds on its own).
- Laptop: `/admin` for stats, search and the leaderboard.
- If registration slows: add a Render instance. Do not restart the database.
- At the end: **Export CSV** from the dashboard *before* you shut anything down.

---

# Part 6 — Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Frontend loads, registration fails silently | CORS mismatch | Set `CORS_ORIGINS` to the exact frontend origin, no trailing slash |
| Calls go to `//api/register` | Trailing slash in `REACT_APP_BACKEND_URL` | Remove it and **redeploy** Vercel |
| Refreshing `/admin` gives 404 | Missing SPA rewrite | Add `frontend/vercel.json` (Part 2.3) |
| `No address associated with hostname` | Using the IPv6-only direct Supabase host | Switch to the `...pooler.supabase.com` host, port 6543 |
| `password authentication failed` | `[YOUR-PASSWORD]` placeholder left in, or wrong user | User must be `postgres.<project-ref>` for the pooler |
| `prepared statement already exists` | Pooler without cache disabled | Ensure the host contains `pooler` so the app disables caching, or use port 5432 |
| First request after idle takes ~50 s | Free Render instance sleeping | Upgrade to Starter+ and set instances ≥ 1 |
| Quiz feels slow (~0.5 s per tap) | Backend and database in different regions | Move Render to Singapore/Mumbai |
| Admin login works then 401s | `JWT_SECRET` changed | Log out and back in |
| "No question sets available" | A set has no questions | Import questions, then re-check Set health |

---

# Part 7 — After the event

```bash
# save the results
pg_dump "postgresql://postgres.<ref>:<password>@aws-0-ap-south-1.pooler.supabase.com:6543/postgres" > mep-quiz-$(date +%F).sql
```

Then export the CSV from the admin dashboard, scale Render back to 1 instance (or suspend it), and rotate `ADMIN_PASSWORD` and the Supabase database password.
