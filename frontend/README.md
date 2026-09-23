[![English](https://img.shields.io/badge/lang-English-4c1?style=flat-square)](./README.md)
[![Español](https://img.shields.io/badge/lang-Espa%C3%B1ol-lightgrey?style=flat-square)](./README.es.md)

# SPA Occupancy — Frontend

![React](https://img.shields.io/badge/React-18.3-61DAFB?style=flat-square&logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-5.6-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-5.4-646CFF?style=flat-square&logo=vite&logoColor=white)
![Render](https://img.shields.io/badge/Deployed%20on-Render-46E3B7?style=flat-square&logo=render&logoColor=white)

Frontend (React + Vite + TypeScript) to showcase the spa occupancy prediction
Machine Learning model. Meant to be deployed on Render as a Static Site.

## Pages

- **Home (`/`)**: explains the app and links to Predict and Retrain.
- **Predict (`/predict`)**:
  - *Single date*: date + time slot (morning/afternoon) → table with the predicted appointments.
  - *Date range*: two stacked bar charts (morning + afternoon) side by side —
    predicted occupancy for the chosen period and the actual occupancy for
    the same period the previous year (if historical data is available).
- **Retrain (`/retrain`)**: download a sample CSV, and submit new data
  (pasted text or a file) validated against that same format before sending
  it to the backend.

## Layout & navigation

Every page shares a common `Layout` (`src/components/Layout.tsx`) with a nav
bar and a floating widget:

- **Nav bar**: links to Home, Predict, Retrain, and an external link to the
  backend's Swagger UI at `/api/docs`.
- **`HealthWidget`**: a floating "status" button visible on every page. It
  calls `GET /api/health` and shows the service status, whether the model is
  loaded, its training date, and the active model version (factory vs.
  retrained). It also includes a button to **restore the original model**
  via `POST /api/retrain/reset`.

## Data format for retraining

CSV with the exact header `fecha_cita,tramo,n_citas`:

```csv
fecha_cita,tramo,n_citas
2026-07-01,manana,4
2026-07-01,tarde,7
```

- `fecha_cita`: `YYYY-MM-DD`
- `tramo`: `manana` or `tarde` (`mañana` is also accepted)
- `n_citas`: integer >= 0 (actual number of appointments for that slot)

This is the same format the backend consumes, so the file is validated in the
browser before it's sent. The sample file is served from
`public/ejemplo_retrain.csv` and can be downloaded from the Retrain page.

## API contract

The endpoints (`/predict/single`, `/predict/range`, `/retrain`) are
documented in [../backend/README.md](../backend/README.md), which is the
source of truth for the contract. The types that model them live in
`src/types.ts`.

## Local development

```bash
npm install
npm run dev
```

| Script | Command | Description |
|---|---|---|
| `npm run dev` | `vite` | Dev server, with the `/api/*` proxy to `:5000` |
| `npm run build` | `tsc --noEmit && vite build` | Type-check, then build to `dist/` |
| `npm run preview` | `vite preview` | Serve the built `dist/` locally |
| `npm run lint` | `tsc --noEmit` | Type-check only — no ESLint installed |

The frontend always calls `/api/*`, and the proxy in `vite.config.ts`
redirects it to the backend at `http://127.0.0.1:5000`. To work with the full
app, start the backend in another terminal:

```bash
cd ../backend
pip install -r requirements.txt
python main.py
```

To work on the frontend **without the backend running**, copy `.env.example`
to `.env` and set `VITE_USE_MOCK=true`: the app then uses mock data
(`src/api/mock.ts`).

## Build & deployment on Render

```bash
npm run build
```

Generates the static site into `dist/`. The `render.yaml` at the repo root
defines this service as a Static Site with `rootDir: frontend`
(`npm ci && npm run build`, publish path `./dist`) and two rewrite rules,
**in this order**: `/api/*` towards the backend, and `/*` towards
`index.html` for React Router's routing. Thanks to the first rule, the
browser only ever sees a single origin, so there's no need for CORS or for
baking the backend URL into the build.
