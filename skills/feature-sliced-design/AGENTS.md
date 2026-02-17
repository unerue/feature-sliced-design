# Feature-Sliced Design Architecture

**Version 0.4.0**  
Engineering  
February 2026

> **Note:**  
> This document is mainly for agents and LLMs to follow when maintaining,  
> generating, or refactoring Next.js codebases using Feature-Sliced Design.  
> Humans may also find it useful, but guidance here is optimized for automation  
> and consistency by AI-assisted workflows.

---

## Abstract

Architectural guidance for implementing Feature-Sliced Design (FSD v2.1) in Next.js applications. Organizes code into a layered hierarchy (app, views, widgets, features, entities, shared) that prevents circular dependencies and promotes maintainability.

**Next.js customization:** `src/app/` merges Next.js App Router with FSD App layer. Page logic lives in `src/views/` (not `pages`). All other FSD rules align with official docs.

---

## Table of Contents

1. [Layer Hierarchy & Import Rules](#1-layer-hierarchy--import-rules) — **CRITICAL**
   - 1.1 [Strict Unidirectional Imports](#11-strict-unidirectional-imports)
   - 1.2 [No Cross-Slice Imports Within Same Layer](#12-no-cross-slice-imports-within-same-layer)
2. [Public API Pattern](#2-public-api-pattern) — **HIGH**
   - 2.1 [Export Through index.ts](#21-export-through-indexts)
   - 2.2 [Avoid Deep Imports](#22-avoid-deep-imports)
3. [Views vs App Separation](#3-views-vs-app-separation) — **HIGH**
   - 3.1 [Routing in app, Logic in views](#31-routing-in-app-logic-in-views)
   - 3.2 [page.tsx Imports from views Only](#32-pagetsx-imports-from-views-only)
4. [Segment Structure](#4-segment-structure) — **MEDIUM**
   - 4.1 [Use Purpose-Based Segments](#41-use-purpose-based-segments)
   - 4.2 [Naming Conventions](#42-naming-conventions)
5. [Resolving Circular Dependencies](#5-resolving-circular-dependencies) — **HIGH**
   - 5.1 [Extract to Lower Layer](#51-extract-to-lower-layer)
   - 5.2 [Extract to Higher Layer (Widget)](#52-extract-to-higher-layer-widget)

---

## 1. Layer Hierarchy & Import Rules

**Impact: CRITICAL**

Fundamental rules for FSD. Violations cause circular dependencies and unmaintainable architecture.

### 1.1 Strict Unidirectional Imports

**Impact: CRITICAL (prevents circular dependencies)**

A module can only import from layers **strictly below** it. Never import upward.

```
app → views → widgets → features → entities → shared
```

**Incorrect: upward import**

```tsx
// ❌ src/features/search/ui/SearchBar.tsx
import { DashboardView } from '@/views/dashboard'  // Feature → View (upward)
```

```tsx
// ❌ src/entities/user/api/getUser.ts
import { LoginForm } from '@/features/auth'  // Entity → Feature (upward)
```

```tsx
// ❌ src/shared/lib/format.ts
import { User } from '@/entities/user'  // Shared → Entity (shared cannot import FSD layers)
```

**Correct: downward or same-level-only imports**

```tsx
// ✅ src/views/dashboard/ui/DashboardView.tsx
import { Header } from '@/widgets/header'       // View → Widget
import { StatsCard } from '@/features/analytics'  // View → Feature
import { User } from '@/entities/user'        // View → Entity
```

```tsx
// ✅ src/features/auth/ui/LoginForm.tsx
import { User } from '@/entities/user'        // Feature → Entity
import { Button } from '@/shared/ui/button'    // Feature → Shared
```

### 1.2 No Cross-Slice Imports Within Same Layer

**Impact: CRITICAL (prevents tight coupling between slices)**

Slices within the same layer cannot import from each other. Extract to a lower layer or compose in a higher layer.

**Incorrect: feature importing feature**

```tsx
// ❌ src/features/search/ui/SearchBar.tsx
import { LoginForm } from '@/features/auth'  // features/search → features/auth (same layer)
```

**Correct: extract to widget that imports both**

```tsx
// ✅ src/widgets/navbar/ui/Navbar.tsx
import { SearchBar } from '@/features/search'
import { LoginForm } from '@/features/auth'

export function Navbar() {
  return (
    <nav>
      <SearchBar />
      <LoginForm />
    </nav>
  )
}
```

---

## 2. Public API Pattern

**Impact: HIGH**

Every slice exposes a controlled public API through `index.ts`. This enables safe refactoring and clear module boundaries.

### 2.1 Export Through index.ts

**Impact: HIGH (enables refactoring, hides internals)**

Each slice must export through `index.ts`. Use explicit named exports; avoid `export * from`.

**Incorrect: no index.ts or export * from**

```tsx
// ❌ No index.ts — consumers don't know what's public
// consumers import: import { LoginForm } from '@/features/auth/ui/LoginForm'

// ❌ index.ts
export * from './ui/LoginForm'  // Exposes everything, hurts discoverability
export * from './model/useAuth'
```

**Correct: explicit exports in index.ts**

```tsx
// ✅ src/features/auth/index.ts
export { LoginForm } from './ui/LoginForm'
export { useAuth } from './model/useAuth'
export type { LoginCredentials, AuthState } from './model/types'
// Internal helpers NOT exported
```

### 2.2 Avoid Deep Imports

**Impact: HIGH (breaks when refactoring internals)**

Consumers must import from the slice root, never from internal paths.

**Incorrect: deep import**

```tsx
// ❌ Breaks when file moves or renames
import { LoginForm } from '@/features/auth/ui/LoginForm'
```

**Correct: public API import**

```tsx
// ✅ Stable — only index.ts exports matter
import { LoginForm } from '@/features/auth'
```

---

## 3. Views vs App Separation

**Impact: HIGH**

This skill uses **views** instead of FSD's standard **pages**. Routing stays in `src/app/`, page business logic in `src/views/`.

### 3.1 Routing in app, Logic in views

**Impact: HIGH (clear separation of routing vs composition)**

`src/app/` contains Next.js routing files only. Page composition and business logic live in `src/views/`.

**Incorrect: business logic in page.tsx**

```tsx
// ❌ src/app/dashboard/page.tsx — too much logic
export default function DashboardPage() {
  const user = useAuth()
  const stats = useDashboardStats()
  return (
    <div>
      <Header user={user} />
      <Sidebar />
      <main>
        <StatsCard stats={stats} />
        <RecentActivity userId={user.id} />
      </main>
    </div>
  )
}
```

**Correct: page delegates to view**

```tsx
// ✅ src/app/dashboard/page.tsx — routing only
import { DashboardView } from '@/views/dashboard'
import { getCurrentUser } from '@/entities/user'

export default async function DashboardPage() {
  const user = await getCurrentUser()
  return <DashboardView user={user} />
}
```

```tsx
// ✅ src/views/dashboard/ui/DashboardView.tsx — composition & logic
import { Header } from '@/widgets/header'
import { Sidebar } from '@/widgets/sidebar'
import { StatsCard } from '@/features/analytics'

export function DashboardView({ user }: { user: User }) {
  return (
    <div>
      <Header user={user} />
      <Sidebar />
      <main><StatsCard userId={user.id} /></main>
    </div>
  )
}
```

### 3.2 page.tsx Imports from views Only

**Impact: MEDIUM (consistent pattern)**

`page.tsx` should import view components from `@/views/`, not directly from widgets or features.

**Incorrect: page importing from widgets/features**

```tsx
// ❌ src/app/dashboard/page.tsx
import { Header } from '@/widgets/header'
import { StatsCard } from '@/features/analytics'
// Bypasses views — composition logic scattered
```

**Correct: page imports view**

```tsx
// ✅ src/app/dashboard/page.tsx
import { DashboardView } from '@/views/dashboard'
export default function DashboardPage() {
  return <DashboardView />
}
```

---

## 4. Segment Structure

**Impact: MEDIUM**

Slices use purpose-based segments: `ui/`, `model/`, `api/`, `lib/`, `config/`. Shared layer has no slices.

### 4.1 Use Purpose-Based Segments

**Impact: MEDIUM (consistent structure across slices)**

| Segment | Purpose | Use for |
|---------|---------|---------|
| ui/ | React components | Components, visual elements |
| model/ | Business logic & state | Types, hooks, stores |
| api/ | Data fetching | HTTP, WebSocket, Server actions |
| lib/ | Slice utilities | Validation, helpers (one focus per lib) |
| config/ | Constants | Feature flags, config objects |

**Incorrect: generic or wrong segment names**

```
features/auth/
├── components/   # ❌ Use ui/
├── hooks/        # ❌ Use model/ for hooks
└── utils/        # ❌ Use lib/
```

**Correct: purpose-based segments**

```
features/auth/
├── ui/
│   ├── LoginForm.tsx
│   └── SignupForm.tsx
├── model/
│   ├── useAuth.ts
│   └── types.ts
├── api/
│   └── login.ts
└── index.ts
```

### 4.2 Naming Conventions

**Impact: MEDIUM (consistency aids discovery)**

| Target | Convention | Example |
|--------|------------|---------|
| Folders, slices | kebab-case | `user-profile/`, `theme-toggle/` |
| Component files | kebab-case | `login-form.tsx`, `user-card.tsx` |
| Hook/store files | camelCase | `useAuth.ts`, `authStore.ts` |
| API/function files | camelCase | `userApi.ts`, `formatDate.ts` |

---

## 5. Resolving Circular Dependencies

**Impact: HIGH**

When two slices need each other, extract shared logic or introduce a higher layer.

### 5.1 Extract to Lower Layer

**Impact: HIGH**

Move shared logic to entities or shared. Both slices import from the lower layer.

**Incorrect: circular feature imports**

```tsx
// features/auth imports features/user-settings
// features/user-settings imports features/auth
// ❌ Circular dependency
```

**Correct: extract to entity**

```tsx
// entities/user — shared user logic
// Both features/auth and features/user-settings import from entities/user
// ✅ No circular dependency
```

### 5.2 Extract to Higher Layer (Widget)

**Impact: HIGH**

Create a widget that imports both features. Widget composes them; features stay independent.

**Correct: widget composes both**

```tsx
// ✅ src/widgets/user-panel/ui/UserPanel.tsx
import { UserProfile } from '@/features/user-profile'
import { AuthButton } from '@/features/auth'

export function UserPanel() {
  return (
    <div>
      <UserProfile />
      <AuthButton />
    </div>
  )
}
```

---

## References

**Skill references:**

- [layers-and-segments.md](./references/layers-and-segments.md) — Layers, slices, segments, migration
- [public-api.md](./references/public-api.md) — Public API, @x, circular imports, shared/ui
- [code-smells.md](./references/code-smells.md) — Desegmentation, anti-patterns
- [examples.md](./references/examples.md) — Auth, types, API patterns
- [react-query.md](./references/react-query.md) — React Query + FSD

**External:**

1. [FSD Documentation](https://feature-sliced.design/)
2. [FSD Layers (v2.1)](https://feature-sliced.design/docs/reference/layers)
3. [FSD Public API](https://feature-sliced.design/docs/reference/public-api)
4. [FSD with Next.js](https://feature-sliced.design/docs/guides/tech/with-nextjs)
5. [FSD with React Query](https://feature-sliced.design/docs/guides/tech/with-react-query)
6. [FSD Examples](https://github.com/feature-sliced/examples)
