# Plan de Implementación Backend: Sistema de Ratings

**Versión**: 1.1
**Alcance**: Backend (FastAPI + PostgreSQL + SQLAlchemy)
**Stack**: FastAPI, SQLAlchemy 2.0, PostgreSQL 15, Alembic, pytest
**Prerequisito**: Leer `spec/00_ratings_system.md`

---

## Contexto Arquitectural

### Patrones identificados en el código actual

1. **Service Layer Pattern**: Toda la lógica de negocio está en servicios (`CourseService`)
2. **Dependency Injection**: FastAPI `Depends()` inyecta la sesión de DB en los servicios
3. **Soft Deletes**: Campo `deleted_at` en `BaseModel` — nunca hard-delete
4. **Eager Loading**: `joinedload()` para relaciones en queries
5. **Tests AAA**: Arrange-Act-Assert con mocks de la sesión de DB

### Estructura actual relevante

```
Backend/app/
├── alembic/versions/
├── models/
│   ├── base.py            # BaseModel con id, timestamps, deleted_at
│   ├── course.py
│   └── __init__.py        # Registro de modelos (necesario para Alembic)
├── services/
│   └── course_service.py
├── db/base.py             # Engine + SessionLocal + get_db()
├── main.py                # FastAPI app + endpoints
└── test_main.py           # Tests con TestClient + dependency_overrides
```

---

## FASE 1: Database Layer

### 1.1 Crear `app/models/course_rating.py`

Nuevo modelo ORM que hereda de `BaseModel`:

```python
class CourseRating(BaseModel):
    __tablename__ = 'course_ratings'

    course_id = Column(Integer, ForeignKey('courses.id'), nullable=False, index=True)
    user_id   = Column(String(36), nullable=False, index=True)  # UUID v4 anónimo
    rating    = Column(Integer, nullable=False)

    course = relationship("Course", back_populates="ratings")

    __table_args__ = (
        UniqueConstraint('course_id', 'user_id', name='uq_course_rating_user'),
        CheckConstraint('rating >= 1 AND rating <= 5', name='ck_rating_range'),
    )

    def __repr__(self):
        return f"<CourseRating(id={self.id}, course_id={self.course_id}, user_id={self.user_id}, rating={self.rating})>"

    def to_dict(self):
        return {
            "id": self.id,
            "course_id": self.course_id,
            "user_id": self.user_id,
            "rating": self.rating,
            "created_at": self.created_at.isoformat() if self.created_at else None,
            "updated_at": self.updated_at.isoformat() if self.updated_at else None,
        }
```

**Decisiones de diseño:**

- `user_id: String(36)` — UUID v4 anónimo generado en el frontend (`crypto.randomUUID()`). Sin FK a una tabla de usuarios porque no existe auth aún. Cuando llegue la auth real, se reemplaza el UUID de localStorage sin cambios al schema.
- `UniqueConstraint('course_id', 'user_id')` **sin incluir `deleted_at`** — en SQL `NULL != NULL`, así que si se incluyera `deleted_at`, múltiples filas con `deleted_at IS NULL` del mismo par pasarían el constraint (ya que todos los NULLs son distintos entre sí). La constraint sin `deleted_at` es la correcta para garantizar un solo rating activo por usuario-curso.
- `lazy="dynamic"` **no** en la relación — se define con el default (`select`), pero en `Course` se usará `lazy="dynamic"` para evitar carga accidental.

### 1.2 Modificar `app/models/course.py`

Agregar la relación inversa en la clase `Course`:

```python
ratings = relationship("CourseRating", back_populates="course", lazy="dynamic")
```

`lazy="dynamic"` es obligatorio aquí: evita que cualquier acceso accidental al atributo `course.ratings` dispare un `SELECT * FROM course_ratings`. La relación inversa existe para que Alembic pueda construir el grafo de relaciones — en el código de servicio nunca se accede a `course.ratings` directamente; siempre se usa `RatingService`.

