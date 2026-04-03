# Plan de Implementación Frontend: Sistema de Ratings

**Versión**: 1.1
**Alcance**: Frontend (Next.js 15 + TypeScript)
**Stack**: Next.js 15, React 19, TypeScript, SCSS Modules, Vitest

---

## Contexto

### Arquitectura actual relevante

```
Frontend/src/
├── app/
│   ├── course/[slug]/page.tsx   # Server Component, hace fetch GET /courses/{slug}
│   └── page.tsx                  # Server Component, hace fetch GET /courses
├── components/
│   ├── Course/                   # Tarjeta del listado
│   ├── CourseDetail/             # Vista completa del curso
│   └── VideoPlayer/
└── types/index.ts               # Tipos globales
```

### Patrones a respetar

- Server Components por defecto — `"use client"` solo cuando hay interactividad de browser
- CSS Modules + SASS, sin Tailwind, sin estilos inline
- Tests en `__test__/` junto al componente, con Vitest + React Testing Library
- Queries semánticas: `getByRole`, `getByText`, `getByLabelText`
- Path alias `@/*` → `src/*`

---

## FASE 1: Tipos TypeScript

**Archivo**: `src/types/index.ts`

### Cambios

**Extender la interfaz `Course` existente:**

```typescript
export interface Course {
  // ...campos existentes sin cambio...
  average_rating: number | null;  // null → sin ratings aún (no usar 0)
  ratings_count: number;
}
```

`average_rating: null` (no `0`) cuando no hay ratings. Con escala 1-5 un promedio de 0 es imposible — el `null` permite al frontend distinguir "sin datos" de un promedio real.

**Nuevas interfaces:**

```typescript
export interface RatingSubmit {
  user_id: string;   // UUID v4 de 36 caracteres
  rating: number;    // 1-5
}

export interface RatingResponse {
  course_id: number;
  user_id: string;
  rating: number;
  average_rating: number | null;
  ratings_count: number;
}
```

**Nuevo tipo de estado:**

```typescript
export type RatingState = 'idle' | 'loading' | 'success' | 'error';
```

Útil para controlar el estado del widget durante el submit.

Los campos en **snake_case** — consistente con el patrón ya establecido en `Progress` y `FavoriteToggle`.

### Criterios de validación

- `pnpm build` compila sin errores de TypeScript
- `pnpm test --run` pasa: los tests existentes de `Course` no se rompen — `CourseProps` es `Omit<CourseType, "slug">` y el mock del test no necesita los campos nuevos si están tipados como `| null`

**Dependencias:** Ninguna — fase completamente independiente.

---

## FASE 2: Componente puro `StarRating`

**Archivos**:
- `src/components/StarRating/StarRating.tsx`
- `src/components/StarRating/StarRating.module.scss`

### Por qué debe ser puro

`StarRating` es el único componente de ratings que se usará en Server Components (tarjeta del listado). Si usara `useState` para el hover, requeriría `"use client"` y contaminaría el boundary. La solución es manejar el hover exclusivamente con CSS `:hover`.

### `StarRating.tsx`

```typescript
interface StarRatingProps {
  value: number | null;
  max?: number;                           // default 5
  readonly?: boolean;                     // default false
  size?: 'small' | 'medium' | 'large';   // default 'medium'
  onRate?: (rating: number) => void;      // solo aplica cuando !readonly
}
```

Comportamiento:
- **Sin `"use client"`** — requisito central
- Renderizar `max` elementos: `<span>` en readonly, `<button>` en modo interactivo
- Estrella llena: índice ≤ `Math.round(value)`. Cuando `value` es `null`, todas vacías
- Click: llama `onRate(index + 1)` si `!readonly`

**Accesibilidad:**
- `role="group"` en el contenedor con `aria-label="Calificación: N de 5 estrellas"`
- Cada estrella: `aria-label="N estrellas"`
- `role="button"` y `aria-pressed` cuando `!readonly`
- `tabIndex={-1}` en modo readonly (no navegable por teclado)
- Navegación por teclado: `ArrowLeft`/`ArrowRight` entre estrellas, `Enter`/`Space` para seleccionar

### `StarRating.module.scss`

- Tres clases de tamaño: `.small`, `.medium`, `.large`
- Color estrella activa: `#ff2d2d` (primary del sistema de diseño existente)
- Color estrella vacía: text-secondary con opacidad reducida
- Hover en modo interactivo: CSS puro — `flex-direction: row-reverse` + selector `~ span` para iluminar estrellas anteriores al hover
- Transición suave (200ms) en `color` y `transform`
- `cursor: pointer` solo en modo interactivo

