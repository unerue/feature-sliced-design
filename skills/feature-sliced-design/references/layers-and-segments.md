# FSD Reference — Layer Definitions & Segment Patterns

Detailed reference for Feature-Sliced Design implementation with Next.js. See [SKILL.md](../SKILL.md) for overview.

> **Next.js note:** This skill uses **views** instead of **pages**. Routing lives in `src/app/` (Next.js + FSD App combined); page logic lives in `src/views/`.

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

**Entity relationships (FSD v2.1):** Slices cannot know each other by default. Use `@x` cross-reference API when one entity's data contains another. See [public-api.md](./public-api.md#x-cross-imports-entities-layer-only).

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

## Slices

Source: [Slices and segments](https://feature-sliced.design/docs/reference/slices-segments)

Slices are the second level in the FSD hierarchy. They group code by **meaning for the product, business, or application**.

**Slice names** are not standardized — they depend on your domain. Examples:
- Photo gallery: `photo`, `effects`, `gallery-page`
- Social network: `post`, `comments`, `news-feed`

**Shared and App have no slices:** Shared has no business logic (no product meaning); App concerns the whole application (no need to split).

### Zero coupling, high cohesion

An ideal slice is:
- **Zero coupling** — independent of other slices on the same layer.
- **High cohesion** — contains most code for its primary goal.

**Enforcement:** A module can only import slices on layers **strictly below** (import rule).

### Public API rule on slices

Every slice must define a **public API**. Code inside a slice can be organized freely, but:
- **Modules outside a slice may only use its public API**, not internal paths.

See [public-api.md](./public-api.md) for details.

---

## Slice Groups

Related slices can live in a folder (e.g. `features/post/compose`, `features/post/like`, `features/post/delete`), but:

> **No code sharing inside that folder.** Each slice must stay isolated. No `some-shared-code.ts` at the group level.

---

## Segment Names — Purpose, Not Essence

Segment names should describe **purpose (why)**, not **essence (what)**.

| Essence (avoid) | Purpose (use) |
|-----------------|---------------|
| `components` | `ui` |
| `hooks` | `model` |
| `types` | `model` |
| `utils` | `lib` |

See [code-smells.md](./code-smells.md) for more.

---

## Segments

Segments are the third level. They group code by **technical purpose** (not business domain).

**Standardized names:** `ui`, `api`, `model`, `lib`, `config`. Custom segments are common in App and Shared, where slices don't exist.

**Name by purpose, not essence** — e.g. avoid `components`, `hooks`, `types`.

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

### From custom architecture (bottom-up)

- Migrate one page at a time
- Start with least complex pages
- Use both old and new structure during transition
- Update imports as you migrate

**Testing throughout:** Run tests after each layer migration; verify import paths.

### From FSD v2.0 to v2.1

Source: [Migration from v2.0 to v2.1](https://feature-sliced.design/docs/guides/migration/from-v2-0)

**Mental model change:** v2.1 uses **pages-first** decomposition. Start with pages (or views); keep most UI and logic in each page. Only extract to features/entities when reuse across several pages is needed. Shared remains the reusable foundation.

**1. Merge slices**

Use [Steiger](https://github.com/feature-sliced/steiger) to find merge candidates:

```bash
npx steiger src
```

- **insignificant-slice** — Entity or feature used in only one page → consider merging into that page.
- **excessive-slicing** — Too many slices on a layer → merge or group to improve navigation.

**Layer = global namespace.** Avoid slices that are used only once; treat each slice as valuable.

**2. Standardize cross-imports**

If you have cross-imports between entities, migrate to `@x` notation. See [public-api.md](./public-api.md#x-cross-imports-entities-layer-only).

---

## Related References

- [public-api.md](./public-api.md) — Public API rules, @x cross-imports, shared/ui structure
- [code-smells.md](./code-smells.md) — Desegmentation, generic folders