No agregar `@property` de `average_rating` ni `total_ratings` en el modelo ORM. Calcularlos con Python recorrería todos los objetos `CourseRating` en memoria — inviable a escala. Las agregaciones se hacen en el service layer con SQL.

### 1.3 Modificar `app/models/__init__.py`

Agregar import y export de `CourseRating`. Sin este paso, Alembic no detecta el modelo en `--autogenerate` — el `env.py` de Alembic importa `Base` desde este `__init__`, así que el registro aquí es el trigger para la detección automática.

### 1.4 Generar y ejecutar la migración

```bash
make create-migration   # Nombre sugerido: "add_course_ratings_table"
```

Revisar el archivo generado en `app/alembic/versions/` antes de ejecutar. Verificar que `op.create_table` incluye:
- Las columnas: `id`, `created_at`, `updated_at`, `deleted_at`, `course_id`, `user_id`, `rating`
- `UniqueConstraint('course_id', 'user_id', name='uq_course_rating_user')`
- `CheckConstraint('rating >= 1 AND rating <= 5', name='ck_rating_range')`
- `ForeignKeyConstraint(['course_id'], ['courses.id'])`
- Índices en `course_id` y `user_id`

La función `downgrade()` debe eliminar primero los índices y luego la tabla.

```bash
make migrate
```

### Criterios de validación de la Fase 1

- `make migrate` ejecuta sin errores
- `\d course_ratings` en la DB muestra columnas, constraints e índices correctos
- `INSERT` con `rating=0` o `rating=6` falla con error de constraint en DB
- Dos `INSERT` con el mismo `(course_id, user_id)` fallan con unique constraint
- `make start` levanta sin errores de importación

**Dependencias:** Ninguna — es la base de todo.

---

## FASE 2: Rating Service

### 2.1 Crear `app/services/rating_service.py`

Nueva clase `RatingService` con `__init__(self, db: Session)`. Tres métodos:

**`get_rating_stats(self, course_id: int) -> Dict`**

Una sola query SQL: `AVG(rating)` y `COUNT(*)` sobre `CourseRating` filtrado por `course_id` y `deleted_at IS NULL`.

Retorna `{"average_rating": float | None, "ratings_count": int}`:
- `average_rating` es `None` cuando `COUNT = 0` — no `0.0`. Con escala 1-5 un promedio de cero es imposible; el `None` permite al frontend distinguir "sin datos" de un promedio real.
- Usar `func.avg()` sin `coalesce` y comprobar si el resultado es `None` antes de retornar.

**`get_bulk_stats(self, course_ids: list[int]) -> Dict[int, Dict]`**

Una sola query con `GROUP BY course_id` — `AVG` y `COUNT` para todos los IDs en una pasada. Resuelve el problema N+1 en `GET /courses`.

Construye y retorna `{course_id: {"average_rating": ..., "ratings_count": ...}}`. Para IDs sin ratings, el valor por defecto es `{"average_rating": None, "ratings_count": 0}` — todo `course_id` de la lista de entrada debe tener entry en el dict de retorno aunque no aparezca en los resultados SQL.

**`upsert_rating(self, course_id: int, user_id: str, rating: int) -> Dict`**

1. Select por `(course_id, user_id)` con `deleted_at IS NULL`
2. Si existe: `UPDATE rating`, `updated_at = now()`
3. Si no existe: `INSERT` nuevo registro
4. `db.commit()`
5. Llama a `get_rating_stats(course_id)` y retorna `{"course_id": ..., "user_id": ..., "rating": ..., **stats}`

El `UniqueConstraint` en DB actúa como segunda línea de defensa ante condiciones de carrera — el upsert en dos pasos es adecuado para la concurrencia actual del proyecto.

### Criterios de validación de la Fase 2

- `get_bulk_stats([])` retorna `{}` sin error
- `get_bulk_stats([id_sin_ratings])` retorna `{id: {"average_rating": None, "ratings_count": 0}}`
- `upsert_rating` llamado dos veces con el mismo par produce exactamente un registro en DB
- `average_rating` es `None` para un curso sin ratings, no `0.0`

