# FSD Reference — Layer Definitions & Segment Patterns

Detailed reference for Feature-Sliced Design implementation. See [SKILL.md](../SKILL.md) for overview.

## Layer Definitions (FSD v2.1)

### app Layer

**Purpose:** App-wide matters — technical (context providers) and business (analytics).

**Segments:** `entrypoint`, `styles`, `store`, `routes`. With Next.js, add `providers/`.

**Responsibilities:** Entrypoint, global providers (theme, auth, query client), global styles, router config, global store config. **Import rules:** Can import from all layers below. **No slices.**

### views Layer

**Purpose:** Pages (screens) that make up the app. One page usually corresponds to one slice.

**FSD v2.1:** No limit on code in a page slice. Unreused UI blocks can stay in the page slice. Typically `ui/`, `api/`. **Import rules:** Can import from widgets, features, entities, shared. **Has slices.**

### widgets Layer

**Purpose:** Large self-sufficient UI blocks. Reusable across multiple pages.

**FSD v2.1:** If a block makes up most of a page and is never reused, place it in the page slice — do not make it a widget. **Import rules:** Can import from features, entities, shared. **Has slices.**

**Tip:** With Next.js App Router, widgets can be full router blocks with data fetching, loading states, error boundaries.

### features Layer

**Purpose:** Main user interactions. Often involve entities.

**FSD v2.1:** Not everything needs to be a feature. Good indicator: reused on several pages. **Import rules:** Can import from entities, shared. **Has slices.**

### entities Layer

**Purpose:** Business domain concepts (User, Post, Group).

**Entity relationships (FSD v2.1):** Slices cannot know each other by default. Use `@x` cross-reference API when one entity's data contains another. See [Public API for cross-imports](https://feature-sliced.design/docs/reference/public-api#public-api-for-cross-imports).

**Import rules:** Can import from shared only. **Has slices.**

**@x cross-reference example:**

```typescript
// entities/artist/model/artist.ts
import type { Song } from "entities/song/@x/artist";
export interface Artist { name: string; songs: Array<Song>; }

// entities/song/@x/artist.ts  — public API for entities/artist only
export type { Song } from "../model/song";
```

### shared Layer

**Purpose:** Foundation — external connections (backends, libs, env) and internal libraries.

**Segments:** `api`, `ui`, `lib`, `config`, `i18n`, `routes`. Avoid `components`, `hooks`, `types` — use purpose-based names. `lib/` should NOT be a helpers dump; one focus per library, document in README.

**Import rules:** Cannot import from any FSD layer. **No slices.**

---

## Segment Patterns

### ui/ Segment

**Purpose:** React components and visual elements. Use for: any React component, UI composition, visual presentation.

### model/ Segment

**Purpose:** Business logic, state management, type definitions. Use for: TypeScript interfaces, React hooks for state, business logic functions, data transformations.

### api/ Segment

**Purpose:** API clients, data fetching, external integrations. Use for: HTTP requests, WebSocket connections, third-party APIs, Server actions (Next.js).

### lib/ Segment

**Purpose:** Slice-specific utilities. Use for: validation, data transformation helpers. One focus per library.

### config/ Segment

**Purpose:** Configuration constants and feature flags. Use for: feature-specific constants, configuration objects, feature flags.

---

## Valid vs Invalid Import Examples

**Invalid (cross-feature):**
```typescript
// ❌ src/features/search/ui/SearchBar.tsx
import { LoginForm } from '@/features/auth'; // Same layer
```

**Valid (extract to widget):**
```typescript
// ✅ src/widgets/navbar/ui/Navbar.tsx
import { SearchBar } from '@/features/search';
import { LoginForm } from '@/features/auth';
export function Navbar() { return <nav><SearchBar /><LoginForm /></nav>; }
```

---

## Migration Details

**Incremental migration:**
- Migrate one page at a time
- Start with least complex pages
- Use both old and new structure during transition
- Update imports as you migrate

**Testing throughout:** Run tests after each layer migration; verify import paths.
