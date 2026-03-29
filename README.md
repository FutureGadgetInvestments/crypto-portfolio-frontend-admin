# crypto-portfolio-frontend-admin

Admin UI for a personal crypto portfolio tracker. Manages positions and transactions, and surfaces XIRR-based return metrics for active and closed plays.

## Architecture

```
Browser (authenticated)
  │
  ├── Read (static JSON)
  │     └── crypto-portfolio-data (GitHub Raw)
  │           └── GCS fallback
  │
  └── Write (positions, transactions)
        └── Backend API (Cloud Run)
              ├── Firebase token validated
              ├── Mutation applied to BigQuery
              ├── XIRR recomputed
              └── Updated JSON published to GitHub + GCS
```

**Reads** are served from static JSON published by the backend after each mutation.
**Writes** go to the backend API, which validates the Firebase token, updates BigQuery, recomputes XIRR, and republishes the static data files.

## Sibling Repos

This is one of four repos that work together:

| Repo | Purpose |
|------|---------|
| `crypto-portfolio-frontend-admin` | This repo — admin CRUD UI |
| `crypto-portfolio-backend` | REST API + scheduled sync jobs |
| `crypto-portfolio-frontend-public` | Public read-only portfolio view |
| `crypto-portfolio-data` | Static JSON files (positions, transactions, XIRR summary) |

## Tech Stack

- **[Hugo](https://gohugo.io/)** — static site generator
- **Bootstrap 5** — UI framework
- **Firebase Auth** — Google sign-in and ID token issuance
- **GitHub Pages** — hosting (deployed via GitHub Actions)
- **GitHub Raw / GCS** — static data sources for reads

## Local Development

1. Copy `.env.example` to `.env` and fill in Firebase config and backend URL.
2. Start the dev server:

```bash
set -a && source .env && set +a && hugo server
```

3. Open [http://localhost:1313](http://localhost:1313) and sign in.

## Configuration

All config is supplied via `HUGO_PARAMS_*` environment variables. See `.env.example` for the full list.

### GitHub Actions Variables (non-sensitive)

| Variable | Purpose |
|----------|---------|
| `HUGO_PARAMS_FIREBASE_AUTH_DOMAIN` | Firebase auth domain |
| `HUGO_PARAMS_FIREBASE_PROJECT_ID` | Firebase project ID |
| `HUGO_PARAMS_FIREBASE_STORAGE_BUCKET` | Firebase storage bucket |
| `HUGO_PARAMS_BACKENDURL` | Backend API base URL |
| `HUGO_PARAMS_ALLOWED_EMAILS` | Comma-separated list of admin emails |
| `HUGO_PARAMS_GCS_DATA_BUCKET` | GCS bucket for static data fallback |
| `HUGO_PARAMS_GITHUB_DATA_REPO` | GitHub repo for static data (e.g. `FutureGadgetInvestments/crypto-portfolio-data`) |

### GitHub Actions Secrets (sensitive)

| Secret | Purpose |
|--------|---------|
| `HUGO_PARAMS_FIREBASE_API_KEY` | Firebase API key |
| `HUGO_PARAMS_FIREBASE_APP_ID` | Firebase app ID |
| `HUGO_PARAMS_FIREBASE_MESSAGING_SENDER_ID` | Firebase messaging sender ID |

## Adding a New Section

1. Create the content directory:
   ```bash
   mkdir -p content/my-section
   echo $'---\ntitle: "My Section"\n---' > content/my-section/_index.md
   ```
2. Add a nav link in `themes/admin/layouts/partials/navbar.html`.
3. Optionally create a custom layout at `themes/admin/layouts/my-section/list.html`.
4. Update the `RESOURCE_PATH` constant in the page scripts to match your backend endpoint.
