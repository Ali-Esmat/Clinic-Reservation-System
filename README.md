# Clinic Reservation System

A three-tier web application where **doctors publish appointment slots** and **patients book, move and cancel appointments** against them. Built for the *Software Tools 3* course.

| Tier | Tech |
|------|------|
| Frontend | React 18 (Create React App), React-Bootstrap, Axios |
| Backend | Python 3.12, Django 4.2, Django REST Framework (session auth) |
| Database | PostgreSQL 15+ |
| Packaging | Docker (one image per tier), OpenShift manifests for Postgres |

---

## Features

**Accounts**
- Register as a `doctor` or a `patient`; log in / log out with Django session cookies.

**Doctors**
- Create, list, update and delete their own time slots.
- A doctor cannot create two slots within 59 minutes of each other (collision check).

**Patients**
- List doctors and see a doctor's *available* slots.
- Book a slot, move an appointment to another slot, cancel an appointment.
- A patient cannot hold two appointments whose start times fall within 59 minutes of each other.
- A slot is flipped to `RESERVED` on booking and back to `AVALIABLE` on cancellation.

---

## Architecture

```
React (port 3000)  ──HTTP/JSON + session cookie──▶  Django API (port 8000)  ──▶  PostgreSQL (5432)
```

The backend follows a strict layered design:

```
views/     HTTP in/out, JSON parsing                (ClincApp/views)
services/  business rules (collisions, status)      (ClincApp/services)
repo/      ORM queries + serializers                (ClincApp/repo)
models.py  table definitions - never exposed above the repo layer
```

### Data model

```
Users            user_id PK | user_name (unique) | password (hashed) | user_type ('doctor' | 'patient')
DoctorSlots      slot_id PK | date | start_hour | doctor FK→Users | status ('AVALIABLE' | 'RESERVED')
Appointments     appointment_id PK | slot_id 1:1→DoctorSlots | patient_id FK→Users
```
All foreign keys cascade on delete. (Note: the status value is spelled `AVALIABLE` throughout the code base - keep it consistent.)

---

## API reference

All routes are prefixed with `/api/`.

| Method | Path | Purpose |
|--------|------|---------|
| POST | `register` | Create account `{user_name, user_type, password}` |
| POST | `login` | Log in `{user_name, password}`; returns the user |
| POST | `logout` | Log out |
| GET | `user` | Current logged-in user |
| GET / POST | `user/` | List / create users |
| GET / PUT / DELETE | `user/<user_id>/` | Read / update / delete a user |
| GET | `doctor/` | All doctors as `[id, name]` pairs |
| GET | `patient/<user_name>/` | Look up a user by name |
| GET / POST | `doctor_slots/` | All slots / create a slot |
| GET | `doctor_slots/<doctor_id>/` | All slots of one doctor |
| PUT / DELETE | `doctor_slots/<slot_id>/` | Update / delete a slot |
| GET | `doctor_slots/user_view/<doctor_id>/` | Only *available* slots of a doctor |
| GET / POST | `appointments/` | All appointments / book `{slot_id, patient_id}` |
| GET | `appointments/<patient_id>/` | A patient's appointments |
| PUT | `appointments/<appointment_id>/` | Move to another slot `{slot_id, patient_id}` |
| DELETE | `appointments/<appointment_id>/` | Cancel |

Dates are `YYYY-MM-DD`, times are `HH:MM:SS`.
`Postman Collections/Postman_Collection.json` contains ready-made requests.

---

## Repository layout

```
Backend/                Django project (API/ settings, ClincApp/ app, Dockerfile, .env)
frontend/               React app (src/, public/, Dockerfile, .env)
Dockerfile              PostgreSQL image (DB "Clinc")
script.sh               One-shot: build + run all three containers
Run_Scripts/            Individual build/run helpers
Env_files/              Alternative .env templates
Extras/                 Design notes: tables, API flows, architecture
Postman Collections/    API requests
postgres-deployment*.yaml   OpenShift manifests for PostgreSQL
```

---

## Running locally

### Prerequisites
Python 3.12, Node 18+ (LTS), PostgreSQL, and optionally Docker.

### Option A - Docker (recommended)

