# Public API Reference

FSD public API rules for Next.js + FSD. Source: [Public API](https://feature-sliced.design/docs/reference/public-api)

---

## What is a Public API?

A public API is a **contract** between a slice (or segment) and the code that uses it. It acts as a gate—only certain objects are exposed, and only through that public API.

In practice: an `index.ts` file with re-exports.

```typescript
// views/auth/index.ts
export { LoginView } from "./ui/LoginView";
export { SignupView } from "./ui/SignupView";
```

---

## Good Public API Goals

1. **Protect from structural changes** — Refactor slice internals without breaking consumers.
2. **Breaking behavior = change public API** — Significant behavior changes should update the contract.
3. **Expose only what's needed** — Don't leak internals.

---

## Avoid `export *`

```typescript
// ❌ BAD
export * from "./ui/Comment";
export * from "./model/comments";
```

- Hurts discoverability (can't tell the interface at a glance).
- May accidentally expose internals, making refactoring harder.

```typescript
// ✅ GOOD
export { CommentCard } from "./ui/CommentCard";
export type { Comment } from "./model/types";
```

---

## @x Cross-imports (Entities Layer Only)

When one entity references another (e.g., `Artist` has `songs: Array<Song>`), use the `@x` public API.

**Structure:**
```
entities/
  song/
    @x/
      artist.ts   — public API only for entities/artist
    index.ts     — regular public API
```

```typescript
// entities/song/@x/artist.ts
export type { Song } from "../model/song";

// entities/artist/model/artist.ts
import type { Song } from "@/entities/song/@x/artist";
export interface Artist { name: string; songs: Array<Song>; }
```

> Use `@x` minimally and **only on the Entities layer**.

---

## Circular Imports

When two modules in a slice need each other, prefer:

- **Same slice** → use **relative** imports with full path (no barrel via index).
- **Different slices** → use **absolute** imports (alias).

```typescript
// ❌ Circular: ui/LoginView.tsx imports from index
import { loadUser } from "@/views/auth";  // index re-exports ui/LoginView → circular

// ✅ Same slice: use relative, direct path
import { loadUser } from "../api/loadUser";
```

---

## shared/ui and shared/lib — Tree-shaking

A single `shared/ui/index.ts` that re-exports everything can break tree-shaking (heavy components pulled into every page).

**Recommended:** per-component index.

```
shared/ui/
  button/
    index.ts
  text-field/
    index.ts
  carousel/
    index.ts
```

```typescript
// ✅ Import directly
import { Button } from "@/shared/ui/button";
import { TextField } from "@/shared/ui/text-field";
```

Same for `shared/lib`: per-library folder with its own index.

---

## Shared Layer — Per-segment Public API

Shared has no slices. Define a **separate public API per segment** instead of one giant index.

```
shared/
  ui/
    button/index.ts
    text-field/index.ts
  api/
    index.ts
  lib/
    format/index.ts
```

---

## Barrel File Performance

Many index files can slow the dev server. Mitigations:

1. Separate index per component in `shared/ui` and `shared/lib`.
2. Avoid segment-level index inside slices (e.g. no `features/auth/ui/index.ts` if you have `features/auth/index.ts`).

---

## Enforcing Public API — Steiger

[Steiger](https://github.com/feature-sliced/steiger) is an FSD linter. It can catch direct imports that bypass the public API (e.g. auto-import choosing a deep path).

---

## See Also

- [layers-and-segments.md](./layers-and-segments.md) — @x example, segment patterns
- [code-smells.md](./code-smells.md) — Desegmentation, generic folders
- [FSD Public API](https://feature-sliced.design/docs/reference/public-api)
