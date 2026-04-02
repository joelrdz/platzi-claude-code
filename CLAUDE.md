# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Platziflix** — a full-stack course platform (Netflix for courses) built as a Platzi learning project. Three layers: FastAPI backend, Next.js frontend, iOS + Android mobile apps.

## Commands

### Frontend (`Frontend/`)
```bash
npm run dev        # Dev server (Turbopack)
npm run build      # Production build
npm run lint       # ESLint
npm run test       # Vitest (watch mode)
npm run test -- --run           # Single test run
npm run test -- path/to/file    # Run specific test file
```

### Backend (`Backend/`)
```bash
make start              # Start Docker containers (PostgreSQL + API)
make stop               # Stop containers
make logs               # View container logs
make migrate            # Run Alembic migrations
make create-migration   # Create new migration (interactive)
make seed               # Seed sample data
make seed-fresh         # Clear and reseed
make help               # All available commands
```

Backend runs via Docker; no local Python env needed. API hot-reloads on file changes.

## Architecture

### Backend (FastAPI)

Service-based architecture with SQLAlchemy 2.0 ORM and PostgreSQL 15.

- `app/main.py` — FastAPI app, route registration
- `app/models/` — SQLAlchemy ORM models; `base.py` provides `id`, `created_at`, `updated_at`, `deleted_at` (soft delete) on all entities
- `app/services/` — Business logic layer; injected via FastAPI `Depends()`
- `app/db/` — Engine, session factory, seed scripts
- `app/alembic/` — Database migrations

Current endpoints: `GET /courses`, `GET /courses/{slug}`. API contracts in `Backend/specs/00_contracts.md`.

### Frontend (Next.js 15 + React 19)

App Router with React Server Components by default.

- `src/app/` — Routes: `course/[slug]/`, `classes/[class_id]/`
- `src/components/` — Feature components with co-located CSS Modules (`.module.scss`) and tests
- `src/types/index.ts` — All shared TypeScript interfaces (`Course`, `CourseDetail`, `Class`, etc.)
- `src/styles/vars.scss` — SASS variables; auto-imported into all SASS files via `next.config.ts`

Use Client Components (`"use client"`) only when browser APIs or interactivity is needed.

### Mobile (iOS)

Clean Architecture in `Mobile/PlatziFlixiOS/`:
- **Presentation**: SwiftUI Views + ViewModels
- **Domain**: Core models + repository protocols
- **Data**: DTOs, Mappers (DTO ↔ Domain), repository implementations, `NetworkManager`

Open `PlatziFlixiOS.xcodeproj` in Xcode.

## Key Conventions

### Frontend
- CSS Modules + SASS for component styles (no Tailwind, no inline styles)
- Component files: `ComponentName.tsx`, test files in `__test__/` sibling directory
- Path alias `@/*` maps to `src/*`
- Tests: Vitest + React Testing Library; use semantic queries (`getByRole`, `getByText`)

### Backend
- Type hints on all function signatures
- Pydantic models for request/response validation
- `async`/`await` for all I/O-bound operations
- Early returns / guard clauses over nested conditionals
- `deleted_at` for soft deletes — never hard-delete records
