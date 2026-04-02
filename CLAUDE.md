# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Platziflix** — plataforma de cursos en video (estilo Netflix), construida como proyecto del curso de Claude Code en Platzi. Tres sub-proyectos independientes que comparten un mismo API REST.

```
Backend/    → FastAPI + PostgreSQL (fuente de verdad)
Frontend/   → Next.js 15 + React 19 (web)
Mobile/
  ├── PlatziFlixAndroid/  → Kotlin + Jetpack Compose
  └── PlatziFlixiOS/      → Swift + SwiftUI
```

---

## System Architecture

```
┌──────────────────────────────────────────────────────┐
│                   PostgreSQL :5432                   │
└────────────────────────┬─────────────────────────────┘
                         │ SQLAlchemy 2.0
┌────────────────────────▼─────────────────────────────┐
│              FastAPI Backend :8000                   │
│    GET /courses   |   GET /courses/{slug}            │
│    Swagger docs → http://localhost:8000/docs         │
└──────────┬────────────────────────┬──────────────────┘
           │ fetch (SSR)            │ URLSession / Retrofit
┌──────────▼───────────┐  ┌────────▼──────────────────┐
│   Next.js 15 :3000   │  │        Mobile             │
│   App Router         │  │  iOS: SwiftUI + MVVM      │
│   Server Components  │  │  Android: Compose + MVI   │
└──────────────────────┘  └───────────────────────────┘
```

---

## Commands

### Backend (`Backend/`)
```bash
make start              # Levanta Docker: PostgreSQL :5432 + API :8000
make stop               # Para contenedores
make logs               # Ver logs en vivo
make migrate            # Ejecuta migraciones Alembic
make create-migration   # Crea nueva migración (interactivo)
make seed               # Inserta datos de prueba
make seed-fresh         # Limpia y re-inserta datos
make help               # Lista todos los comandos
```

El backend corre exclusivamente vía Docker (gestión de dependencias con UV). No se necesita entorno Python local. La API hace hot-reload al guardar.

### Frontend (`Frontend/`)
```bash
pnpm dev          # Dev server con Turbopack (:3000)
pnpm build        # Build de producción
pnpm lint         # ESLint
pnpm test         # Vitest en modo watch
pnpm test --run              # Ejecución única
pnpm test path/to/file       # Un archivo específico
```

### Mobile
- **iOS**: Abrir `Mobile/PlatziFlixiOS/PlatziFlixiOS.xcodeproj` en Xcode
- **Android**: Abrir `Mobile/PlatziFlixAndroid/` en Android Studio

---

## Backend

### Database (local Docker)

| Variable | Valor |
|----------|-------|
| Host | `localhost:5432` |
| Database | `platziflix_db` |
| User | `platziflix_user` |
| Password | `platziflix_password` |

Migraciones en `Backend/app/alembic/versions/`.

### Data model

Todos los modelos heredan de `BaseModel` (`app/models/base.py`): `id`, `created_at`, `updated_at`, `deleted_at`.
**Nunca hacer hard-delete** — siempre usar `deleted_at` para soft delete.

| Modelo | Campos clave | Notas |
|--------|-------------|-------|
| `Course` | `name`, `description`, `thumbnail` (URL), `slug` (unique, indexed) | `slug` es el identificador público |
| `Teacher` | `name`, `email` (unique, indexed) | |
| `Lesson` | `course_id` (FK), `name`, `description`, `slug`, `video_url` | El API los expone como `classes` |
| `CourseTeacher` | `course_id`, `teacher_id` | Join table many-to-many, sin ID propio |

### API endpoints

| Método | Ruta | Descripción |
|--------|------|-------------|
| `GET` | `/` | Welcome message |
| `GET` | `/health` | Status + DB health + `courses_count` |
| `GET` | `/courses` | Lista de cursos (sin relaciones) |
| `GET` | `/courses/{slug}` | Curso completo con `teacher_id[]` y `classes[]` |

**Endpoint faltante:** `GET /classes/{class_id}` — el Frontend ya lo llama desde `app/classes/[class_id]/page.tsx` pero no está implementado en el Backend.

Contratos de API detallados en `Backend/specs/00_contracts.md`.

### Response shapes

```json
// GET /courses → array directo (sin wrapper)
[{ "id": 1, "name": "...", "description": "...", "thumbnail": "...", "slug": "..." }]

// GET /courses/{slug}
{
  "id": 1, "name": "...", "description": "...", "thumbnail": "...", "slug": "...",
  "teacher_id": [1, 2],
  "classes": [{ "id": 1, "name": "...", "description": "...", "slug": "..." }]
}
```

### Estructura interna

```
app/
  main.py              # FastAPI app + registro de rutas
  core/config.py       # Settings (DATABASE_URL, etc.)
  models/              # SQLAlchemy ORM models
  services/            # Lógica de negocio, inyectada con Depends()
  db/base.py           # Engine + SessionLocal + get_db()
  db/seed.py           # 3 teachers, 3 courses, 7 lessons de prueba
  alembic/             # Migraciones
```

