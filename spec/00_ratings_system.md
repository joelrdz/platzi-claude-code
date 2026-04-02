# Análisis Técnico: Sistema de Ratings

**Versión**: 1.1
**Alcance**: Backend + Frontend

## Problema

Agregar la capacidad de que usuarios anónimos califiquen cursos con 1-5 estrellas. El rating de un usuario para un curso debe ser editable (si vuelve a calificar, se actualiza). Los resultados deben mostrarse como promedio en el listado de cursos y en la vista de detalle, con un widget interactivo en el detalle para enviar o cambiar la calificación.

El proyecto no tiene autenticación. La identidad del usuario se resuelve con un **UUID anónimo generado y persistido en `localStorage`** — compatible con la auth real cuando llegue, sin cambios al modelo de datos.

## Análisis Arquitectural del Contexto Actual

### Patrones identificados

- **Backend**: Service Layer Pattern + Dependency Injection con FastAPI `Depends()`
- **Database**: Soft deletes con `deleted_at`, timestamping automático via `BaseModel`
- **Frontend**: Next.js 15 App Router — Server Components por defecto, Client Components solo para interactividad de browser
- **Testing**: Vitest + React Testing Library con queries semánticas

### Arquitectura de datos actual

```
BaseModel (id, created_at, updated_at, deleted_at)
├── Course (name, description, thumbnail, slug)
├── Teacher (name, email)
├── Lesson (course_id, name, description, slug, video_url)
└── CourseTeacher (course_id, teacher_id) [Many-to-Many]
```

## Impacto Arquitectural

- **Base de datos**: nueva tabla `course_ratings` con `UniqueConstraint(course_id, user_id)` y `CheckConstraint(rating 1-5)`
- **Backend**: nuevo modelo `CourseRating`, nuevo `RatingService` con aggregate queries, modificación de `CourseService` para incluir stats, nuevo endpoint `POST /courses/{slug}/ratings`
- **Frontend**: nuevos campos en tipo `Course`, nuevo componente puro `StarRating`, nuevo Client Component `RatingWidget` (concentra toda la lógica de browser), integración en `CourseDetail` y tarjeta de listado

## Propuesta de Solución

### Contratos de API

**Nuevo endpoint: `POST /courses/{slug}/ratings`**

```json
// Request
{ "user_id": "550e8400-e29b-41d4-a716-446655440000", "rating": 4 }

// Response 200
{
  "course_id": 1,
  "user_id": "550e8400-e29b-41d4-a716-446655440000",
  "rating": 4,
  "average_rating": 4.2,
  "ratings_count": 17
}
```

Errores: `404` si el curso no existe, `422` si `rating` está fuera de rango o `user_id` no tiene 36 caracteres.

**Endpoints GET extendidos** (`GET /courses` y `GET /courses/{slug}`)

```json
"average_rating": 4.2,   // null cuando no hay ratings (no 0.0)
"ratings_count": 17      // 0 cuando no hay ratings
```

### Decisiones técnicas clave

**`average_rating` es `null`, no `0.0`** cuando no hay ratings — permite al frontend distinguir "sin datos aún" de un promedio calculado. Con escala 1-5 un promedio de 0 es imposible, pero la semántica importa.

**`get_bulk_stats()` resuelve el N+1 en `GET /courses`** — una sola query SQL con `AVG` + `COUNT` + `GROUP BY course_id` reemplaza una query de stats por curso. El enfoque alternativo (`@property` en el modelo ORM) cargaría todos los objetos `CourseRating` en memoria, que es inviable a escala.

**Upsert en dos pasos** (select + insert/update) — adecuado para la concurrencia actual. El `UniqueConstraint` en DB es la segunda línea de defensa. Para producción con alta concurrencia usar `INSERT ... ON CONFLICT DO UPDATE`.

**`UNIQUE(course_id, user_id)` simple** — no incluir `deleted_at` en el constraint. En SQL `NULL != NULL`, con lo cual `UNIQUE(course_id, user_id, deleted_at)` permite múltiples rows soft-deleted del mismo par, generando duplicados silenciosos. El upsert actualiza el registro existente en lugar de crear uno nuevo.

**`RatingService` separado** (no extender `CourseService`) — Single Responsibility: `CourseService` orquesta cursos, `RatingService` orquesta ratings. `CourseService` lo instancia internamente para evitar cambios en la firma de `Depends()`.

