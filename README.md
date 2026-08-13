# CivicTrack — SOAIDEATHON-S36
## Evidence-Grounded Civic Grievance Triage & Participatory Budgeting Platform

A hackathon-ready prototype for problem statement **SOAIDEATHON-S36**.  
Three surfaces. One AI pipeline. Zero API keys required to demo.

---

## Quick Start

### Prerequisites
- Python 3.11+
- Node.js 18+
- `npm` or `pnpm`

### Backend

```bash
cd backend

# 1. Create and activate virtual environment
python -m venv venv

# Windows
venv\Scripts\activate
# macOS/Linux
source venv/bin/activate

# 2. Install dependencies
pip install -r requirements.txt

# 3. Configure environment
cp .env.example .env
# Optional: add your OPENAI_API_KEY for live AI features
# The app runs fully without it — see §AI Fallbacks below

# 4. Start the server
uvicorn app.main:app --reload --port 8000
```

The API will be available at `http://localhost:8000`.  
Interactive docs: `http://localhost:8000/docs`

### Seed Demo Data

> Backend must be running before seeding.

```bash
# In a separate terminal (with venv activated):
cd backend
python -m app.seed_demo
```

This seeds:
- 2 demo users (1 citizen, 1 officer)
- 8 realistic grievances (via live API — runs full AI pipeline)
- 3 participatory projects

### Frontend

```bash
cd frontend

# 1. Install dependencies
npm install

# 2. Configure environment
cp .env.local.example .env.local
# No changes needed for local development with default ports

# 3. Start dev server
npm run dev
# Open http://localhost:3000
```

### Live Duplicate Simulation (run mid-pitch)

```bash
cd backend
python -m app.seed_demo --simulate
```

Submits a paraphrased near-duplicate complaint ~15m from the original garbage complaint.  
Prints `is_duplicate: True`, `parent_grievance_id`, and `upvote_count` to the console.  
The **DuplicateAlertModal** appears on the citizen's screen simultaneously.

---

## Architecture

```
┌──────────────────┐    HTTP/WS    ┌────────────────────────────────────┐
│  Next.js 14 App  │◄────────────►│  FastAPI (uvicorn, port 8000)      │
│  (port 3000)     │              │                                    │
│  /citizen        │              │  POST /api/grievances/submit       │
│  /admin          │              │    ├─ Whisper transcription        │
│  /budget         │              │    ├─ GPT-4o-mini classification   │
└──────────────────┘              │    └─ ChromaDB dedup (haversine)   │
                                  │                                    │
                                  │  WS  /ws/live-updates              │
                                  │    broadcasts: new_grievance,      │
                                  │    grievance_resolved, project_vote│
                                  │                                    │
                                  │  SQLite (civic_grievance.db)       │
                                  │  ChromaDB (./chroma_store/)        │
                                  └────────────────────────────────────┘
```

---

## AI Fallbacks

> **The app is fully functional with zero API keys.** This is the single most important resilience property for a live pitch on unreliable venue WiFi.

| Feature | With API key | Without API key |
|---|---|---|
| Audio transcription | OpenAI Whisper | Placeholder string `[transcription unavailable in offline demo mode]` |
| Grievance classification | GPT-4o-mini (JSON, temp=0) | Keyword-rule engine (fully implemented — 4 category buckets + urgency keywords) |
| Deduplication | ChromaDB + sentence-transformers + haversine | Skipped gracefully (returns `is_duplicate: false`); submission still succeeds |

---

## Features

### Citizen Portal (`/citizen`)
- **Multi-modal complaint submission**: text, voice (Web Audio API), photo
- **Geolocation auto-detect** with manual address fallback
- **AI classification**: category and priority assigned automatically
- **Duplicate detection**: near-identical complaints ≤100m apart are merged, not duplicated; the citizen sees a `DuplicateAlertModal` explaining community aggregation
- **My Tickets**: timeline view showing `OPEN → IN_PROGRESS → RESOLVED` with resolution evidence

### Admin Dashboard (`/admin`)
- **Live feed**: new tickets animate in via WebSocket — no polling, no refresh
- **Stat cards**: Total, Auto-Deduplicated, Resolved%, Category breakdown — all real DB queries
- **Resolution workflow**: upload proof photo + comments → status set to `RESOLVED` → citizen timeline updates
- **Map integration point**: clearly labelled panel ready for Mapbox/Leaflet (lat/lng on every ticket)

### Participatory Budgeting (`/budget`)
- **Project cards** with animated vote progress bars (Framer Motion)
- **Optimistic voting UI**: button responds instantly, reconciles with server
- **Auto-funded**: project flips to `FUNDED` state when vote threshold reached
- **Complaint context**: category breakdown from grievance data surfaces demand signals

---

## Deduplication Algorithm