### Convenciones Backend
- Type hints en todas las firmas de funciones
- Pydantic para validación de request/response
- `async`/`await` para operaciones I/O
- Early returns / guard clauses en vez de condicionales anidados
- `joinedload()` para eager loading de relaciones

---

## Frontend

### Rutas y data fetching

Todas las páginas son Server Components que hacen `fetch` con `cache: "no-store"`.

```
app/page.tsx                    → GET /courses          → grid de <Course/>
app/course/[slug]/page.tsx      → GET /courses/{slug}   → <CourseDetailComponent/>
app/classes/[class_id]/page.tsx → GET /classes/{id}     → <VideoPlayer/> (endpoint pendiente)
```

**Bug conocido:** `page.tsx` espera `response.data` pero el backend devuelve el array directamente, lo que rompe el listado de cursos.

### TypeScript types (`src/types/index.ts`)

```typescript
Course        { id, title, teacher, duration, thumbnail, slug }
CourseDetail  extends Course + { description, classes: Class[] }
Class         { id, title, description, video, duration, slug }
Progress      { progress, user_id }
Quiz          { id, question, options[] }
FavoriteToggle { course_id }
```

### Componentes

| Componente | Ubicación | Responsabilidad |
|-----------|-----------|-----------------|
| `Course` | `components/Course/` | Tarjeta en el listado (recibe `Omit<Course, "slug">`) |
| `CourseDetailComponent` | `components/CourseDetail/` | Vista de detalle con lista de clases |
| `VideoPlayer` | `components/VideoPlayer/` | Player de video `<video>` nativo |

### Convenciones Frontend
- CSS Modules + SASS; sin Tailwind, sin estilos inline
- Variables SASS globales en `src/styles/vars.scss` (auto-importadas en todos los archivos SCSS vía `next.config.ts`)
- Archivos: `ComponentName.tsx` + `ComponentName.module.scss`
- Tests en carpeta `__test__/` junto al componente
- Vitest + React Testing Library; queries semánticas (`getByRole`, `getByText`)
- Path alias `@/*` → `src/*`

---

## Mobile — iOS

### Arquitectura: Clean Architecture + MVVM

```
Presentation              Domain                  Data
─────────────────         ─────────────────────   ───────────────────────────
CourseListView   ←→       CourseRepository        RemoteCourseRepository
  @StateObject             (protocol)              ↓ NetworkManager (URLSession)
  CourseListVM                                    CourseDTO
    @Published                                     ↓ CourseMapper
    courses: [Course]                             Course (domain model)
    isLoading: Bool
    searchText: String (debounce 300ms, Combine)
```

### Capas

| Capa | Directorio | Contenido |
|------|-----------|-----------|
| Domain | `Domain/Models/` | `Course`, `Teacher`, `Class` — structs con computed props (`isActive`, `displayDescription`, `hasVideo`) |
| Domain | `Domain/Repositories/` | Protocolos (`CourseRepository`) |
| Data | `Data/Entities/` | DTOs con `CodingKeys` para snake_case → camelCase |
| Data | `Data/Mapper/` | `CourseMapper`, `TeacherMapper`, `ClassMapper` |
| Data | `Data/Repositories/` | `RemoteCourseRepository` + `CourseAPIEndpoints` (enum) |
| Services | `Services/` | `NetworkManager` (singleton), `NetworkService` (protocol), `NetworkError`, `APIEndpoint` (protocol) |
| Presentation | `Presentation/ViewModels/` | `CourseListViewModel: ObservableObject` |
| Presentation | `Presentation/Views/` | `CourseListView`, `CourseCardView` |

**API base URL:** `http://localhost:8000`

---

## Mobile — Android

### Arquitectura: MVI + Jetpack Compose

```
CourseListScreen
  ↓ collectAsState()
CourseListViewModel
  _uiState: MutableStateFlow<CourseListUiState>
  handleEvent(LoadCourses | RefreshCourses | ClearError)
  ↓
RemoteCourseRepository (Retrofit + OkHttp + Gson)
  ↓
CourseDTO → CourseMapper → Course
```

- `di/AppModule.kt` — DI manual; tiene flag `USE_MOCK_DATA` para alternar entre API real y datos mock
- `CourseListUiState` — `{ isLoading, courses[], error, isRefreshing }`

**API base URL:** `http://10.0.2.2:8000` (alias de `localhost` en el emulador Android)

---

## Inconsistencias conocidas entre capas

Al agregar o modificar campos en el Backend, hay que actualizar en paralelo los tipos del Frontend y los DTOs/Mappers de ambas apps móviles.

| Backend | Frontend (`src/types/index.ts`) | iOS / Android |
|---------|--------------------------------|---------------|
| `name` | `title` | `name` |
| `video_url` | `video` | `videoUrl` |
| Array directo en `/courses` | Espera `response.data` (bug) | Array directo |
| `classes[].name` | `classes[].title` | `classes[].name` |

---

## Naming conventions

| Contexto | Convención |
|---------|-----------|
| Python (Backend) | `snake_case` |
| TypeScript / JavaScript | `camelCase` |
| Swift / Kotlin | `camelCase` (variables), `PascalCase` (tipos) |
| Rutas de API | `kebab-case` |
| Slugs de cursos | `kebab-case` |
