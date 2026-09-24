# CivicLink

Unified Smart Municipal & Citizen Governance Platform.

CivicLink is a production-oriented municipal operating system: React frontend, FastAPI backend, PostgreSQL, JWT auth, RBAC with ward/department/project scopes, complaint workflow with GPS and camera evidence, and pluggable adapters for maps, storage, email, SMS, OTP, AI verification, notifications, and municipal/emergency data.

Demo seed data is **fictional** and labeled `DEMO DATA`. It is not official government information.

---

## 1. Frontend structure

```
frontend/
  index.html
  src/
    main.tsx
    App.tsx                 # routes + guards
    index.css
    lib/api.ts              # JWT access + refresh
    lib/auth.tsx            # session, can(module, action)
    lib/nav.ts              # role-adaptive navigation
    components/
      CivicScene.tsx        # R3F network / reduced-motion fallback
      Layouts.tsx           # public + authenticated shells
      Guards.tsx            # RequireAuth / GuestOnly
    pages/
      LandingPage.tsx
      AuthPages.tsx
      Complaints.tsx        # GPS, camera, timeline
      CivicPages.tsx        # reels, emergency, directory, feedback
      AdminPages.tsx        # users, permission matrix, audit, map
```

## 2. Backend structure

```
backend/
  app/
    main.py                 # FastAPI, CORS, /docs, startup seed
    core/                   # config, enums, JWT, Argon2, errors
    db/                     # SQLAlchemy engine/session
    models/entities.py      # normalized schema
    schemas/common.py
    api/deps.py             # current user + AccessContext
    api/v1/                 # auth, complaints, civic, admin
    services/               # auth, rbac, scope filters, audit
    providers/              # storage, comms, verification, integrations
    seed.py                 # DEMO users and labeled demo records
  alembic/
```

## 3. Database schema (core tables)

`users`, `roles`, `permissions`, `role_permissions`, `user_roles`, `refresh_tokens`, `wards`, `departments`, `complaints`, `complaint_evidence`, `complaint_status_history`, `projects`, `project_milestones`, `project_updates`, `announcements`, `development_reels`, `reel_likes`, `reel_comments`, `reel_saves`, `feedback`, `cleaning_schedules`, `emergency_services`, `municipal_officials`, `notifications`, `audit_logs`

UUIDs primary keys, FKs, and indexes on ward/status/email.

## 4. API endpoint list

| Method | Path | Notes |
|---|---|---|
| GET | `/health` | DB check |
| GET | `/docs` | OpenAPI |
| GET | `/api/v1/system/public-config` | maps provider (no secrets) |
| POST | `/api/v1/auth/register` | always CITIZEN |
| POST | `/api/v1/auth/login` | JWT access + refresh |
| POST | `/api/v1/auth/refresh` | rotate refresh |
| POST | `/api/v1/auth/logout` | revoke refresh |
| GET | `/api/v1/auth/me` | profile + permission matrix |
| GET/POST | `/api/v1/complaints` | scoped list / create |
| GET | `/api/v1/complaints/{id}` | timeline + evidence |
| PATCH | `/api/v1/complaints/{id}/status` | workflow |
| POST | `/api/v1/complaints/{id}/evidence` | upload + verification |
| POST | `/api/v1/complaints/evidence/{id}/review` | officer/admin |
| GET | `/api/v1/wards` | public wards |
| GET/POST/PATCH | `/api/v1/projects` | scoped |
| GET/POST/PATCH/DELETE | `/api/v1/announcements` | ward scoped |
| GET/POST/DELETE | `/api/v1/reels` | + like/comment/save/view |
| GET/POST | `/api/v1/feedback` | + respond |
| GET/POST | `/api/v1/cleaning` | schedules |
| GET | `/api/v1/emergency` | `liveDataUnavailable` when no provider |
| GET | `/api/v1/municipal-officials` | demo labeled |
| GET | `/api/v1/analytics` | scoped aggregates |
| GET | `/api/v1/search` | permission-filtered |
| GET | `/api/v1/notifications` | in-app |
| GET | `/api/v1/map` | markers |
| GET/POST/PATCH | `/api/v1/users` | admin |
| GET/PUT | `/api/v1/roles`, `/roles/permissions` | admin |
| GET | `/api/v1/audit-logs` | admin |
| GET | `/api/v1/departments` | |

Unauthenticated protected calls return **401**. Authorized but out-of-scope or wrong role return **403**. Validation returns **422**.

## 5. Environment variables

See `.env.example` at repo root (copy to `backend/.env`). Required for a real deploy:

`DATABASE_URL`, `JWT_SECRET`, `JWT_REFRESH_SECRET`, `SECRET_KEY`, `FRONTEND_ORIGIN`

Optional integrations: `MAPS_API_KEY`, `STORAGE_*`, `EMAIL_API_KEY`, `SMS_API_KEY`, `OTP_API_KEY`, `AI_VERIFICATION_*`, emergency/municipal endpoints.