**Dependencias:** Fase 1 completa.

---

## FASE 3: Modificar CourseService

### 3.1 Modificar `app/services/course_service.py`

**En `__init__`**: instanciar `self.rating_service = RatingService(db)`.

**En `get_all_courses()`**:
- Extraer lista de `course.id` de los cursos obtenidos
- Llamar `self.rating_service.get_bulk_stats(course_ids)` — una sola llamada para todos los cursos
- En el list comprehension, añadir `**bulk_stats.get(course.id, {"average_rating": None, "ratings_count": 0})` a cada dict

**En `get_course_by_slug()`**:
- Tras obtener el curso, llamar `self.rating_service.get_rating_stats(course.id)`
- Añadir `**stats` al dict de retorno

> **Atención:** `TestContractCompliance.test_courses_list_contract_fields_only` fallará — los sets `expected_fields` deben actualizarse para incluir `"average_rating"` y `"ratings_count"`. Es una expansión intencional del contrato, no un test roto.

### Criterios de validación de la Fase 3

- `GET /courses` devuelve `average_rating` y `ratings_count` en cada objeto
- `GET /courses/{slug}` devuelve `average_rating` y `ratings_count`
- Curso sin ratings devuelve `"average_rating": null`, `"ratings_count": 0`
- Solo una query SQL para todos los stats de todos los cursos (N+1 resuelto)

**Dependencias:** Fase 2 completa.

---

## FASE 4: Endpoint POST y schema Pydantic

### 4.1 Modificar `app/main.py`

**Schema Pydantic** (inline, el proyecto no tiene carpeta `schemas/`):

```python
class RatingRequest(PydanticBaseModel):
    user_id: str = Field(..., min_length=36, max_length=36)  # UUID v4
    rating:  int = Field(..., ge=1, le=5)
```

**Dependency factory:**

```python
def get_rating_service(db: Session = Depends(get_db)) -> RatingService:
    return RatingService(db)
```

**Endpoint:**

```python
@app.post("/courses/{slug}/ratings")
def rate_course(
    slug: str,
    body: RatingRequest,
    course_service: CourseService = Depends(get_course_service),
    rating_service: RatingService = Depends(get_rating_service)
) -> dict:
    course = course_service.get_course_by_slug(slug)
    if not course:
        raise HTTPException(status_code=404, detail="Course not found")
    return rating_service.upsert_rating(course["id"], body.user_id, body.rating)
```

El endpoint usa `slug` (no `course_id`) para ser consistente con el resto de la API. Recibe dos dependencias independientes — FastAPI reutiliza la misma sesión de DB dentro del mismo request si el callable de `Depends()` es el mismo.

### Criterios de validación de la Fase 4

- `POST /courses/{slug}/ratings` con body válido → `200` con `course_id`, `user_id`, `rating`, `average_rating`, `ratings_count`
- Slug inexistente → `404`
- `rating: 0` o `rating: 6` → `422` (validación Pydantic, antes de tocar DB)
- `user_id` de 35 o 37 chars → `422`
- Dos POSTs consecutivos con el mismo `user_id` y `slug` producen exactamente un registro en DB
- `GET http://localhost:8000/docs` muestra el nuevo endpoint con schema correcto

**Dependencias:** Fase 3 completa.

---

## FASE 5: Tests

### 5.1 Actualizar `app/test_main.py`

Dos tests de `TestContractCompliance` fallarán al añadir los nuevos campos:
- `test_courses_list_contract_fields_only`: añadir `"average_rating"` y `"ratings_count"` al set `expected_fields`
- `test_course_detail_contract_fields_only`: ídem en `expected_course_fields`

Los dicts de datos mock (`MOCK_COURSES_LIST`, `MOCK_COURSE_DETAIL`) también deben incluir los nuevos campos.

### 5.2 Crear `app/tests/test_course_ratings.py`

Crear el directorio `app/tests/` con un `__init__.py` vacío. Los tests siguen el patrón de `test_main.py`: `TestClient` + `dependency_overrides` para inyectar mocks de `CourseService` y `RatingService`.