**`user_id: String(36)`** — UUID v4 en formato `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`. No `Integer`: sin auth real no hay entero significativo. Cuando llegue la auth, se reemplaza el UUID de localStorage con el ID del usuario autenticado sin cambios al schema.

### Boundary Server/Client en Next.js

`page.tsx` es un Server Component — no puede tener `useState`, `useEffect`, ni acceder a `localStorage`. La interactividad se encapsula en `RatingWidget` ("use client"), que recibe solo los datos que necesita como props.

```
app/course/[slug]/page.tsx        (Server Component)
  └── getCourseData()              → GET /courses/{slug} → incluye average_rating, ratings_count
  └── <CourseDetailComponent />
        ├── contenido estático     (server-rendered, sin cambios)
        └── <RatingWidget          ← boundary: aquí empieza el territorio client
              slug={course.slug}
              initialAverage={course.average_rating}
              initialCount={course.ratings_count}
            />
```

### Props de `StarRating`

`StarRating` es un componente **puro sin hooks** — el hover se maneja con CSS `:hover`, no con `useState`. Esto permite usarlo en Server Components (tarjeta de listado) sin necesidad de `"use client"`.

```tsx
interface StarRatingProps {
  value: number | null;
  max?: number;                            // default 5
  readonly?: boolean;                      // default false
  size?: 'small' | 'medium' | 'large';    // default 'medium'
  onRate?: (rating: number) => void;       // solo aplica cuando !readonly
}
```

## Plan de Implementación

### FASE 1 — Database Layer

**1.1 Modelo `CourseRating`**

Crear `Backend/app/models/course_rating.py`:

```python
class CourseRating(BaseModel):
    __tablename__ = 'course_ratings'

    course_id = Column(Integer, ForeignKey('courses.id'), nullable=False, index=True)
    user_id   = Column(String(36), nullable=False, index=True)  # UUID v4
    rating    = Column(Integer, nullable=False)

    course = relationship("Course", back_populates="ratings")

    __table_args__ = (
        UniqueConstraint('course_id', 'user_id', name='uq_course_rating_user'),
        CheckConstraint('rating >= 1 AND rating <= 5', name='ck_rating_range'),
    )
```

Modificar `Backend/app/models/course.py` — añadir relación inversa:

```python
ratings = relationship("CourseRating", back_populates="course", lazy="dynamic")
# lazy="dynamic" evita carga accidental; NO usar cascade="all, delete-orphan"
```

**1.2 Registrar en `__init__.py`**

Modificar `Backend/app/models/__init__.py` — añadir import de `CourseRating`.
Sin este paso, Alembic no detecta el modelo en `--autogenerate`.

**1.3 Migración**

```bash
make create-migration  # nombre: "add_course_ratings_table"
make migrate
```

---

### FASE 2 — Backend Service & API

**2.1 `RatingService`**

Crear `Backend/app/services/rating_service.py`:

```python
class RatingService:
    def __init__(self, db: Session): ...

    def upsert_rating(self, course_id: int, user_id: str, rating: int) -> Dict:
        # select por (course_id, user_id) → update si existe, insert si no
        # commit → retorna stats actualizados

    def get_rating_stats(self, course_id: int) -> Dict:
        # Una query: AVG + COUNT filtrado por course_id y deleted_at IS NULL

    def get_bulk_stats(self, course_ids: list[int]) -> Dict[int, Dict]:
        # Una query: AVG + COUNT + GROUP BY course_id para N cursos — evita N+1
        # Retorna dict keyed by course_id con default {average_rating: None, ratings_count: 0}
```

**2.2 Modificar `CourseService`**

Modificar `Backend/app/services/course_service.py`:

- `__init__`: instanciar `self.rating_service = RatingService(db)`
- `get_all_courses()`: obtener `bulk_stats` de todos los IDs, hacer `**bulk_stats[course.id]` en cada item
- `get_course_by_slug()`: obtener `stats` del curso, hacer `**stats` en el return

**2.3 Endpoint + Pydantic schema**

Modificar `Backend/app/main.py`:

```python
class RatingRequest(PydanticBaseModel):
    user_id: str = Field(..., min_length=36, max_length=36)
    rating:  int = Field(..., ge=1, le=5)

def get_rating_service(db: Session = Depends(get_db)) -> RatingService:
    return RatingService(db)

@app.post("/courses/{slug}/ratings")
def rate_course(slug: str, body: RatingRequest,
                course_service: CourseService = Depends(get_course_service),
                rating_service: RatingService = Depends(get_rating_service)) -> dict:
    course = course_service.get_course_by_slug(slug)
    if not course:
        raise HTTPException(status_code=404, detail="Course not found")
    return rating_service.upsert_rating(course["id"], body.user_id, body.rating)
```

---

### FASE 3 — Frontend Types & Components

**3.1 Tipos TypeScript**

Modificar `Frontend/src/types/index.ts`:

```typescript
export interface Course {
  // ...campos existentes sin cambio...
  average_rating: number | null;  // null → sin ratings aún
  ratings_count: number;
}

export interface RatingSubmit {
  user_id: string;
  rating: number;
}

export interface RatingResponse {
  course_id: number;
  user_id: string;
  rating: number;
  average_rating: number | null;
  ratings_count: number;
}
```

Los campos en **snake_case** — el backend los devuelve así y no existe capa de transformación en el proyecto.

**3.2 Componente `StarRating`**

Crear `Frontend/src/components/StarRating/StarRating.tsx` — componente puro, sin hooks:

```tsx
// Sin "use client" — compatible con Server Components
interface StarRatingProps {
  value: number | null;
  max?: number;
  readonly?: boolean;
  size?: 'small' | 'medium' | 'large';
  onRate?: (rating: number) => void;
}
```

- Hover manejado con CSS `:hover` (no `useState`)
- `aria-label` en cada estrella para accesibilidad
- `role="button"` cuando `!readonly`

Crear `Frontend/src/components/StarRating/StarRating.module.scss`

**3.3 `RatingWidget` (Client Component)**

Crear `Frontend/src/components/RatingWidget/RatingWidget.tsx`:

```tsx
"use client";

interface RatingWidgetProps {
  slug: string;
  initialAverage: number | null;
  initialCount: number;
}
```

Responsabilidades:

- `getOrCreateUserId()` — lee/escribe UUID en `localStorage` con `crypto.randomUUID()`
- `useState(null)` + `useEffect` para hidratar `userId` desde localStorage — **evita hydration mismatch**
- El widget interactivo se renderiza solo cuando `userId !== null` — evita flash sin identidad
- `handleRate()` — hace POST y actualiza estado local (`average`, `count`, `userRating`) con la respuesta
- Usa `NEXT_PUBLIC_API_URL` (env var) como base de la URL, con fallback a `http://localhost:8000`

Crear `Frontend/src/components/RatingWidget/RatingWidget.module.scss`

**3.4 Integrar en `CourseDetailComponent`**

Modificar `Frontend/src/components/CourseDetail/CourseDetail.tsx`:

```tsx
import { RatingWidget } from "@/components/RatingWidget/RatingWidget";

// En la sección de info del curso:
<RatingWidget
  slug={course.slug}
  initialAverage={course.average_rating}
  initialCount={course.ratings_count}
/>
```

`CourseDetailComponent` **no necesita** `"use client"` — el boundary se establece en el import de `RatingWidget`.

**3.5 `StarRating` en tarjeta de listado**

Modificar `Frontend/src/components/Course/Course.tsx`:

```tsx
import { StarRating } from "@/components/StarRating/StarRating";

// Desestructurar average_rating y ratings_count:
<div className={styles.ratingRow}>
  <StarRating value={average_rating} readonly size="small" />
  {ratings_count > 0 && <span className={styles.ratingCount}>({ratings_count})</span>}
</div>
```

---

### FASE 4 — Testing

**4.1 Backend**

Crear `Backend/app/tests/test_course_ratings.py`:

```python
def test_add_course_rating()          # POST devuelve rating + stats actualizados
def test_update_existing_rating()     # segundo POST del mismo user actualiza, no duplica
def test_rating_constraint_min()      # rating=0 → 422
def test_rating_constraint_max()      # rating=6 → 422
def test_rating_course_not_found()    # slug inexistente → 404
def test_average_rating_calculation() # AVG correcto con múltiples ratings
def test_bulk_stats_no_n_plus_one()   # GET /courses no genera N+1
def test_average_rating_null_when_empty() # curso sin ratings → average_rating: null
```

