# crypto-portfolio-frontend-admin

## Project Overview

This is the **admin frontend** for a personal crypto portfolio tracker — a Hugo-based site with Firebase auth that lets you log positions and transactions, and computes XIRR to measure true time-weighted returns across active and closed plays.

This is one of four sibling repos that work together. All four should be cloned as siblings under the same parent directory.

## Multi-Repo Setup

```
FutureGadgetInvestments/
├── crypto-portfolio-frontend-admin/   ← Admin UI — Hugo, Firebase auth (this repo)
├── crypto-portfolio-backend/          ← REST API + scheduled Cloud Run jobs
├── crypto-portfolio-frontend-public/  ← Public read-only portfolio view, no auth
└── crypto-portfolio-data/             ← Static JSON files committed by the backend
```

## All Repositories

| Repo | GitHub URL | Local Path | Purpose |
|------|-----------|------------|---------|
| `crypto-portfolio-frontend-admin` | `https://github.com/FutureGadgetInvestments/crypto-portfolio-frontend-admin` | `.` | Admin CRUD UI — manage positions and transactions |
| `crypto-portfolio-backend` | `https://github.com/FutureGadgetInvestments/crypto-portfolio-backend` | `../crypto-portfolio-backend` | REST API + scheduled data sync jobs |
| `crypto-portfolio-frontend-public` | `https://github.com/FutureGadgetInvestments/crypto-portfolio-frontend-public` | `../crypto-portfolio-frontend-public` | Public read-only portfolio view |
| `crypto-portfolio-data` | `https://github.com/FutureGadgetInvestments/crypto-portfolio-data` | `../crypto-portfolio-data` | Static JSON files consumed by both frontends |

## GCP Infrastructure

| Resource | Name | Details |
|----------|------|---------|
| GCP project | `your-gcp-project` | Hosts all GCP resources |
| Cloud Run service | `crypto-portfolio-api`, `us-central1` | REST API |
| Cloud Run job | `crypto-portfolio-sync`, `us-central1` | Scheduled data sync |
| GCS bucket | `crypto-portfolio-data-bucket` | Exported JSON snapshots |
| BigQuery | dataset: `crypto_portfolio` | Source of truth for positions + transactions |
| Firebase project | `your-firebase-project` | Auth provider |

## Domain Model

### Positions
A **position** is a crypto asset you have or had money in (e.g., BTC, ETH, a specific altcoin play).

| Field | Description |
|-------|-------------|
| `id` | Unique identifier |
| `asset` | Ticker or name (e.g., `BTC`, `ETH`, `SOL`) |
| `label` | Optional human label (e.g., "BTC long-term hold") |
| `status` | `active` or `closed` |
| `current_value_usd` | Current market value of holdings in USD (updated by sync job) |
| `notes` | Free-text notes on the thesis or outcome |

### Transactions
A **transaction** is a cash flow event on a position — either putting money in or taking money out.

| Field | Description |
|-------|-------------|
| `id` | Unique identifier |
| `position_id` | FK to position |
| `date` | Date of the cash flow (YYYY-MM-DD) |
| `amount_usd` | Amount in USD — **negative = cash out (investment/buy), positive = cash in (proceeds/sell)** |
| `type` | `deposit` (buying in) or `withdrawal` (taking profits or exiting) |
| `notes` | Optional description |

### XIRR
XIRR (Extended IRR) is computed per-position and across all positions by the backend. It accounts for the timing and size of each cash flow to give a true annualized return rate.

- **Per-position XIRR**: all transactions for that position + current market value as final cash flow
- **Portfolio XIRR**: all transactions across all positions + sum of current market values as final cash flow
- The backend computes and stores XIRR in BigQuery; the sync job republishes it to the data repo

## Architecture

### This Repo — Admin Frontend
- **Framework:** [Hugo](https://gohugo.io/) — static site generator with Go templates
- **Theme:** Custom theme (`themes/admin/`) — Bootstrap 5
- **Auth:** Firebase Authentication — users must sign in before any write request
- **Data sources (reads):**
  1. JSON files from `crypto-portfolio-data` GitHub repo (primary)
  2. JSON files from GCS bucket (fallback)
  3. Live API calls to the backend (for writes and immediate post-write reads)
- **Write flow:** `api()` helper in `static/js/api.js` attaches Firebase ID token as `Authorization: Bearer <token>` on all backend requests

### Backend
- **API microservice** (Cloud Run service): validates Firebase tokens, CRUD on BigQuery, computes XIRR, publishes updated JSON to GCS and the data repo
- **Scheduled job** (Cloud Run Job): daily sync to update `current_value_usd` from market prices and recompute XIRR

### Public Frontend
- Read-only view of portfolio performance
- No Firebase auth required
- Reads from `crypto-portfolio-data` repo or GCS only — never calls the write API

### Data Repo
- Plain JSON files committed by the backend
- Key files: `positions.json`, `transactions.json`, `portfolio_summary.json`
- Do not manually edit; the backend owns writes here

## Key Files (This Repo)

| Path | Purpose |
|------|---------|
| `hugo.toml` | Hugo config — `params.backendURL` sets the API base |
| `themes/admin/layouts/` | Hugo templates (baseof, list, index) |
| `themes/admin/layouts/partials/` | head, navbar, footer, scripts partials |
| `static/js/firebase-init.js` | Firebase app init + global `authSignOut()` |
| `static/js/api.js` | Authenticated `api(method, path, body)` helper |
| `static/js/app.js` | Global `showToast()` utility |
| `static/js/data-loader.js` | `loadJsonData(filename)` — GitHub-first, GCS-fallback data fetching |
| `static/css/app.css` | Style overrides on Bootstrap 5 |
| `content/positions/_index.md` | Positions section — list and manage crypto positions |
| `content/transactions/_index.md` | Transactions section — log deposits and withdrawals |
| `.env.example` | Template for Firebase + backend env vars |

## Auth Flow

1. User lands on the site and signs in via Firebase Auth (Google or email/password).
2. Firebase issues an ID token.
3. The frontend attaches it as `Authorization: Bearer <token>` on all backend requests via `api()`.
4. The backend validates the token and checks the `ALLOWED_EMAILS` allowlist before processing writes.

**Never hardcode Firebase credentials** — they belong in `.env` (gitignored).

## Development Notes

- Hugo config lives in `hugo.toml`
- Firebase config goes in `.env` — never commit this file (already in `.gitignore`)
- Environment variables are injected as `HUGO_PARAMS_*` and map to `.Site.Params.*` in templates

## Running the Dev Server

```bash
set -a && source .env && set +a && hugo server
```

## Cross-Repo Coordination Rules

1. **New API endpoint:** implement the handler in `crypto-portfolio-backend` AND wire up the `api()` call here.
2. **New data field:** update the BigQuery schema in the backend, the JSON output shape, `crypto-portfolio-data` structure, and both frontends.
3. **Scheduled job changes:** edit the job code in `crypto-portfolio-backend`; it deploys as a separate Cloud Run Job.
4. **Public frontend data change:** if the JSON shape changes, update `crypto-portfolio-frontend-public` to match.
5. **Commit separately** in each affected repo with matching commit messages.
6. **Never hardcode Firebase credentials** — use environment variables only.

## Custom Agents

A `full-stack` sub-agent is defined in `.claude/agents/full-stack.md`. It:
- Checks all four sibling repos and clones any that are missing
- Has full context on every repo's role, data flow, and GCP infrastructure
- Handles tasks that span multiple repos simultaneously
