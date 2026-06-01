# ScoutEats — NYC restaurant inspections, visualized

A searchable, filterable explorer for NYC DOHMH restaurant health inspections
(Socrata `43nn-pn8j`), focused on restaurants **Closed by DOHMH**. Click a card to
get an **AI-assisted "lease takeover" plan** — actionable steps derived from the
enriched inspection data.

Two halves:

| Half | Stack | Dir | Dev port |
|------|-------|-----|----------|
| **Backend** (`scouteats_intel`) | FastAPI + SQLAlchemy 2.0 + async httpx | `backend/` | 8099 |
| **Frontend** | TanStack Start + Vite 7 + React 19 + shadcn/ui | `frontend/` | 8080 |

The frontend calls the backend's `GET /listings` (card feed) and `POST /analysis/enrich`
(the takeover modal). With `VITE_API_URL=""` it runs off the bundled
`frontend/public/data/inspections.json` snapshot and a client-side enrichment fallback —
so the UI works even with no backend.

---

## Run it locally

Two terminals — backend on `:8099`, frontend on `:8080`.

### 1. Backend (FastAPI) — terminal 1

```bash
cd backend
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
uvicorn scouteats_intel.api:app --reload --port 8099
```

Defaults to local **SQLite** (`scouteats.db`, auto-created on startup) and **codify.cafe
is off** unless you set a token — so you get the deterministic local takeover plan. No
Postgres or P2X setup required.

Sanity-check the enrichment endpoint (new terminal):

```bash
curl -s -X POST http://localhost:8099/analysis/enrich \
  -H 'Content-Type: application/json' \
  -d '{"restaurant":{"camis":"40365938","name":"Nancy'\''s Restaurant","borough":"Queens"},
       "summary":{"total_violations":4,"critical_violations":3,"risk":"high","is_closed":true}}' \
  | python3 -m json.tool
```

You should see a `{card, risk_assessment, takeover, enrichment}` envelope with
`takeover.steps` populated and `risk_assessment.source: "local_fallback"`.

### 2. Frontend (TanStack Start) — terminal 2

```bash
cd frontend
npm install
npm run dev        # http://localhost:8080
```

Open http://localhost:8080 and **click any restaurant card** → a modal opens, calls
`POST /analysis/enrich`, and renders the risk badge, takeover steps, and a **codify.cafe**
button.

The frontend points at `http://localhost:8099` by default. Override (or run with no
backend) via `frontend/.env`:

```bash
# frontend/.env
VITE_API_URL=                         # empty → static snapshot + client-side fallback
# VITE_API_URL=http://localhost:8099  # the default if unset
```

### 3. (Optional) Live codify.cafe AI

Only if you want the real P2X path instead of the local plan — add to `backend/.env`:

```bash
SCOUTEATS_CODIFY_TOKEN=<P2X Sanctum machine token>
SCOUTEATS_CODIFY_BASE_URL=http://localhost:8000   # or https://api.codify.inc
SCOUTEATS_CODIFY_X_DOMAIN=codify.cafe
# SCOUTEATS_CODIFY_SUBPROJECT_ID=123              # set to skip the resolve-subproject call
```

Without these it degrades gracefully — nothing breaks. See `backend/.env.example` for all
`SCOUTEATS_*` vars.

> **First run:** the DB starts empty, so enrichment runs off the submitted card plus live
> Socrata hydration (needs internet; if Socrata is blocked it sets `hydrated: false` and
> still returns steps). The card grid is fed by `/listings`, which merges the bundled
> `inspections.json` snapshot, so restaurants appear immediately with no ingest step.

---

## Other commands

```bash
# Backend CLI (from backend/, venv active)
python -m scripts.ingest_closed ingest --limit 500     # bulk closed-by-DOHMH ingest
python -m scripts.ingest_closed search "NANCY'S RESTAURANT"
python -m scripts.ingest_closed rehydrate 1            # parallel full-history refresh

# Frontend (from frontend/)
npm run build      # production build
npm run lint       # eslint
npm run format     # prettier

# Regenerate the static snapshot from the CSV (from repo root)
python3 build_listings.py     # writes frontend/public/data/inspections.json
```

There is no automated test suite in this repo.

---

## How it fits together

```
Socrata 43nn-pn8j ─┐
local CSV snapshot ─┼─► GET /listings ─► card grid
MANHATTAN_CLOSED ──┘

click a card ─► POST /analysis/enrich ─► { card, risk_assessment, takeover, enrichment }
                       │                          └─► modal: risk + steps + codify.cafe link
                       ├─ resolve by CAMIS → hydrate from Socrata when thin
                       ├─ rebuild authoritative summary from the DB
                       └─ call codify.cafe (P2X) for AI risk + steps, else local fallback
```

Architecture details for both halves live in `CLAUDE.md`.

Data source: NYC DOHMH via NYC OpenData (`43nn-pn8j`).
