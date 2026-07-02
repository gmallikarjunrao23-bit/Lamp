# KARTHIK×CORE v2 — Railway Deploy Steps (phone-only, no PC needed)

Repo needs these files (same as before, `public/index.html` is the new
redesigned version):
- `package.json`
- `server.js`
- `db.js`
- `public/index.html`
- `ARCHITECTURE.md` (reference only, not required to run)

## 1. Upload to GitHub
"Add file → Create new file" for each path above — GitHub auto-creates the
`public/` folder when you type `public/index.html` as the path.

## 2. Create the project on Railway
1. Railway dashboard → **New Project → Deploy from GitHub repo**
2. Select your repo
3. Railway auto-detects Node.js and runs `npm install` + `npm start`.

## 3. Add PostgreSQL
1. Same Railway project → **New → Database → Add PostgreSQL**
2. Open your **web service** → **Variables** tab
3. **Add Variable Reference** → `DATABASE_URL` = `${{Postgres.DATABASE_URL}}`
4. Redeploy if it doesn't restart automatically.

## 4. Generate a public domain
Web service → **Settings → Networking → Generate Domain**

## 5. Done
- Visit your domain → boot sequence → key gate.
- Admin: `https://<your-domain>/#admin` → password `Karthik#1234` →
  **Manually Issue a Key** to generate your first key.

## Notes
- `db.js` only forces SSL when it detects a `render.com` host in
  `DATABASE_URL`, or if you set `PGSSL=true` yourself — Railway's internal
  connection doesn't need it, so nothing to configure.
- Admin password lives in `server.js` (top of file) — edit and push to
  redeploy.
- You currently have an earlier version live on Render
  (`prabhas-1.onrender.com`). This is a separate deployment — the Render one
  will keep running until you delete that service. Let me know if you want
  to fully move off Render or keep both for now.