### Criterios de validación

- `pnpm build` compila — verificar que el archivo **no** contiene `"use client"`
- Se renderiza en el servidor sin hydration errors
- Los tres tamaños son visualmente correctos

**Dependencias:** Fase 1 completa.

---

## FASE 3: `RatingWidget` (Client Component)

**Archivos**:
- `src/components/RatingWidget/RatingWidget.tsx`
- `src/components/RatingWidget/RatingWidget.module.scss`

### Responsabilidades

Este es el único componente con lógica de browser: gestión de identidad anónima, estado interactivo y comunicación con la API. Todo lo que necesita el browser está aquí; nada más necesita `"use client"`.

### `RatingWidget.tsx`

```typescript
"use client";

interface RatingWidgetProps {
  slug: string;
  initialAverage: number | null;
  initialCount: number;
}
```

**Identidad anónima:**

```typescript
function getOrCreateUserId(): string {
  let userId = localStorage.getItem('platziflix_user_id');
  if (!userId) {
    userId = crypto.randomUUID();
    localStorage.setItem('platziflix_user_id', userId);
  }
  return userId;
}
```

**Patrón anti-hydration mismatch:**

```typescript
const [userId, setUserId] = useState<string | null>(null);
// ↑ inicia en null deliberadamente — localStorage no existe en el servidor

useEffect(() => {
  setUserId(getOrCreateUserId());
}, []);
// ↑ hidrata userId solo en el cliente, post-montaje
```

**Estados:**

```typescript
const [ratingState, setRatingState] = useState<RatingState>('idle');
const [average, setAverage] = useState(initialAverage);
const [count, setCount] = useState(initialCount);
const [userRating, setUserRating] = useState<number | null>(null);
```

**Manejo de submit:**

```typescript
async function handleRate(rating: number) {
  if (!userId) return;
  setRatingState('loading');

  try {
    const res = await fetch(
      `${process.env.NEXT_PUBLIC_API_URL ?? 'http://localhost:8000'}/courses/${slug}/ratings`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ user_id: userId, rating }),
      }
    );

    if (!res.ok) throw new Error(`HTTP ${res.status}`);

    const data: RatingResponse = await res.json();
    setUserRating(data.rating);
    setAverage(data.average_rating);
    setCount(data.ratings_count);
    setRatingState('success');
  } catch {
    setRatingState('error');
  }
}
```

**Estrategia de renderizado:**

- `userId === null` → SSR-safe: muestra `<StarRating>` en readonly con `initialAverage` (sin flash)
- `userId !== null` → widget interactivo completo con la identidad del usuario

El feedback de `ratingState` (loading, success, error) se muestra junto al widget interactivo.

### Criterios de validación

- Sin hydration warnings en la consola del navegador
- DevTools → Application → LocalStorage: aparece `platziflix_user_id` tras primera visita
- DevTools → Network: click en estrella dispara `POST /courses/{slug}/ratings` con body correcto
- El widget interactivo **no** aparece en el HTML del servidor (inspeccionar page source)
- Si el backend no responde, el widget falla gracefully (estado `error`, sin crash)

**Dependencias:** Fase 1 (tipos `RatingSubmit`, `RatingResponse`, `RatingState`), Fase 2 (`StarRating` disponible).

---

## FASE 4: Integración en `CourseDetail`

**Archivo**: `src/components/CourseDetail/CourseDetail.tsx`

### Cambios

Importar `RatingWidget` y añadirlo en la sección de info del curso, después de `div.stats`:

```tsx
<RatingWidget
  slug={course.slug}
  initialAverage={course.average_rating}
  initialCount={course.ratings_count}
/>
```

`CourseDetail.tsx` **no recibe `"use client"`** — el boundary lo establece el import de `RatingWidget`. Importar un Client Component desde un Server Component es el patrón correcto de Next.js.

**`src/app/course/[slug]/page.tsx` no requiere cambios** — el fetch ya existe y el tipo `CourseDetail` ya incluirá los nuevos campos tras la Fase 1.

### Criterios de validación

- `pnpm build` sin errores de TypeScript
- Navegando a `/course/[slug]` aparece el widget en la sección de info
- `CourseDetail.tsx` **no tiene** `"use client"`
- React DevTools: `CourseDetailComponent` como Server Component, `RatingWidget` como Client Component

