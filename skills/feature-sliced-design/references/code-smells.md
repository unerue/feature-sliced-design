# FSD Code Smells & Anti-patterns

Common pitfalls when using FSD with Next.js. Source: [Desegmentation](https://feature-sliced.design/docs/guides/issues/desegmented)

---

## Desegmentation (Horizontal Slicing)

**Desegmentation** = grouping by **technical role** instead of **business domain**.

### Anti-pattern: Generic folders

```
❌ BAD — grouped by type, not domain
features/delivery/
  ui/
    components/     ← generic
      DeliveryCard.tsx
      DeliveryChoice.tsx
      UserAvatar.tsx
  model/
    utils.ts        ← generic
entities/recommendations/
  utils/            ← generic
```

**Problems:** Low cohesion, tight coupling, harder refactoring.

### Solution: Group by domain

```
✅ GOOD — domain-based files
views/delivery/
  ui/
    DeliveryView.tsx
    DeliveryCard.tsx
    DeliveryChoice.tsx
    UserInfo.tsx
  model/
    delivery.ts     ← domain-specific
    user.ts         ← domain-specific
  api/
    delivery.ts
    user.ts
```

---

## Generic File Names

Avoid files that aggregate multiple domains:

- `types.ts` — mixes Delivery, User, etc.
- `utils.ts` — mixed helpers
- `helpers.ts`
- `endpoints.ts`

**Instead:** Name by domain: `delivery.ts`, `user.ts`, `delivery-api.ts`.

```typescript
// ❌ pages/delivery/model/types.ts — mixed domains
export interface DeliveryOption { id: string; name: string; price: number; }
export interface UserInfo { id: string; name: string; avatar: string; }

// ✅ pages/delivery/model/delivery.ts
export interface DeliveryOption { id: string; name: string; price: number; }

// ✅ pages/delivery/model/user.ts
export interface UserInfo { id: string; name: string; avatar: string; }
```

---

## Segment Names — Purpose, Not Essence

| ❌ Bad (essence) | ✅ Good (purpose) |
|------------------|-------------------|
| `components` | `ui` |
| `hooks` | `model` (for state hooks) |
| `types` | `model` (for types) |
| `utils` | `lib` |
| `modals` | `ui` |

Segment names should describe **why** the code exists, not **what** it is.

---

## shared Layer Anti-patterns

- **`shared/types`** — groups unrelated things by "being a type". Use `shared/api`, `shared/lib`, etc., or domain-specific folders.
- **`shared/components`** — use `shared/ui`.
- **`shared/hooks`** — use purpose-based segments (e.g. `shared/lib`, or put in the slice that needs them).

---

## See Also

- [layers-and-segments.md](./layers-and-segments.md) — Segment patterns
- [public-api.md](./public-api.md) — Export rules
- [FSD Desegmentation](https://feature-sliced.design/docs/guides/issues/desegmented)