1. Make the frontend talk to the backend **as seen from your browser**.
   Edit `frontend/.env`:
   ```
   REACT_APP_BE_URL="http://localhost:8000"
   PORT=3000
   ```
2. From the repository root:
   ```bash
   bash script.sh
   ```
   This builds three images, creates the `mynetwork` Docker network, starts Postgres, waits 5 s, then starts the backend (runs migrations automatically) and the frontend.
3. Open <http://localhost:3000>.
4. If the backend exited because Postgres was not ready yet: `docker restart backend`.

Stop and clean up:
```bash
docker rm -f frontend backend database
docker network rm mynetwork
```

### Option B - Without Docker

```bash
# 1. Database (credentials are currently hard-coded in Backend/API/settings.py)
psql -U postgres -c "ALTER USER postgres PASSWORD 'database';"
psql -U postgres -c 'CREATE DATABASE "Clinc";'

# 2. Backend
cd Backend
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirement.txt
export DB_HOST=localhost PGPORT=5432 INTERNAL_PORT=8000
python manage.py migrate
python manage.py runserver 0.0.0.0:8000

# 3. Frontend (new terminal)
cd frontend
npm ci
REACT_APP_BE_URL=http://localhost:8000 PORT=3000 npm start
```

### Quick smoke test (curl)

```bash
API=http://localhost:8000/api
curl -X POST $API/register -H 'Content-Type: application/json' \
  -d '{"user_name":"dr_ali","user_type":"doctor","password":"Pass1234!"}'
curl -X POST $API/register -H 'Content-Type: application/json' \
  -d '{"user_name":"pat_sara","user_type":"patient","password":"Pass1234!"}'
curl -X POST $API/doctor_slots/ -H 'Content-Type: application/json' \
  -d '{"date":"2026-12-01","start_hour":"10:00:00","doctor":1,"status":"AVALIABLE"}'
curl -X POST $API/appointments/ -H 'Content-Type: application/json' \
  -d '{"slot_id":1,"patient_id":2}'
```

---

## Configuration

| Variable | Used by | Meaning | Default |
|----------|---------|---------|---------|
| `DB_HOST` | backend | Postgres hostname | `database` |
| `PGPORT` | backend | Postgres port | `5432` |
| `INTERNAL_PORT` | backend container | Port `runserver` binds to | `8000` |
| `REACT_APP_BE_URL` | frontend | Backend base URL **as reachable from the browser** | `http://backend` |
| `PORT` | frontend | Dev-server port | `3000` |

Database name, user and password (`Clinc` / `postgres` / `database`) are hard-coded in `Backend/API/settings.py` and the root `Dockerfile`.
`REACT_APP_*` variables are baked in when the frontend starts/builds - restart it after changing them.

---

## Deployment (OpenShift / Kubernetes)

The project was deployed on the OpenShift Developer Sandbox (namespace `ali-esmat-dev`).

1. **Database** - create a PostgreSQL deployment (`postgres-deployment.yaml`, or the OpenShift catalog PostgreSQL template) with database `Clinc`, user `postgres`, password `database` (or change `settings.py` to match), and a Service on port 5432.
   - In `postgres-deployment.yaml` rename the env var `POSTGRES_DATABASE` → `POSTGRES_DB` (the official image ignores the former).
   - `postgres-deployment-2.yaml` is an *exported snapshot* of a running pod (RHEL PostgreSQL 15, secret `postgresql-1`); use it as a reference, not as an apply-able manifest.
2. **Backend** - build from `Backend/Dockerfile` (e.g. *Import from Git → Dockerfile → context dir `Backend`*). Set env `DB_HOST=<postgres service name>`, `PGPORT=5432`, `INTERNAL_PORT=8000`. Expose a Service + Route on 8000. Add the route host to `ALLOWED_HOSTS` and the frontend origin to `CSRF_TRUSTED_ORIGINS`.
3. **Frontend** - build from `frontend/Dockerfile` with `REACT_APP_BE_URL=https://<backend-route-host>` and `PORT=3000`; expose a Route.
4. Because the two routes are different origins, session cookies need `SESSION_COOKIE_SAMESITE="None"`, `SESSION_COOKIE_SECURE=True` (same for the CSRF cookie) in `settings.py`.
