---
name: Ratings feature — design decisions
description: Decisiones técnicas tomadas en el diseño del sistema de ratings de Platziflix
type: project
---

Sistema de ratings para Platziflix — diseño aprobado, pendiente de implementación.

Modelo: tabla `course_ratings` con `(course_id, user_id UUID string, rating int 1-5)`. UniqueConstraint en `(course_id, user_id)` para upsert. Hereda de BaseModel (soft delete pero no se usa para ratings).

Identidad anónima: UUID v4 generado con `crypto.randomUUID()` y persistido en `localStorage` bajo la clave `platziflix_user_id`.

Nuevo endpoint: `POST /courses/{slug}/ratings` — recibe `{ user_id, rating }`, devuelve stats actualizadas.

GET /courses y GET /courses/{slug} extendidos con `average_rating` (float | null) y `ratings_count` (int). N+1 evitado en GET /courses con `RatingService.get_bulk_stats()`.

Boundary Server/Client en Next.js: `CourseDetailComponent` permanece Server Component. `RatingWidget` (Client Component con "use client") se monta dentro de él como Client Component island. El `userId` se inicializa en `null` y se asigna en `useEffect` para evitar errores de hidratación SSR/CSR.

Tipos TS: `average_rating` en `Course` es opcional (`average_rating?: number | null`) para no romper tests existentes sin ratings en el mock.

**Why:** feature de aprendizaje del curso de Platzi. No hay auth real, se usa UUID anónimo como identidad temporal.

**How to apply:** si se retoma la implementación, el plan completo está en `/Users/joel/code/learn/platzi__claude-code/RATINGS_IMPLEMENTATION_PLAN.md`.