**4.2 Frontend**

Modificar `Frontend/src/components/Course/__test__/Course.test.tsx`:

- Añadir `average_rating` y `ratings_count` al `mockCourse`
- Test: renderiza estrellas cuando hay `average_rating`
- Test: no renderiza contador cuando `ratings_count === 0`

Crear `Frontend/src/components/StarRating/__test__/StarRating.test.tsx`:

```typescript
test('renders correct number of stars')
test('marks filled stars up to rounded value')
test('calls onRate when star is clicked in interactive mode')
test('does not call onRate when readonly')
test('has correct aria-labels for accessibility')
test('renders empty stars when value is null')
```

## Riesgos Técnicos


| Riesgo                                                   | Probabilidad | Mitigación                                                 |
| -------------------------------------------------------- | ------------ | ---------------------------------------------------------- |
| Hydration mismatch por `localStorage` en SSR             | Alta         | `useState(null)` + `useEffect` para hidratar `userId`      |
| N+1 en `GET /courses` al agregar stats                   | Alta         | `get_bulk_stats()` con un solo `GROUP BY`                  |
| Duplicados por concurrencia en upsert                    | Baja         | `UniqueConstraint` como segunda línea de defensa           |
| `StarRating` en Server Component requiere `"use client"` | Media        | Componente puro sin hooks, hover via CSS                   |
| Tests existentes de `Course` fallan por campos nuevos    | Alta         | `average_rating: number | null` hace los campos opcionales |


## Criterios de Aceptación

### Database

- Migración ejecuta sin errores con `make migrate`
- `UNIQUE(course_id, user_id)` impide duplicados activos
- `CHECK(rating >= 1 AND rating <= 5)` rechaza valores inválidos

### Backend API

- `POST /courses/{slug}/ratings` crea o actualiza (upsert)
- `GET /courses` incluye `average_rating` y `ratings_count` en cada item
- `GET /courses/{slug}` incluye `average_rating` y `ratings_count`
- `average_rating` es `null` (no `0`) cuando el curso no tiene ratings
- Rating con `user_id` inválido devuelve `422`
- Rating para curso inexistente devuelve `404`

### Frontend

- `StarRating` se renderiza correctamente en tamaños `small`, `medium`, `large`
- Click en estrella hace POST y actualiza el promedio mostrado sin recargar página
- El widget interactivo no aparece hasta que `userId` esté disponible (sin flash)
- Tarjeta de curso muestra promedio en modo readonly
- `CourseDetailComponent` no tiene `"use client"` — solo `RatingWidget`
- Tests existentes siguen pasando sin modificación

## Archivos afectados

### Backend


| Archivo                                              | Acción                                        |
| ---------------------------------------------------- | --------------------------------------------- |
| `app/models/course_rating.py`                        | Crear                                         |
| `app/services/rating_service.py`                     | Crear                                         |
| `app/alembic/versions/*_add_course_ratings_table.py` | Generar con `make create-migration`           |
| `app/models/course.py`                               | Modificar — relación `ratings`                |
| `app/models/__init__.py`                             | Modificar — registrar `CourseRating`          |
| `app/services/course_service.py`                     | Modificar — integrar `RatingService`          |
| `app/main.py`                                        | Modificar — `RatingRequest` schema + endpoint |


### Frontend


| Archivo                                                  | Acción    |
| -------------------------------------------------------- | --------- |
| `src/components/StarRating/StarRating.tsx`               | Crear     |
| `src/components/StarRating/StarRating.module.scss`       | Crear     |
| `src/components/StarRating/__test__/StarRating.test.tsx` | Crear     |
| `src/components/RatingWidget/RatingWidget.tsx`           | Crear     |
| `src/components/RatingWidget/RatingWidget.module.scss`   | Crear     |
| `src/types/index.ts`                                     | Modificar |
| `src/components/Course/Course.tsx`                       | Modificar |
| `src/components/Course/Course.module.scss`               | Modificar |
| `src/components/CourseDetail/CourseDetail.tsx`           | Modificar |
| `src/components/Course/__test__/Course.test.tsx`         | Modificar |


