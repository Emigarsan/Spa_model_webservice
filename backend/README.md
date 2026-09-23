[![English](https://img.shields.io/badge/lang-English-4c1?style=flat-square)](./README.md)
[![Español](https://img.shields.io/badge/lang-Espa%C3%B1ol-lightgrey?style=flat-square)](./README.es.md)

# SPA Occupancy — Backend

![Python](https://img.shields.io/badge/Python-3.13-3776AB?style=flat-square&logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-3.1-000000?style=flat-square&logo=flask&logoColor=white)
![gunicorn](https://img.shields.io/badge/gunicorn-26.2-499848?style=flat-square&logo=gunicorn&logoColor=white)
![scikit-learn](https://img.shields.io/badge/scikit--learn-1.8-F7931E?style=flat-square&logo=scikitlearn&logoColor=white)
![Render](https://img.shields.io/badge/Deployed%20on-Render-46E3B7?style=flat-square&logo=render&logoColor=white)

REST API (Flask) serving the spa occupancy prediction model and allowing it to be retrained with new data. Deployed on Render as a Web Service.

* Hand-rolled Swagger UI at `GET /docs` (spec at `GET /openapi.json`) — no extra dependencies.
* Optional token-protected retraining (`RETRAIN_TOKEN`).
* Every retrain candidate is validated against a temporal holdout (MAE) before it's published.

## Structure

```
backend/
├── main.py              single Flask file
├── requirements.txt
├── README.md / README.es.md
├── app/                 domain logic, framework-free
│   ├── model_service.py
│   ├── train_model.py
│   ├── retrain.py
│   ├── utils/
│   └── README_reentrenamiento.md
├── tests/
├── data/                occupancy CSV
└── models/              serialized artifacts
```

| File | Responsibility |
|---|---|
| `main.py` | **The only file that touches Flask**: every route, request-shape validation and the `errorhandler`s. |
| `app/model_service.py` | Model artifact (cached loading) and prediction, single-day and range. Framework-free. |
| `app/train_model.py` | Retraining, dataset validation and reading the history. Framework-free. |
| `app/retrain.py` | Ingestion of the CSV received through `POST /retrain`. Framework-free. |
| `app/utils/` | Feature engineering, shared by training and prediction. |
| `data/` | Occupancy CSV (`fecha_cita,tramo,n_citas`). Doesn't live inside `app/`: it's a resource, not code. |
| `models/` | Serialized artifacts (see below). Also not moved, for the same reason. |

`data/` and `models/` stay siblings of `main.py`, not children of `app/`:
they're resources and artifacts, not application code.

### The two artifacts

```
models/
├── modelo_ocupacion.joblib     factory model, version-controlled, NEVER overwritten
├── scaler.joblib                factory scaler, version-controlled
└── modelo_reentrenado.joblib    published by /retrain, git-ignored
```

If `modelo_reentrenado.joblib` exists, it's the one used; otherwise it falls
back to the factory model. The version-controlled artifact is **read-only**,
so returning to the original never depends on a backup being intact: deleting
one file is enough, which is exactly what `POST /retrain/reset` does.

Routing is deliberately concentrated in `main.py`: the rest of the modules are
framework-free dependencies, so they can be tested and reused without
spinning up the app, and there's only one place to look to know what the API
exposes.

## API contract

This is the source of truth for the frontend integration (`frontend/src/types.ts`).
Time slots always travel **without the `ñ`** (`manana` / `tarde`): the `ñ` is
an internal detail of the model and the dataset. Both spellings are accepted
as input.

### `GET /` — landing

API description and list of endpoints.

### `GET /docs` — interactive documentation

Equivalent to FastAPI's automatic `/docs`: a Swagger UI, served from
`app/docs.py` (`OPENAPI_SPEC` plus a minimal HTML page that loads Swagger UI
from a CDN). The spec itself lives at `GET /openapi.json`. Deliberately no
new dependencies added to `requirements.txt`.

### `GET /health`

Always `200`, even without a model loaded — it's a *liveness check*, and
returning an error would only make Render restart the service in a loop.

```json
{"status": "ok", "model_loaded": true, "entrenado_hasta": "2026-01-24",
 "version_modelo": "original", "es_original": true}
```

`es_original` is `false` when a retrained model is active. The frontend uses
it to offer the restore button in the floating status widget.

### `GET /predict`, `GET /predict/single`

`GET` only, no `POST`: it's a read-only query (it changes nothing on the
server), so `GET` is the only verb that makes sense here — and it's also
what the evaluation criteria require (`requests.get(url, params={...})`, or
pasting the URL into a browser). `/predict` and `/predict/single` are the
same function under two names; the frontend uses `/predict/single`.

```
GET /predict?fecha=2026-09-10&tramo=tarde
```
```json
{"date": "2026-09-10", "tramo": "tarde", "citasPrevistas": 5.7,
 "es_cierre": false, "version_modelo": "original", "entrenado_hasta": "2026-01-24"}
```

Accepts `fecha` or `date` indistinctly as the parameter name. `citasPrevistas`
is the field the frontend consumes; the rest is metadata that doesn't get in
the way but makes clear this is a real prediction from the active model, not
a placeholder value.

Closed days (Dec 25th, Jan 1st, Jan 6th) return `0` without querying the model.

### `GET /predict/range`

Same criteria as `/predict`: `GET` only, with `startDate`/`endDate` in the
query string.

```
GET /predict/range?startDate=2026-09-08&endDate=2026-09-14
```
```json
{
  "current":      [{"date": "2026-09-08", "manana": 2.7, "tarde": 3.6}, ...],
  "previousYear": [{"date": "2025-09-08", "manana": 3.0, "tarde": 5.0}, ...]
}
```

- `current` holds **predictions**; `previousYear` is the **actual occupancy**
  for the same period one year before, read from the historical data in `data/`.
- `previousYear` is `null` if there's no historical data for that period. If
  only part of it exists, only the days that actually exist are returned: it's
  never padded with zeros, because `n_citas = 0` is a genuinely observed value
  and padding would make the chart lie.
- The range can't exceed 366 days.

### `GET /retrain`

Status of the deployed model, detected datasets and usage instructions.

### `POST /retrain`

Accepts the CSV in two ways: as a file in the `file` field
(`multipart/form-data`) or as JSON `{"csvText": "<content>"}`.

```jsonc
// 200 — the model has been replaced
{"status": "ok",    "rowsIngested": 14, "message": "Se han incorporado 14 filas y el modelo..."}
// 200 — the candidate didn't pass validation; the previous model stays active
{"status": "error", "rowsIngested": 14, "message": "Se han recibido 14 filas, pero el modelo NO..."}
```

A rejected candidate deliberately responds with `200` rather than an HTTP
error: the CSV was processed correctly, so the frontend can show how many
rows it read and why the model wasn't published.

### `POST /retrain/reset`

Returns to the factory state: deletes `modelo_reentrenado.joblib`, the
uploaded CSVs (`data/subida_*.csv`) and the backups. Idempotent.

```jsonc
{"status": "ok", "modelRestored": true, "filesRemoved": 1, "message": "..."}
```

Requires `X-Retrain-Token` under the same conditions as `POST /retrain`.

### Errors

All errors come back as JSON, shaped as `{"error": "..."}`:

| Code | When |
|---|---|
| `400` | Malformed data: date, time slot, inverted range, invalid CSV, no new rows. |
| `401` | Missing `X-Retrain-Token` while the server has `RETRAIN_TOKEN` configured. |
| `405` | Method not allowed — `/predict` and `/predict/range` are `GET`-only. |
| `409` | A retrain is already in progress. |
| `413` | The CSV is larger than 2 MB. |
| `503` | The model artifact isn't available. |

## Retraining

The received CSV **is added to the history**, not swapped in as a
replacement: it's saved under `data/` with a name (`subida_<timestamp>.csv`)
that sorts after the base dataset, so that on overlap the most recent data
wins.

The model is then retrained on everything under `data/`, and the new model is
published only if it **doesn't clearly perform worse** than the current one
on a temporal holdout: the threshold is the current model's MAE on that same
window, with a 10% margin (`MARGEN_TOLERANCIA_MAE`) — not a comparison
against a naive baseline or a fixed ceiling, which used to reject valid
retrains with real data just because, by chance, that particular window
favored the reference. That fixed ceiling (`MAE_MAXIMO_ACEPTABLE`) is only
used for the very first training, when there's no previous model to compare
against.

**The CSV is kept even if the model isn't published.** It's real data: a
candidate not beating the current model in this particular validation
doesn't invalidate it as an observation. It's only discarded if it couldn't
even be evaluated (e.g. it doesn't add any row that wasn't already there).

**The only requirement is that there are genuinely new rows** (more than
there were when the current model was trained). Without that, the upload is
rejected: retraining and republishing without any new information would be
pointless. Given that, there are two valid ways to contribute new rows, and
the validation window adapts to which one it is:

- **Extending the horizon** (uploading a new week, for example): the
  validation window (60 days by default) is shortened — never lengthened —
  to the days that are genuinely later than the current `entrenado_hasta`.
  Without this adjustment, uploading data week by week would never get past
  the first time: the cutoff (max date minus 60 days) would fall before the
  currently trained date even though the uploaded week was genuinely new.
- **Filling a historical gap** (dates earlier than the latest one already
  recorded): since there are no new days at the end to isolate as a
  hold-out, the full validation window is used over the already-known final
  stretch, comparing whether adding those rows improves or worsens the model
  there.

In both cases the model is only published if it genuinely improves; filling
a gap isn't guaranteed to do so (if the gap is small relative to the total
history, it's normal for it to barely move the validation MAE).

Publishing means writing `modelo_reentrenado.joblib`; the factory artifact is
never touched, so `POST /retrain/reset` undoes any retrain without needing to
restore backups.

The active artifact is cached in memory and only reloaded when the file
changes, so a retrain takes effect without restarting the service.

Retraining-lane details live in
[app/README_reentrenamiento.md](app/README_reentrenamiento.md) — kept close
to the code, so it may lag behind the higher-level description above.

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `RETRAIN_TOKEN` | No | If set, protects `POST /retrain` and `POST /retrain/reset` via the `X-Retrain-Token` header. Commented out by default in `render.yaml`, since enabling it makes the frontend's Retrain page get `401`s (the browser has nowhere safe to store the token). There's no `.env.example` for the backend — set it as a plain shell env var locally, or uncomment it in `render.yaml` for deployment. |

## Local development

```bash
pip install -r requirements.txt
python main.py            # http://127.0.0.1:5000
python -B -m unittest discover -s tests -v
```

`scikit-learn` is pinned to the version the artifacts in `models/` were
serialized with; see the comment in `requirements.txt`.

## Deployment

The `render.yaml` at the repo root defines this service with
`rootDir: backend`, started with `gunicorn main:app` and `/health` as the
health check.

⚠️ On Render's free plan the disk is ephemeral: the retrained model and any
uploaded CSVs are lost when the service restarts or goes to sleep, and it
falls back to the artifact checked into the repository.