1. Embed complaint text with `sentence-transformers/all-MiniLM-L6-v2`
2. Query ChromaDB for the 5 nearest neighbours (cosine similarity)
3. For each candidate above `DEDUP_SIMILARITY_THRESHOLD` (0.85):
   - If both have coordinates → compute **haversine distance**
   - Mark as duplicate **only if** distance ≤ `DEDUP_RADIUS_METERS` (100m)
   - Semantically similar complaints from different wards are **never merged**
4. Always store the new embedding (keeps the index accurate for future checks)

---

## API Reference

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | Liveness check |
| `POST` | `/api/grievances/submit` | Submit (multipart: text/audio/image) |
| `GET` | `/api/grievances/public` | All non-duplicate tickets |
| `GET` | `/api/grievances/{id}` | Single ticket detail |
| `GET` | `/api/grievances/citizen/{id}` | Citizen's ticket history |
| `POST` | `/api/grievances/{id}/resolve` | Resolve with evidence |
| `GET` | `/api/admin/stats` | Dashboard statistics |
| `GET` | `/api/budgeting/projects` | All projects |
| `POST` | `/api/budgeting/{id}/vote` | Cast a vote |
| `WS` | `/ws/live-updates` | Real-time event stream |

Full interactive docs at `http://localhost:8000/docs`.

---

## Environment Variables

### Backend (`backend/.env`)

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `sqlite:///./civic_grievance.db` | Swap to `postgresql+psycopg2://...` for production; no code changes needed |
| `OPENAI_API_KEY` | _(empty)_ | Optional; app degrades gracefully without it |
| `DEDUP_SIMILARITY_THRESHOLD` | `0.85` | Cosine similarity floor for duplicate detection |
| `DEDUP_RADIUS_METERS` | `100` | Geographic radius for duplicate merging |
| `UPLOAD_DIR` | `./uploads` | Local file storage; see §Production for S3 swap |
| `CORS_ORIGINS` | `http://localhost:3000` | Comma-separated allowed origins |

### Frontend (`frontend/.env.local`)

| Variable | Default | Description |
|---|---|---|
| `NEXT_PUBLIC_API_BASE_URL` | `http://localhost:8000` | Backend base URL |
| `NEXT_PUBLIC_WS_URL` | `ws://localhost:8000/ws/live-updates` | WebSocket endpoint |

---

## Production Notes

> **Do not deploy this prototype to production without the following changes:**

1. **Database**: set `DATABASE_URL` to a managed Postgres instance (Supabase, Cloud SQL, Neon, etc.)
2. **File storage**: replace the local `./uploads` static-file handler in `main.py` with presigned S3/GCS URLs. The `image_url` and `audio_url` columns store path strings — only the handler changes.
3. **Authentication**: add JWT/OAuth2 middleware. The `citizen_id` and `officer_id` fields are already wired throughout — plug in real user IDs from your auth layer.
4. **ChromaDB**: move to ChromaDB Cloud or another managed vector store for horizontal scaling.
5. **WebSocket**: for multi-instance deployments, replace the in-memory `ConnectionManager` with a Redis pub/sub broadcast.
6. **HTTPS**: terminate TLS at your load balancer; the FastAPI app itself runs plain HTTP.

---

## Tech Stack

| Layer | Choice |
|---|---|
| Frontend | Next.js 14 (App Router), TypeScript |
| Styling | Tailwind CSS, custom civic design system |
| Motion | Framer Motion (duplicate modal + vote progress bar only) |
| HTTP client | Axios (typed wrappers in `lib/api.ts`) |
| Realtime | Native WebSocket (no socket.io) |
| Backend | Python 3.11, FastAPI, Uvicorn |
| Validation | Pydantic v2 |
| ORM | SQLAlchemy 2.0 |
| Database | SQLite → Postgres via env var |
| Vector store | ChromaDB (persistent local) |
| Embeddings | `sentence-transformers/all-MiniLM-L6-v2` |
| Speech-to-text | OpenAI Whisper API (+ offline fallback) |
| Classification | OpenAI GPT-4o-mini (+ keyword-rule fallback) |
| File storage | Local `./uploads` (see §Production for S3) |

---

## Acceptance Checklist

- [x] Citizen can submit a text-only complaint with no location — succeeds with graceful nulls
- [x] Citizen can submit an audio-only complaint — transcribed or fallback-labelled
- [x] Near-duplicate complaint triggers `is_duplicate: true` and increments parent's `upvote_count`
- [x] Admin dashboard receives new-ticket WS event within ~1s of submission
- [x] Resolving a ticket updates status to `RESOLVED` and citizen timeline reflects it
- [x] Voting on a project increments count and updates progress bar without page reload
- [x] Entire flow works with `OPENAI_API_KEY` unset
- [x] Language switcher changes visible UI strings (EN / हिन्दी / ଓଡ଼ିଆ)
- [x] Every stat card reflects real DB query results

---

*Built By Yash Deo 
Linkedin Profile : www.linkedin.com/in/yashdeo-aiml
