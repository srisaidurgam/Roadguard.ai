# ROADGUARD AI

A civic road-damage reporting and repair workflow. Citizens submit a road photo and optional GPS pin; the FastAPI service validates the image, creates a tracked report, runs a backend detector, stores annotated evidence, and records status history. Authorities can review, assign, document repair work, and resolve reports.

## Features

- User registration/login with bcrypt password hashes, signed JWTs, protected user/admin routes, and roles (`USER`, `AUTHORITY`, `ADMIN`).
- JPG, JPEG, PNG, and WEBP upload validation with a 10 MB default limit, UUID storage names, GPS capture, optional manual address, and nearby-report duplicate warnings.
- Background AI workflow with a deterministic mock detector by default, confidence/bounding boxes/severity, processed images, and an optional Ultralytics adapter.
- Report dashboard, AI result images, lifecycle timeline, public report tracking, notifications, authority assignment, before/after repair evidence, and PDF export.
- Database-backed overview/analytics, severity and damage charts, and Leaflet report map.
- SQLite for local use; SQLAlchemy supports PostgreSQL through the same models.
- Light/dark theme preference and responsive desktop/mobile layouts.

## Architecture

```text
frontend (React + TypeScript + Vite)
       | REST / JWT
backend (FastAPI + SQLAlchemy) ---- PostgreSQL or SQLite
       | background detector
mock adapter or optional YOLO weights
       | files
uploads/ and processed/
```

AI executes only in the backend. `AI_PROVIDER=mock` works without model weights. The YOLO adapter is loaded lazily when selected, so default installation stays lightweight.

## Requirements

- Node.js 20.19+ (or 22.12+) and npm.
- Python 3.11+ (tested with Python 3.13).
- Optional: Docker Desktop for the PostgreSQL Compose stack.

## Local Setup

From the project root, create a local environment file:

```powershell
Copy-Item .env.example .env
```

Set `JWT_SECRET` to a random secret before exposing the app beyond local development. Relative database/upload paths in `.env` resolve from the project root.

### Backend

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
cd ..
python scripts/seed.py
cd backend
python -m uvicorn app.main:app --reload
```

If PowerShell blocks the environment activation script, use the interpreter directly: `backend\.venv\Scripts\python.exe -m pip install -r backend\requirements.txt`, then `backend\.venv\Scripts\python.exe -m uvicorn app.main:app --reload --app-dir backend` from the root.

- API: http://localhost:8000
- Swagger: http://localhost:8000/docs
- Health: http://localhost:8000/api/health

### Frontend

In another terminal:

```powershell
cd frontend
npm install
npm run dev
```

- App: http://localhost:5173
- Set `VITE_API_URL` in `frontend/.env` if the backend API is hosted elsewhere.

### Demo Accounts

- Authority administrator: `admin@roadguard.ai` / `admin123`
- Citizen: `demo@roadguard.ai` / `demo123`

Seed data is generated only when fewer than 20 reports exist. Running the seed script again refreshes the demo-account passwords and leaves an existing report set intact.

## AI Configuration

Mock AI is the default and produces annotated detections without downloaded weights. To enable a compatible custom Ultralytics model:

1. Put the trained model at `models/road_damage.pt`.
2. Install the optional AI dependencies with `python -m pip install -r backend/requirements-yolo.txt`.
3. Set `AI_PROVIDER=yolo` and `MODEL_PATH=./models/road_damage.pt` in the root `.env`.
4. Restart the backend.

No trained weights are bundled. The included YOLO adapter maps model class names to results; a model trained with the road-damage class labels is required for meaningful production output. OpenCV is included in the optional requirements for compatible preprocessing/extensions; image validation and annotation use Pillow.

Severity is computed from model confidence, estimated damage area, class weighting, and the number of detected regions. A nearby GPS match is flagged for authority review and is never auto-rejected.

## Database

The default is `sqlite:///./database/roadguard.db`. SQLAlchemy creates tables on API startup. To use PostgreSQL, install the included `psycopg[binary]` dependency and set a URL such as:

```text
DATABASE_URL=postgresql+psycopg://roadguard:password@localhost:5432/roadguard
```

## Demo Data

```powershell
python scripts/seed.py
```

The script creates 20 reports with road images, AI detections, varied statuses, location coordinates, timeline entries, assignments, repair records, and notifications. It also creates/refreshes the demo accounts above.

## Tests

```powershell
python -m pytest backend/tests -q -p no:cacheprovider
```

The workflow test covers registration, authentication, image upload and AI output, tracking, role authorization, verification, assignment, before/after repair evidence, resolution, PDF download, duplicate warning, invalid-image rejection, and missing report handling.

## Docker

From the project root:

```powershell
Copy-Item .env.example .env
docker compose up --build
```

Then seed the running database in another terminal:

```powershell
docker compose exec backend python -c "from app.main import SessionLocal; print('Database ready')"
```

The backend image includes the seed script. Run `docker compose exec backend python /app/scripts/seed.py` to add demo accounts and reports. Compose provides frontend, API, and PostgreSQL services. Frontend: http://localhost:5173, API: http://localhost:8000/docs.

## API Outline

- `POST /api/auth/register`, `POST /api/auth/login`, `GET /api/users/me`
- `POST /api/reports`, `GET /api/reports`, `GET /api/reports/{id}`, `GET /api/reports/{id}/status`, `/history`, `/results`, `/pdf`
- `GET /api/reports/track/{report_number}`
- `POST /api/reports/{id}/verify`, `/reject`, `/assign`, `/repair`, `/resolve`
- `GET /api/admin/dashboard`, `/api/admin/reports`, `/api/admin/reports/{id}`, `PATCH /api/admin/reports/{id}/status`
- `GET /api/analytics/summary`, `/damage-types`, `/severity`, `/status`, `/timeline`, `/locations`
- `GET /api/notifications`, `PATCH /api/notifications/{id}/read`

OpenAPI details are available at `/docs`.

## Deployment Notes

Change the JWT secret, use HTTPS, configure exact CORS origins, put uploads behind authenticated/object storage access, and use managed PostgreSQL before production. BackgroundTasks are intentionally suitable for local/small deployments; move AI work to a durable queue for higher volume. The included mock detector is for development/demo validation, not road-authority decisions.

## Known Limitations

- The mock detector generates plausible demo boxes rather than model predictions; a trained YOLO weight file is not included.
- Report map pins are displayed only when a citizen supplies GPS; manual address geocoding is not configured.
- Duplicate detection currently uses recent coordinate proximity. Image similarity and damage-class comparison are not enabled.
- Notifications are persisted and shown in-app; email/SMS delivery is not configured.
- Repair comparison is shown as before/after images, without a draggable comparison slider.
- The local background processing path is synchronous within FastAPI `BackgroundTasks`; it is not a durable worker queue.
