# SWMS Backend -- Drop 1: Geographical Maps & Data Integration

Backend implementation of the Smart Waste Management Simulator's first
drop: authentication/RBAC, the seven habitation parameter categories,
GIS map-layer ingestion, and the upload validation/normalization
pipeline. This project implements the accompanying design document
(`final_backend_design_document.docx`) section by section -- see the
comments at the top of each file for the exact section it satisfies.

## Stack

Node.js 20 · Express · TypeScript · PostgreSQL 15 + PostGIS · Redis +
BullMQ (with an automatic in-process fallback if Redis isn't running)
· Zod · bcrypt · JWT.

## 1. Prerequisites

- Node.js >= 20
- Docker (for local Postgres+PostGIS and Redis), **or** your own
  PostgreSQL 15+ instance with the `postgis` and `pgcrypto` extensions
  available, and optionally your own Redis instance.

## 2. Setup

```bash
# 1. Install dependencies
npm install

# 2. Copy environment variables and adjust if needed
cp .env.example .env

# 3. Start Postgres+PostGIS and Redis locally
docker compose up -d

# 4. Run migrations (creates all tables, enums, indexes)
npm run migrate

# 5. Seed a starting admin account + one sample habitation
npm run seed
#   -> prints the seeded admin email/password (also in .env.example
#      as SEED_ADMIN_EMAIL / SEED_ADMIN_PASSWORD)

# 6. Start the API
npm run dev
#   -> listening on http://localhost:4000

# 7. (Optional, separate terminal) start the background worker
#    Only needed if USE_REDIS_QUEUE=true in .env -- otherwise the API
#    process already handles jobs inline (see src/queue/queue.ts).
npm run worker:dev
```

No Redis available? Set `USE_REDIS_QUEUE=false` in `.env` -- GIS
normalization and upload validation jobs then run inline in the API
process instead of via BullMQ. The API contract (`POST .../layers`
returns `{ batchId, status: "processing" }` either way) is identical,
so nothing else changes.

## 3. Running on a phone / other device on your network

The API binds to all interfaces by default. To reach it from a phone
on the same Wi-Fi network, use your machine's LAN IP instead of
`localhost` (e.g. `http://192.168.1.23:4000`) and set `CORS_ORIGIN` in
`.env` accordingly if a frontend is calling it from a different origin.

## 4. Testing

```bash
npm test
```

Tests are integration tests against a real database (they create real
users/habitations), so Postgres must be running and migrated first
(steps 3-4 above). Covers, among others, the design document's Section
7 test matrix: optimistic-locking conflicts (409), RBAC (researcher
read-only, planner ownership), unknown-category rejection (400), and a
self-intersecting-polygon GIS upload resolving to a `failed` batch
status.

## 5. API quick reference

All responses use one envelope:
`{ "success": true, "data": ... }` or
`{ "success": false, "error": { "code", "message", "details?" } }`.

```bash
# Register (planner/researcher only -- admins are seeded/created by an admin)
curl -X POST localhost:4000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Asha Rao","email":"asha@example.com","password":"SecurePass123!","role":"planner"}'

# Login
curl -X POST localhost:4000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"asha@example.com","password":"SecurePass123!"}'
# -> { data: { accessToken, refreshToken, user } }

TOKEN="<accessToken from above>"

# Create a habitation
curl -X POST localhost:4000/api/habitations \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"Shirva Village","type":"village","latitude":13.36,"longitude":74.74}'

# Write demography data (first write = create, version 1)
curl -X PUT localhost:4000/api/parameters/<habitationId>/demography \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"population":8200,"populationDensityPerSqKm":640,"growthRatePct":1.8}'

# Update it again, with optimistic-locking version check
curl -X PUT localhost:4000/api/parameters/<habitationId>/demography \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"population":8300,"populationDensityPerSqKm":645,"expectedVersion":1}'

# Upload a GIS layer (GeoJSON or zipped Shapefile)
curl -X POST localhost:4000/api/habitations/<habitationId>/layers \
  -H "Authorization: Bearer $TOKEN" \
  -F "layerType=road" -F "file=@roads.geojson"

# Upload a parameter spreadsheet (CSV) for bulk ingestion
curl -X POST localhost:4000/api/uploads/<habitationId>/batches \
  -H "Authorization: Bearer $TOKEN" \
  -F "category=demography" -F "file=@demography_rows.csv"

# Data-quality summary (Section 13.2 innovation)
curl localhost:4000/api/habitations/<habitationId>/data-quality-summary \
  -H "Authorization: Bearer $TOKEN"

# Validation dashboard (Section 13.4 innovation)
curl localhost:4000/api/habitations/<habitationId>/validation-dashboard \
  -H "Authorization: Bearer $TOKEN"

# Freeze a reproducible scenario snapshot (Section 13.3 innovation)
curl -X POST localhost:4000/api/habitations/<habitationId>/snapshots \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"label":"Pre-monsoon 2026 baseline"}'
```

## 6. Project layout

```
db/
  migrations/        Plain numbered .sql files, applied in order
  seed/              Seed script (admin user + sample habitation)
  migrate.ts         Minimal migration runner (up/down)
src/
  config/            env, db pool, redis connection, logger
  middleware/         auth, rbac, validate, requestId, errorHandler
  utils/             apiResponse envelope, AppError, jwt, password,
                     checksum (SHA-256), storage (local disk)
  modules/
    auth/            register/login/refresh
    users/           admin user management
    habitations/     CRUD + ownership + soft delete
    parameters/      generic CRUD for all 7 categories (one registry,
                     one service -- Section 5's "identical business
                     rules across all 7 categories")
    mapLayers/       GIS upload (GeoJSON/Shapefile), parsing/
                     validation, bbox/layer-type queries
    uploads/         spreadsheet (CSV) ingestion, validation,
                     normalization, statistical-anomaly flagging
    dataQuality/     Section 13.2 data-quality-summary endpoint
    snapshots/       Section 13.3 reproducible scenario snapshots
  queue/             BullMQ + in-process fallback, job handlers
  app.ts / server.ts / worker.ts
tests/               vitest + supertest integration tests
```

## 7. What is NOT in this drop (by design)

Per the design document's Section 9 and Section 13.6: no 20-year
simulation logic, no disaster/calamity simulation, no conversational
interface backend, and no frontend. Two schema columns
(`flood_risk_zone`, `is_calamity_baseline`) are reserved for a future
drop but are not populated or read by any endpoint here.