Frontend only: `VITE_API_URL`. Do not put storage/JWT/email secrets in the frontend.

## 6. Authentication flow

1. Register → hashed password (Argon2) → role **CITIZEN** only.
2. Login → verify hash → access JWT (short TTL) + refresh JWT stored hashed with `jti`.
3. API `Authorization: Bearer <access>`.
4. On 401, frontend calls `/auth/refresh`; failed refresh clears session.
5. Logout revokes refresh row.
6. Frontend route guards **and** backend `get_current_user` + RBAC. Direct URL to `/app/admin` as a citizen redirects to their dashboard; API still returns 403.

## 7. RBAC permission matrix (seed defaults)

Actions: VIEW CREATE EDIT DELETE UPDATE ASSIGN APPROVE PUBLISH MANAGE  
Scopes: OWN | ASSIGNED_WARD | ASSIGNED_DEPARTMENT | ASSIGNED_PROJECT | ALL

| | Citizen | Nagar Sevak | Officer | Engineer | Contractor | Admin |
|---|---|---|---|---|---|---|
| Complaint | VIEW/CREATE/EDIT own | Manage assigned ward | Manage assigned dept/ward | VIEW assigned project related | — | ALL |
| Project | VIEW public | CRUD assigned ward | VIEW/EDIT dept | EDIT assigned projects | UPDATE assigned | ALL |
| Reels | VIEW (like/comment) | Manage assigned ward | VIEW | VIEW | VIEW | ALL |
| Announcements | VIEW published | CRUD assigned ward | VIEW | — | — | ALL |
| Users/Roles/Audit | — | — | — | — | — | ALL |

Query filters apply `WHERE ward_id = assigned_ward` (Nagar Sevak), `contractor_id` / `engineer_id` (contractor/engineer), `citizen_id` (citizen). Records are **not** fetched globally and hidden in the UI.

## 8. External API integration points

| Adapter | Env | Default |
|---|---|---|
| Maps | `MAPS_PROVIDER`, `MAPS_API_KEY` | osm |
| Storage | `STORAGE_PROVIDER` local \| s3 | local uploads |
| Email | `EMAIL_PROVIDER`, `EMAIL_API_KEY` | console log |
| SMS | `SMS_PROVIDER`, `SMS_API_KEY` | console |
| OTP | `OTP_PROVIDER` | console stub |
| AI verification | `AI_VERIFICATION_*` | local metadata heuristics (never claims genuine with certainty) |
| Notifications | `NOTIFICATION_PROVIDERS` | in-app DB |
| Emergency live | `EMERGENCY_DATA_*` | none → “Live data unavailable” |
| Municipal import | `MUNICIPAL_DATA_*` | none |

## 9. Setup instructions (local)

Prerequisites: Python 3.12+, Node 20+, PostgreSQL 16 (or Docker).

```powershell
cd civiclink
docker compose up db -d

cd backend
python -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
copy .env .env   # already present for local demo
uvicorn app.main:app --reload --port 8000
```

Tables and demo seed are created on API startup (`create_all` + `seed_if_needed`). Alembic: `alembic upgrade head`.

```powershell
cd frontend
npm install
npm run dev
```

Open http://localhost:5173 — API docs at http://localhost:8000/docs

## 10. Production deployment

1. Set `APP_MODE=production`. Do not seed fictional officials as real.
2. Strong unique `JWT_SECRET`, `JWT_REFRESH_SECRET`, `SECRET_KEY`.
3. Managed PostgreSQL; run Alembic migrations.
4. `STORAGE_PROVIDER=s3` with a real bucket; serve media via CDN.
5. Restrict CORS to the real frontend origin.
6. Put the API behind TLS (reverse proxy). Run `gunicorn -k uvicorn.workers.UvicornWorker app.main:app`.
7. Build frontend `npm run build` and host `dist/` on HTTPS; `VITE_API_URL` = public API URL.
8. Domain-restrict any public maps key.
9. Import official directory and emergency numbers from verified municipal sources — never invent them.
10. Disable or clearly isolate demo users.

## 11. DEMO credentials (fictional)

Password for all demo accounts: **`Demo@CivicLink2026`**

| Role | Email |
|---|---|
| Citizen | citizen@demo.civiclink.local |
| Nagar Sevak (Ward 12 only) | nagarsevak@demo.civiclink.local |
| Officer | officer@demo.civiclink.local |
| Engineer | engineer@demo.civiclink.local |
| Contractor | contractor@demo.civiclink.local |
| Admin | admin@demo.civiclink.local |

These identities are labeled `is_demo` and must not be presented as real officials.

## 12. Remaining integrations (need real keys / official data)

- Google Maps or Mapbox (OSM works without a key)
- S3-compatible or Supabase Storage
- Transactional email and SMS/OTP gateways
- Push notifications (FCM/APNs)
- External AI authenticity / reverse-image APIs
- Live ambulance/emergency availability feed
- Official municipal directory import
- Government employee verification / 2FA enrollment

---

Project path: `C:\Users\Khushi Yadav\civiclink`