**Estructura de fixtures:**

```python
@pytest.fixture
def mock_course_service():
    service = Mock()
    service.get_course_by_slug.return_value = {
        "id": 1, "name": "Test Course", "slug": "test-course",
        "average_rating": None, "ratings_count": 0
    }
    return service

@pytest.fixture
def mock_rating_service():
    service = Mock()
    service.upsert_rating.return_value = {
        "course_id": 1,
        "user_id": "550e8400-e29b-41d4-a716-446655440000",
        "rating": 4,
        "average_rating": 4.0,
        "ratings_count": 1,
    }
    return service

@pytest.fixture
def client(mock_course_service, mock_rating_service):
    app.dependency_overrides[get_course_service] = lambda: mock_course_service
    app.dependency_overrides[get_rating_service] = lambda: mock_rating_service
    yield TestClient(app)
    app.dependency_overrides.clear()
```

**Tests a implementar:**

| Test | Qué verifica |
|---|---|
| `test_add_course_rating` | POST válido devuelve 200 con todos los campos del contrato |
| `test_update_existing_rating` | Segundo POST del mismo `user_id` retorna el rating actualizado |
| `test_rating_constraint_min` | `rating: 0` → 422, servicio no llamado |
| `test_rating_constraint_max` | `rating: 6` → 422, servicio no llamado |
| `test_rating_user_id_too_short` | `user_id` de 35 chars → 422 |
| `test_rating_user_id_too_long` | `user_id` de 37 chars → 422 |
| `test_rating_course_not_found` | `get_course_by_slug` retorna `None` → 404 |
| `test_average_rating_null_when_empty` | `upsert_rating` retorna `average_rating: None` → response contiene `null`, no `0` |

Los tests de validación Pydantic (`rating: 0`, `user_id` corto) no necesitan mock del servicio — la validación ocurre antes de cualquier `Depends()`. Se puede verificar con `mock_rating_service.upsert_rating.assert_not_called()`.

### Criterios de validación de la Fase 5

- Todos los tests pasan incluyendo los actualizados de `TestContractCompliance`
- Ningún test que pasaba antes ahora falla (salvo los dos actualizados en 5.1)

**Dependencias:** Fase 4 completa.

---

## Orden de ejecución

```
Fase 1 (DB) → Fase 2 (RatingService) → Fase 3 (CourseService) → Fase 4 (Endpoint) → Fase 5 (Tests)
```

Las fases son estrictamente secuenciales: cada una presupone que la anterior está validada.

---

## Trampas específicas de este codebase

**`UniqueConstraint` sin `deleted_at`** — el spec del curso incluye `deleted_at` en el constraint. Esto es un bug: en SQL, `NULL != NULL`, por lo que dos filas con `deleted_at IS NULL` y el mismo `(course_id, user_id)` son tratadas como distintas por el constraint y ambas pasarían. La constraint correcta es solo `(course_id, user_id)`.

**`lazy="dynamic"` en `Course.ratings`** — sin él, cualquier acceso a `course.ratings` en contexto cargado haría `SELECT *` sobre `course_ratings`. La relación inversa existe solo para que Alembic construya el grafo correctamente — nunca acceder a `course.ratings` en código de servicio.

**`get_bulk_stats` debe cubrir IDs sin ratings** — usar `.get(course.id, default)` en lugar de `[course.id]` para evitar `KeyError` cuando un curso no tiene ratings y por tanto no aparece en los resultados SQL.

**`TestContractCompliance` son guardianes del contrato** — los asserts `actual_fields == expected_fields` fallan intencionalmente al expandir el contrato. Actualizarlos es lo correcto, no un workaround.

**`average_rating: None` vs `0.0`** — usar `func.avg()` sin `coalesce`. Cuando no hay ratings, `AVG` retorna `NULL` en SQL que SQLAlchemy convierte a `None` en Python — exactamente lo que se quiere. Agregar `coalesce(..., 0.0)` enmascara la ausencia de datos.
