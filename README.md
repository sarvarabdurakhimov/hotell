# StarHotel — Multi-Tenant Hotel Management SaaS Platform

A production-ready, scalable, multi-tenant Hotel Management System (HMS) built with **Django 5.2 LTS**, **Django REST Framework**, **Next.js 15 (App Router)**, **PostgreSQL 17**, and **Redis/Celery**.

Designed to manage from independent boutique hotels to hotel chains exceeding 500 properties with strict tenant isolation, double-booking prevention, role-based access control, and asynchronous task execution.

---

## 🏗️ Architecture & Technology Stack

| Layer | Technology | Key Capabilities |
| :--- | :--- | :--- |
| **Backend** | Python 3.13 / Django 5.2 LTS | Modular monolith with clean domain apps |
| **API** | Django REST Framework | Versioned REST API (`/api/v1/`), OpenAPI via `drf-spectacular` |
| **Frontend** | Next.js 15 / React 19 / TypeScript / Tailwind | App Router, Server/Client components, TanStack Query |
| **Database** | PostgreSQL 17 + `psycopg 3` | Authoritative persistence, `SELECT FOR UPDATE` locking |
| **Cache & Queue** | Redis 7 + Celery 5.4 | Asynchronous email dispatch, caching, rate limiting |
| **Security** | HttpOnly Cookies + JWT + CSRF | Cross-tenant isolation filters, token blacklisting |
| **Infrastructure** | Docker & Docker Compose | Multi-container composition with health checks |

---

## 📁 Repository Structure

```text
├── backend/                       # Django Modular Monolith
│   ├── apps/
│   │   ├── users/                 # Custom AbstractBaseUser & JWT auth
│   │   ├── organizations/         # Hotel chains / Parent corporate entities
│   │   ├── hotels/                # Core tenant model & property settings
│   │   ├── memberships/           # User-hotel roles (Owner, Manager, etc.)
│   │   ├── rooms/                 # Room types, physical room units
│   │   ├── guests/                # Privacy-compliant guest profiles
│   │   ├── bookings/              # Stay interval reservations, concurrency locks
│   │   ├── payments/              # Idempotent payments & audit trail
│   │   ├── housekeeping/          # Task allocation & cleaning statuses
│   │   ├── notifications/         # Celery background dispatch
│   │   ├── reports/               # Occupancy & revenue analytics
│   │   └── common/                # Permissions, pagination, middleware
│   ├── config/                    # Django base/dev/prod settings, WSGI/ASGI
│   ├── tests/                     # Pytest suite with cross-tenant security tests
│   └── requirements/              # base.txt, development.txt, production.txt
│
├── frontend/                      # Next.js 15 App Router
│   └── src/
│       ├── app/                   # (auth), dashboard, hotels/[hotelId]/*
│       ├── components/            # Layouts, navigation, sidebar, providers
│       ├── lib/api/               # Axios client with CSRF & 401 interceptors
│       └── types/                 # Strict TypeScript entity schemas
│
├── nginx/                         # Production reverse proxy config
├── .github/workflows/             # GitHub Actions CI matrix
├── docker-compose.yml             # Full-stack local development cluster
└── README.md
```

---

## 🔒 Multi-Tenancy & Security Rules

1. **Shared PostgreSQL Schema with Explicit Tenant Foreign Keys**: Every tenant-owned resource (rooms, reservations, guests, bills) links to a validated `hotel_id`.
2. **Derive Tenant Context from Authenticated Request**: The API never trusts raw client-supplied `hotel_id` parameters without checking active membership.
3. **Double-Booking Race Condition Prevention**:
   - Stay interval defined as `[check_in, check_out)` — departure day is unoccupied.
   - Bookings use `Room.objects.select_for_update()` inside `@transaction.atomic` to serialize concurrent requests.
4. **Cookie-Based JWT & CSRF**:
   - Access & refresh tokens stored in secure `HttpOnly` cookies.
   - CSRF token in accessible cookie and passed via `X-CSRFToken` header.
5. **Data Privacy**: Sensitive guest identification numbers and staff notes are strictly excluded from public serialization.

---

## 🚀 Quick Start (Docker Compose)

### 1. Clone & Configure
```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env.local
```

### 2. Launch Services
```bash
docker-compose up --build
```
This spins up:
- **PostgreSQL 17** at port `5432`
- **Redis 7** at port `6379`
- **Django Backend** at port `8000`
- **Celery Worker** connected to Redis
- **Next.js Frontend** at port `3000`

### 3. Apply Migrations & Create Superuser
```bash
docker-compose exec backend python manage.py migrate
docker-compose exec backend python manage.py createsuperuser
```

### 4. Access Platform
- **Frontend App**: `http://localhost:3000`
- **API Documentation (Swagger UI)**: `http://localhost:8000/api/docs/`
- **OpenAPI Schema (JSON)**: `http://localhost:8000/api/schema/`
- **Django Administration**: `http://localhost:8000/admin/`

---

## 🧪 Running Automated Tests

Run backend tests using Pytest inside the container or local virtual environment:

```bash
docker-compose exec backend pytest -v
```

Tests include:
- `test_users.py`: Authentication, JWT issuance, cookie handling.
- `test_hotels.py`: Multi-property creation, automatic ownership assignment.
- `test_cross_tenant.py`: Hardened verification that staff of Hotel A receive `403 Forbidden` when attempting to access Hotel B data.
- `test_bookings.py`: Race condition prevention, overlapping date rejection, and valid status transitions.
