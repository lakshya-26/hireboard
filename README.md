# HireBoard

**HireBoard** is a full-stack job application tracker: move opportunities through a **Kanban** or **list** view, capture **notes**, **contacts**, and **email reminders**, see **pipeline analytics**, and **export** data to CSV. Accounts use **email + password** with **OTP verification** before login; due reminders are delivered by **email** on a schedule.

This repository is organized as two apps:

| Directory | Description |
|-----------|-------------|
| [`hireboard-backend`](./hireboard-backend/) | Express REST API, MongoDB (Mongoose), JWT + refresh cookies, Redis (OTP + reminder locks), Nodemailer, `node-cron` |
| [`hireboard-frontend`](./hireboard-frontend/) | React 18 SPA (Vite, Tailwind CSS 4), React Router, drag-and-drop board |

---

## Features

- **Auth** — Register, login, JWT access token (Bearer) + httpOnly refresh cookie, logout  
- **Email verification** — 6-digit OTP in Redis (10-minute TTL), Nodemailer; login blocked until verified; resend / verify flows  
- **Applications** — CRUD, filters (status in list view, work type, priority, applied date range, search), pagination-friendly list API  
- **Kanban** — Drag-and-drop status updates (`@hello-pangea/dnd`); confetti when a card reaches **Offer**  
- **Application detail** — Editable fields, notes, contacts, reminders (create, list, mark complete)  
- **Stats** — Dashboard charts (overview endpoint)  
- **Reminders** — Cron job (every 5 minutes, timezone configurable) emails due reminders; `sent` only after successful send; Redis lock to reduce duplicate sends  
- **CSV export** — Filtered export aligned with list filters  
- **UI** — Responsive layout (filter rail on desktop, drawer on mobile), fixed navbar, modal scroll lock, SPA-friendly hosting  

---

## Tech stack

**Backend**

- Node.js 18+, Express 4, Mongoose 8  
- Zod validation, centralized error responses  
- `bcrypt`, `jsonwebtoken`, `cookie-parser`, `cors`  
- **Redis** (`ioredis`) — OTP storage, reminder dispatch locks  
- **Nodemailer** — verification and reminder emails  
- **node-cron** — due reminder processing  

**Frontend**

- React 18, React Router 7, Vite 6  
- Tailwind CSS 4, Axios (credentials for refresh)  
- Recharts (analytics), canvas-confetti (Offer)  
- `@hello-pangea/dnd` (Kanban)

---

## Prerequisites

- **Node.js** ≥ 18  
- **MongoDB** (local or Atlas)  
- **Redis** (local or cloud URL)  
- **SMTP** credentials for sending mail (verification + reminders)

---

## Quick start

### 1. Clone and install

```bash
git clone <your-repo-url> hireboard
cd hireboard
```

**Backend**

```bash
cd hireboard-backend
cp .env.example .env
# Edit .env — set MONGODB_URI, JWT secrets, REDIS_URL, SMTP_*, EMAIL_FROM, CLIENT_ORIGIN
npm install
npm run dev
```

Default API: `http://localhost:3000` (or `PORT` in `.env`).

**Frontend**

```bash
cd hireboard-frontend
cp .env.example .env
# Set VITE_API_BASE_URL to your API base, e.g. http://localhost:3000/api/v1
npm install
npm run dev
```

Default UI: `http://localhost:5173`.

### 2. Health check

```http
GET /api/v1/health
```

---

## Environment variables

### Backend (`hireboard-backend/.env`)

See **[`hireboard-backend/.env.example`](./hireboard-backend/.env.example)** for the full list. Highlights:

| Variable | Purpose |
|----------|---------|
| `MONGODB_URI` | MongoDB connection string |
| `JWT_ACCESS_SECRET` / `JWT_REFRESH_SECRET` | Signing keys (use strong random values in production) |
| `CLIENT_ORIGIN` | Browser origin for CORS + credentialed cookies (e.g. `http://localhost:5173` or your Vercel URL) |
| `REDIS_URL` | Redis for OTP and reminder locks |
| `SMTP_*`, `EMAIL_FROM` | Outbound email |
| `CRON_TIMEZONE` | IANA zone for reminder cron (default `UTC` in example) |

