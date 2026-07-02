# KARTHIK×CORE — Full Architecture

## What this is
A private, invite-only personal site for G Karthik. Visitors need an access
key to get in. Once in, they see a profile section and CORE — a console with
4 hidden personas that hold a real conversation (with memory). An admin panel
lets Karthik see access requests and issue/revoke keys — no database GUI
needed.

## System diagram
```
┌─────────────────────┐         ┌──────────────────────────┐        ┌─────────────────┐
│   Browser (visitor)  │  HTTP   │   Render Web Service       │  SQL   │   Render          │
│   public/index.html  │◄──────► │   server.js (Express)      │◄──────►│   PostgreSQL       │
│   - gate              │         │   /api/* routes             │        │   kv_store table    │
│   - profile            │         │   db.js (pg pool + helpers) │        └─────────────────┘
│   - console (chat)     │         └──────────────────────────┘
└─────────────────────┘                        │
                                                 │ HTTP (server-side only)
                                                 ▼
                                    ┌───────────────────────────┐
                                    │  4 persona endpoints        │
                                    │  gpt5 / gemini / llama /    │
                                    │  copilot .adi7ya.workers.dev│
                                    └───────────────────────────┘
```

Visitors never talk to the 4 persona endpoints directly — `server.js` calls
them and relays the reply. This is why no model name ever reaches the
frontend.

## File map
```
karthik-core/
├── package.json          → dependencies (express, pg)
├── server.js              → all backend routes, in one file
├── db.js                  → Postgres connection + key-value helpers
└── public/
    └── index.html          → entire frontend (HTML + CSS + JS, no build step)
```

## Data model
Everything lives in one table, used as a generic key-value store:

```sql
kv_store (
  key         TEXT PRIMARY KEY,
  value       JSONB NOT NULL,
  updated_at  TIMESTAMPTZ DEFAULT now()
)
```

Key naming conventions:
| Key pattern                     | Holds                                   |
|----------------------------------|------------------------------------------|
| `keys:{ACCESS-KEY}`              | `{ issuedTo, issuedAt, active }`          |
| `requests:{timestamp}-{rand}`    | `{ name, session, timestamp, status }`    |
| `chat:{session}:{mode}`          | array of `{ role, text }` messages         |

Table is created automatically on first boot (`initDb()` in `db.js`) — no
manual migration step.

## Request flow: getting access
1. Visitor opens the site → `index.html` runs a boot animation, then shows
   the key form.
2. They either paste a key, or tap **Request access on WhatsApp** — this
   POSTs to `/api/request-access`, writing a `requests:` row, then opens a
   pre-filled WhatsApp chat to Karthik's number.
3. Karthik opens `/#admin`, logs in, sees the request under **Pending
   Requests**, taps **Issue Key** → a `keys:` row is created and the request
   is marked `approved`.
4. Karthik sends the key to the visitor (manually, via WhatsApp).
5. Visitor pastes the key → `/api/verify-key` checks `keys:{KEY}.active` →
   if true, the key is stored in the visitor's `localStorage` and the site
   unlocks. Future visits skip the gate automatically.

## Request flow: chatting with CORE
1. Visitor picks a mode (PULSE / NOVA / APEX / ECHO) — purely a label change
   client-side.
2. On send, the frontend POSTs `{ mode, session, message }` to `/api/chat`.
3. `server.js`:
   - loads the last 8 turns of that session+mode's history from `kv_store`
   - builds one prompt: a persona instruction + recent conversation + the
     new message
   - calls the matching persona endpoint server-side
   - saves the updated history back to `kv_store`
   - returns `{ reply, history }`
4. Frontend just renders whatever `history` it gets back — it holds no
   conversation state of its own, so a page reload doesn't lose anything.

## Admin authentication
Simple shared-secret model: `server.js` checks a header `x-admin-pass`
against a constant defined at the top of the file. No sessions, no JWTs —
intentionally minimal for a single-admin personal tool.

**Where the password lives:** `server.js`, top of file, `ADMIN_PASSWORD`.
Change it anytime by editing that line in GitHub — Render redeploys on push.
For stronger security later, move it to Render's **Environment** tab as a
variable and read it via `process.env.ADMIN_PASSWORD` instead — happy to
wire that in whenever you want it.

## Design system (frontend)
- **Palette** — void `#0A0A0C`, panel `#131316`, signal (ember) `#FF5F1F`,
  ion (cyan) `#00F0D2`, muted text `#8A8A93`.
- **Type** — Unbounded (display headlines), Manrope (body), Martian Mono
  (labels, console text, timestamps).
- **Signature element** — the canvas-drawn reactor core in the hero: a
  pulsing center with orbiting particles in the ember/ion duality, echoing
  the "CORE" name literally.
- **Console** — styled as a real terminal window (chrome bar, tabs, log
  lines with `❯` / `CORE ▸` prefixes) instead of generic chat bubbles, since
  the product is a private console, not a public chatbot.
- Motion respects `prefers-reduced-motion` throughout (canvas animation,
  boot sequence, and scroll reveals all degrade to a static state).

## Deployment target
Render — see `DEPLOY-STEPS-RENDER.md` for the exact click-by-click steps
(PostgreSQL + Web Service + environment variable wiring).

## Known trade-offs (worth knowing, not urgent)
- Free-tier Render services sleep after inactivity — first request after a
  while takes ~30–50s. Fine for a personal/invite-only site.
- Admin auth is a single shared password, not per-admin accounts — fine for
  one operator, would need real auth if more than one person manages it.
- Chat history has no automatic expiry — over a long time `kv_store` will
  grow. Not a problem at personal-site scale, but worth a cleanup job later
  if it ever matters.

