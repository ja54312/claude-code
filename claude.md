# Big Picture — Arquitectura Platziflix

Proyecto **monorepo multi-plataforma** con 4 sub-proyectos independientes que consumen una única fuente de verdad (API REST). El patrón común en los 3 clientes es **Clean Architecture por capas**: Domain → Data/Services → Presentation.

## Diagrama de sistema

```
┌────────────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│  Frontend (Next.js 15)     │  │ Android (Kotlin)     │  │ iOS (Swift)          │
│  React 19 + SCSS + Vitest  │  │ Compose + MVVM       │  │ SwiftUI + MVVM       │
│  App Router (SSR/SSC)      │  │ Retrofit             │  │ URLSession           │
└──────────────┬─────────────┘  └──────────┬───────────┘  └──────────┬───────────┘
               │                           │                         │
               └───────────────┬───────────┴─────────────────────────┘
                               │ HTTP / JSON (REST)
                               ▼
                ┌──────────────────────────────┐
                │  Backend API — FastAPI       │
                │  :8000  (Docker container)   │
                │  Service Layer + SQLAlchemy  │
                └──────────────┬───────────────┘
                               │
                               ▼
                ┌──────────────────────────────┐
                │  PostgreSQL 15 (Docker)      │
                │  :5432  + Alembic migrations │
                └──────────────────────────────┘
```

## 1. Backend — `Backend/` (FastAPI + PostgreSQL)

Capas bien separadas dentro de `app/`:
- `main.py` — router HTTP, declaración de endpoints, DI con `Depends`.
- `services/course_service.py` — **Service Layer** con toda la lógica de negocio (ratings, agregaciones, validaciones).
- `models/` — SQLAlchemy 2.0 ORM: `course`, `teacher`, `lesson`, `class_`, `course_teacher` (tabla puente M:N), `course_rating`.
- `schemas/rating.py` — Pydantic request/response DTOs.
- `core/config.py` — settings con `pydantic-settings`.
- `db/` — `base.py` (engine + `get_db`), `seed.py`.
- `alembic/` — migraciones versionadas.
- `tests/` + `test_main.py` — pytest + httpx.

**Entregado vía `docker-compose`**: dos servicios (`db`, `api`), volumen de código montado en caliente. Gestión de deps con **UV**. Todo comando se ejecuta vía `make` dentro del contenedor.

**Endpoints (REST, 3 áreas):**
- `courses`: `GET /courses`, `GET /courses/{slug}`, `GET /classes/{class_id}`
- `ratings` (CRUD completo, soft-delete): `POST/GET/PUT/DELETE /courses/{id}/ratings...` + `/stats` + `/user/{uid}`
- `health`: `GET /health` (verifica DB con `SELECT COUNT(*) FROM courses`)

## 2. Frontend — `Frontend/` (Next.js 15)

**App Router** con rutas dinámicas:
- `src/app/page.tsx` — catálogo (home).
- `src/app/course/[slug]/` — detalle del curso (SSR-friendly, SEO por slug).
- `src/app/classes/[class_id]/` — reproductor de clase.
- `src/components/` — `Course`, `CourseDetail`, `StarRating`, `VideoPlayer` (CSS Modules + SCSS).
- `src/services/ratingsApi.ts` — cliente HTTP con `fetch` + timeout/`AbortController`, lee `NEXT_PUBLIC_API_URL` (default `http://localhost:8000`).
- `src/types/` — DTOs tipados espejo de los Pydantic schemas.
- `vitest.config.ts` + `src/test/` — testing con jsdom.

## 3. Mobile — Dos apps nativas con arquitectura paralela

### Android — `Mobile/PlatziFlixAndroid/` (Kotlin + Jetpack Compose)

Paquete `com.espaciotiago.platziflixandroid` estructurado **Clean Architecture**:
- `data/` → `entities` (DTOs red), `mappers`, `repositories` (impl), `network` (Retrofit).
- `domain/` → `models` (entidades puras), `repositories` (interfaces).
- `presentation/courses/` → `screen` (Composables), `viewmodel` (MVVM + StateFlow), `state`, `components`.
- `di/` — inyección de dependencias.
- `ui/theme/` — Material/Compose theme.

### iOS — `Mobile/PlatziFlixiOS/` (Swift + SwiftUI)

Misma arquitectura por capas:
- `Data/` → `Entities`, `Mapper`, `Repositories` (impl).
- `Domain/` → `Models`, `Repositories` (protocolos).
- `Presentation/` → `Views` (SwiftUI), `ViewModels` (`@Observable`/Combine).
- `Services/` — capa de red reutilizable: `NetworkManager`, `NetworkService`, `APIEndpoint`, `HTTPMethod`, `NetworkError`.

## Ideas arquitectónicas transversales

1. **Backend como única fuente de verdad** — los 3 clientes son "delgados", sin lógica de negocio duplicada.
2. **Mismo patrón por capas en los 3 clientes** — Domain/Data/Presentation, con repositorios que abstraen la red. Esto hace que cambios de endpoint se localicen en una sola capa por cliente.
3. **Modelo de datos central**: Course (1:N) Lesson (1:N) Class · Course (N:M) Teacher · Course (1:N) CourseRating (soft-delete via `deleted_at`).
4. **Ratings es un subsistema reciente y completo** — soft-delete, agregación server-side (`average_rating`, `total_ratings`, distribución 1-5), unicidad user+course, ya integrado en `GET /courses`.
5. **Contenerización solo para backend** — frontend y mobile corren nativos en la máquina del dev; cualquier comando backend va dentro del contenedor `api`.
6. **Migraciones Alembic autogeneradas** vía `make create-migration` → `make migrate`, con un seed determinista (`make seed-fresh`).
7. **Testing en todas las capas**: pytest (backend), Vitest + Testing Library (frontend), `androidTest`/`test` (Android), `PlatziFlixiOSTests`/`UITests` (iOS).

## Cómo fluye un caso de uso (ej.: ver detalle de curso)

```
Usuario → UI (Next page / Compose Screen / SwiftUI View)
       → ViewModel / Server Component
       → Repository (mobile) / services/ratingsApi.ts (web)
       → HTTP GET /courses/{slug}
       → FastAPI main.py → CourseService.get_course_by_slug
       → SQLAlchemy (joinedload teachers+lessons+ratings)
       → PostgreSQL
       ← JSON (course + teachers + lessons + rating stats)
       ← Mapper → Domain Model → State → UI
```