### Frontend (`hireboard-frontend/.env`)

| Variable | Purpose |
|----------|---------|
| `VITE_API_BASE_URL` | API base URL including `/api/v1` (e.g. `https://api.example.com/api/v1`) |

---

## API overview

All JSON routes are under **`/api/v1`** (see [`hireboard-backend/routes/index.js`](./hireboard-backend/routes/index.js) for auto-mounting).

| Area | Methods | Path pattern |
|------|---------|----------------|
| Health | GET | `/api/v1/health` |
| Auth | POST | `/api/v1/auth/register`, `login`, `refresh`, `send-otp`, `verify-otp` |
| Applications | GET, POST | `/api/v1/applications` |
| Applications | GET | `/api/v1/applications/export/csv` |
| Applications | GET, PATCH, DELETE | `/api/v1/applications/:id` |
| Notes | POST | `/api/v1/applications/:id/notes` |
| Reminders | GET, POST | `/api/v1/reminders` |
| Reminders | GET | `/api/v1/reminders/due-count` |
| Reminders | PATCH | `/api/v1/reminders/:id` |
| Stats | GET | `/api/v1/stats/overview` |

Protected routes expect a valid **Bearer** access token (and refresh cookie for `/auth/refresh`).

---

## Frontend routes

| Path | Access |
|------|--------|
| `/login`, `/register`, `/verify-email` | Public |
| `/dashboard`, `/applications/:id` | Authenticated (protected layout) |

---

## Scripts

**Backend** (`hireboard-backend`)

| Command | Description |
|---------|-------------|
| `npm run dev` | Run API with file watch |
| `npm start` | Run API (production-style) |
| `npm run db:sync-indexes` | Sync MongoDB indexes (if used in your workflow) |

**Frontend** (`hireboard-frontend`)

| Command | Description |
|---------|-------------|
| `npm run dev` | Vite dev server |
| `npm run build` | Production build → `dist/` |
| `npm run preview` | Preview production build locally |

---

## Deployment notes

1. **Host the API** on any Node-friendly platform; set **`CLIENT_ORIGIN`** to your **exact** frontend origin (scheme + host, no trailing slash mismatch).  
2. **Host the SPA** (e.g. Vite build on **Vercel**). Set **`VITE_API_BASE_URL`** to the deployed API **`/api/v1`** base.  
3. **SPA refresh** — Configure the host so unknown paths serve `index.html` (e.g. a **`vercel.json`** `rewrites` entry) so `/dashboard` and `/applications/:id` do not 404 on refresh.  
4. **Redis + SMTP** must be reachable from the API process for verification and reminder email.

---

## Repository layout (high level)

```
hireboard/
├── README.md                 ← this file
├── hireboard-backend/
│   ├── index.js              # Express entry, CORS, cookies, routes, cron
│   ├── config/               # database, auth env, redis, OTP/email env
│   ├── models/               # User, Application, Reminder
│   ├── routes/               # *.route.js → mounted under /api/v1/...
│   ├── controllers/
│   ├── services/
│   ├── validators/
│   ├── jobs/                 # reminder email cron
│   └── ...
└── hireboard-frontend/
    ├── index.html
    ├── src/
    │   ├── App.jsx
    │   ├── pages/
    │   ├── components/
    │   ├── context/
    │   └── lib/
    └── ...
```

---

## Contributing

Issues and PRs are welcome. Please keep changes focused and match existing patterns (thin controllers, service layer, Zod on the boundary).

---

## Acknowledgements

Built as a practical pipeline for job search workflows — Kanban UX inspired by tools teams already know (Jira / ClickUp–style clarity), with email and scheduling handled explicitly for reliability.