**Dependencias:** Fase 3 completa. Backend Fase 3 para datos en runtime (puede avanzar sin backend — los valores llegarán como `undefined` hasta que esté listo).

---

## FASE 5: Integración en tarjeta `Course`

**Archivos**:
- `src/components/Course/Course.tsx`
- `src/components/Course/Course.module.scss`

### Cambios

`Course.tsx` — desestructurar `average_rating` y `ratings_count`, añadir dentro de `courseInfo`:

```tsx
<div className={styles.ratingRow}>
  <StarRating value={average_rating} readonly size="small" />
  {ratings_count > 0 && (
    <span className={styles.ratingCount}>({ratings_count})</span>
  )}
</div>
```

`Course.tsx` **no recibe `"use client"`** — `StarRating` es un componente puro. Las estrellas se renderizan en el servidor.

`Course.module.scss`:
- `.ratingRow`: `display: flex`, `align-items: center`, `gap: 0.5rem`
- `.ratingCount`: estilos de texto secundario, consistente con `.duration`

### Criterios de validación

- `pnpm test --run` pasa — los tests existentes de `Course` no se rompen
- Cursos sin ratings no muestran el contador `(0)`
- Las estrellas aparecen en el HTML del servidor (SSR, sin hidratación)

**Dependencias:** Fase 2 (`StarRating`), Fase 1 (tipos). Backend Fase 2 para datos en runtime.

---

## FASE 6: Tests

### 6.1 Actualizar `src/components/Course/__test__/Course.test.tsx`

- Añadir `average_rating: 4.2` y `ratings_count: 17` al `mockCourse` existente
- Test nuevo: cuando `average_rating` no es null, verificar que se renderizan estrellas (por role o aria-label)
- Test nuevo: cuando `ratings_count === 0`, verificar que el contador `(0)` no está en el DOM

### 6.2 Crear `src/components/StarRating/__test__/StarRating.test.tsx`

| Test | Qué verifica |
|---|---|
| `renders correct number of stars` | Render con `max=5`, existen 5 elementos |
| `marks filled stars up to rounded value` | `value=3` → 3 con clase filled, 2 con clase empty |
| `calls onRate when star is clicked` | `readonly=false`, `userEvent.click` en estrella 4 → `onRate(4)` llamado |
| `does not call onRate when readonly` | `readonly=true`, click → `onRate` no llamado |
| `has correct aria-labels` | Cada estrella tiene `aria-label` apropiado |
| `renders empty stars when value is null` | `value=null` → ninguna estrella tiene la clase filled |
| `does not render count when ratings_count is 0` | `ratings_count=0` → contador ausente del DOM |
| `renders count when ratings_count is positive` | `ratings_count=17` → aparece `(17)` |

> **Tests de `RatingWidget`:** Requieren mockear `localStorage`, `fetch` y `crypto.randomUUID` (web APIs en JSDOM). Son candidatos a una fase posterior dado el setup necesario.

### Criterios de validación

- `pnpm test --run` pasa al 100%, sin warnings de tipos
- Los tests preexistentes de `Course` pasan sin modificar sus assertions — solo se extiende el mock

**Dependencias:** Todas las fases anteriores completas.

---

## Mapa de dependencias

```
Fase 1 (tipos)
  ├── Fase 2 (StarRating puro)
  │     ├── Fase 5 (tarjeta Course)
  │     └── Fase 3 (RatingWidget client)
  │           └── Fase 4 (integración CourseDetail)
  └── Fase 6 (tests) ← depende de todas
```

Las Fases 2 y 3 pueden trabajarse en paralelo — Fase 3 importa Fase 2 al cierre.

---

## Dependencias con el backend

| Fase frontend | Qué necesita del backend | Puede avanzar sin backend |
|---|---|---|
| 1, 2, 3 | Nada | Sí, completamente |
| 4 | `GET /courses/{slug}` retorne `average_rating`, `ratings_count` | Sí (campos llegan `undefined`, widget muestra `null`) |
| 5 | `GET /courses` retorne `average_rating`, `ratings_count` | Sí (misma situación) |
| 3 (POST) | `POST /courses/{slug}/ratings` funcional | Sí (widget renderiza, click falla gracefully) |

Las primeras tres fases son completamente independientes del backend.
