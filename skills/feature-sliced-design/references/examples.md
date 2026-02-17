# FSD Code Examples

Full code examples for Feature-Sliced Design with Next.js. See [SKILL.md](../SKILL.md) for overview.

> **Next.js + FSD:** Routing in `src/app/`, page logic in `src/views/`, FSD App layer merged with Next.js App Router.

**Adapted from:** [Auth](https://feature-sliced.design/docs/guides/examples/auth) · [Types](https://feature-sliced.design/docs/guides/examples/types) · [Page layouts](https://feature-sliced.design/docs/guides/examples/page-layout) · [API requests](https://feature-sliced.design/docs/guides/examples/api-requests)

## Standalone Next.js Structure

```
my-nextjs-app/
├── src/
│   ├── app/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── (auth)/
│   │   │   ├── login/page.tsx
│   │   │   └── signup/page.tsx
│   │   ├── dashboard/page.tsx
│   │   ├── api/users/route.ts
│   │   ├── providers/
│   │   ├── styles/globals.css
│   │   └── config/
│   ├── views/
│   ├── widgets/
│   ├── features/
│   ├── entities/
│   └── shared/
│       ├── ui/
│       ├── lib/
│       ├── api/
│       └── config/
├── public/
└── package.json
```

## app Layer

```typescript
// src/app/providers/Providers.tsx
'use client';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from '@/shared/api/queryClient';
import { AuthProvider } from '@/features/auth';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      <AuthProvider>{children}</AuthProvider>
    </QueryClientProvider>
  );
}

// src/app/layout.tsx
import { Providers } from '@/app/providers/Providers';
import '@/app/styles/globals.css';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body><Providers>{children}</Providers></body>
    </html>
  );
}
```

## views Layer

```typescript
// src/views/dashboard/ui/DashboardView.tsx
import { Header } from '@/widgets/header';
import { Sidebar } from '@/widgets/sidebar';
import { StatsCard } from '@/features/analytics';
import { User } from '@/entities/user';

export function DashboardView({ user }: { user: User }) {
  return (
    <div className="dashboard">
      <Header user={user} />
      <div className="dashboard-content">
        <Sidebar />
        <main><StatsCard userId={user.id} /></main>
      </div>
    </div>
  );
}

// src/views/dashboard/index.ts
export { DashboardView } from './ui/DashboardView';

// src/app/dashboard/page.tsx
import { DashboardView } from '@/views/dashboard';
import { getCurrentUser } from '@/entities/user';

export default async function DashboardPage() {
  const user = await getCurrentUser();
  return <DashboardView user={user} />;
}
```

## widgets Layer

```typescript
// src/widgets/header/ui/Header.tsx
import { SearchBar } from '@/features/search';
import { UserMenu } from '@/features/user-menu';
import { User } from '@/entities/user';
import { Logo } from '@/shared/ui/logo';

export function Header({ user }: { user: User }) {
  return (
    <header className="header">
      <Logo />
      <SearchBar />
      <div className="header-actions"><UserMenu user={user} /></div>
    </header>
  );
}

// src/widgets/header/index.ts
export { Header } from './ui/Header';
```

## features Layer

```typescript
// src/features/auth/model/types.ts
export interface LoginCredentials { email: string; password: string; }

// src/features/auth/api/login.ts
import { User } from '@/entities/user';
import { apiClient } from '@/shared/api/client';
import type { LoginCredentials } from '../model/types';

export async function login(credentials: LoginCredentials): Promise<User> {
  const response = await apiClient.post('/auth/login', credentials);
  return response.data;
}

// src/features/auth/ui/LoginForm.tsx
'use client';
import { useState } from 'react';
import { login } from '../api/login';
import { Button } from '@/shared/ui/button';
import { Input } from '@/shared/ui/input';

export function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    await login({ email, password });
  };
  return (
    <form onSubmit={handleSubmit}>
      <Input type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="Email" />
      <Input type="password" value={password} onChange={(e) => setPassword(e.target.value)} placeholder="Password" />
      <Button type="submit">Login</Button>
    </form>
  );
}

// src/features/auth/index.ts
export { LoginForm } from './ui/LoginForm';
export { login } from './api/login';
export type { LoginCredentials } from './model/types';
```

## entities Layer

```typescript
// src/entities/user/model/types.ts
export interface User {
  id: string;
  name: string;
  email: string;
  avatar?: string;
  role: 'admin' | 'user';
}

// src/entities/user/api/getUser.ts
import { apiClient } from '@/shared/api/client';
import type { User } from '../model/types';

export async function getUser(id: string): Promise<User> {
  const response = await apiClient.get(`/users/${id}`);
  return response.data;
}

export async function getCurrentUser(): Promise<User> {
  const response = await apiClient.get('/users/me');
  return response.data;
}

// src/entities/user/ui/UserCard.tsx
import type { User } from '../model/types';
import { Avatar } from '@/shared/ui/avatar';

export function UserCard({ user }: { user: User }) {
  return (
    <div className="user-card">
      <Avatar src={user.avatar} alt={user.name} />
      <div><h3>{user.name}</h3><p>{user.email}</p></div>
    </div>
  );
}

// src/entities/user/index.ts
export { UserCard } from './ui/UserCard';
export { getUser, getCurrentUser } from './api/getUser';
export type { User } from './model/types';
```

## shared Layer

> **shared/ui structure:** Use per-component index for tree-shaking. See [public-api.md](./public-api.md).

```typescript
// src/shared/ui/button/Button.tsx
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
}

export function Button({ variant = 'primary', size = 'md', className, children, ...props }: ButtonProps) {
  return <button className={`button button--${variant} button--${size} ${className}`} {...props}>{children}</button>;
}

// src/shared/lib/format.ts
export function formatDate(date: Date): string {
  return new Intl.DateTimeFormat('en-US').format(date);
}

// src/shared/api/client.ts
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  headers: { 'Content-Type': 'application/json' },
});

// src/shared/api/endpoints/login.ts
import { apiClient } from '../client';

export interface LoginCredentials { email: string; password: string; }

export function login(credentials: LoginCredentials) {
  return apiClient.post('/auth/login', credentials);
}

// src/shared/api/index.ts — export client + endpoints; avoid one big barrel for shared/ui
export { apiClient } from './client';
export { login } from './endpoints/login';
export type { LoginCredentials } from './endpoints/login';
```

**shared/ui:** Use per-component index (`shared/ui/button/index.ts`, `shared/ui/text-field/index.ts`) for tree-shaking. See [public-api.md](./public-api.md).

## Segment Examples

### model/ — useAuth store

```typescript
// src/features/auth/model/useAuth.ts
'use client';
import { create } from 'zustand';
import type { User } from '@/entities/user';

export const useAuth = create<{ user: User | null; login: (u: User) => void; logout: () => void }>((set) => ({
  user: null,
  login: (user) => set({ user }),
  logout: () => set({ user: null }),
}));
```

### lib/ — validation

```typescript
// src/features/auth/lib/validation.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

export function validateLogin(data: unknown) {
  return loginSchema.parse(data);
}
```

### Form with react-hook-form

```typescript
// src/features/product-form/ui/ProductForm.tsx
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { productSchema } from '../lib/validation';
import { createProduct } from '../api/createProduct';

export function ProductForm() {
  const { register, handleSubmit } = useForm({
    resolver: zodResolver(productSchema),
  });
  return <form onSubmit={handleSubmit(createProduct)}>...</form>;
}
```

### Data Fetching with Server Components

```typescript
// src/app/dashboard/page.tsx
import { DashboardView } from '@/views/dashboard';
import { getUser } from '@/entities/user';

export default async function DashboardPage() {
  const user = await getUser('current');
  return <DashboardView user={user} />;
}

// src/views/dashboard/ui/DashboardView.tsx
import type { User } from '@/entities/user';

export function DashboardView({ user }: { user: User }) {
  return <div>Welcome, {user.name}</div>;
}
```

---

## Page Layouts

Shared layout for multiple pages. Source: [Page layouts](https://feature-sliced.design/docs/guides/examples/page-layout)

### Simple layout (shared/ui or app/layouts)

When layout has no business logic—header, sidebars, footer—put in `shared/ui` or `app/layouts`. Pass slots via props.

```typescript
// src/app/layouts/MainLayout.tsx (or shared/ui/layout)
import Link from "next/link";

export function MainLayout({
  children,
  siblingPages,
  headings,
}: {
  children: React.ReactNode;
  siblingPages?: Array<{ slug: string; title: string }>;
  headings?: Array<{ id: string; text: string }>;
}) {
  return (
    <div>
      <header>
        <nav><Link href="/">Home</Link><Link href="/docs">Docs</Link></nav>
      </header>
      <main>
        {siblingPages && <Sidebar items={siblingPages} />}
        {children}
        {headings && <Toc items={headings} />}
      </main>
      <footer>...</footer>
    </div>
  );
}

// app/(docs)/layout.tsx — Next.js route group
import { MainLayout } from "@/app/layouts/MainLayout";

export default function DocsLayout({ children }: { children: React.ReactNode }) {
  return <MainLayout siblingPages={...} headings={...}>{children}</MainLayout>;
}
```

### Layout with widgets

If the layout needs widgets (business logic), you cannot put it in Shared (Shared cannot import widgets).

**Alternatives:**
1. **Inline in app** — Use Next.js route groups. Apply layout only to grouped routes.
2. **Copy-paste** — Layouts rarely change. Duplicate across a few pages; add a comment if they should stay in sync.
3. **Render props / slots** — Pass widget as `children` or a slot prop from the page.
4. **app/layouts** — Put layout in `app/layouts` and compose widgets there (app can import widgets).

```typescript
// app/layouts/DashboardLayout.tsx
import { Header } from "@/widgets/header";
import { Sidebar } from "@/widgets/sidebar";

export function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div>
      <Header />
      <main><Sidebar />{children}</main>
    </div>
  );
}
```

---

## Authentication Patterns

Auth flow: 1) Collect credentials → 2) Call backend → 3) Store token.  
Source: [Auth](https://feature-sliced.design/docs/guides/examples/auth)

**OAuth:** If only OAuth, create a login page with a link to the provider; skip to step 3 (token storage).

### Option 1: Dedicated page (views/login)

Simple login/register on a dedicated route. Best when auth is page-level. No heavy decomposition needed.

```
views/
  login/
    ui/
      LoginView.tsx
      RegisterPage.tsx
    api/
      login.ts
    model/
      registration-schema.ts   # Zod for validation
    index.ts
```

### Option 2: Dialog widget (widgets/login-dialog)

Reusable login dialog used on any page. Avoid duplicating login logic across pages.

```
widgets/
  login-dialog/
    ui/
      LoginDialog.tsx
    api/
      login.ts
    index.ts
```

### Login function placement

| Use case | Location | Reason |
|----------|----------|--------|
| Global reuse | `shared/api/endpoints/login.ts` | Import from any slice |
| Login-only | `views/login/api/login.ts` | Keep logic in slice; no need to export in index |

Login can be called from: TanStack Query `useMutation`, Zustand/Redux side effect, or directly in component. Same applies to other state/mutation libraries.

### Client-side validation

Put schema in `model/` segment, use in `ui/`. Example with Zod:

```typescript
// views/login/model/registration-schema.ts
import { z } from "zod";

export const registrationSchema = z.object({
  email: z.string().email(),
  password: z.string().min(6),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords do not match",
  path: ["confirmPassword"],
});
```

### Two-factor auth (2FA)

- If `/login` returns `has2FA` flag, redirect to `/login/2fa`.
- 2FA page and APIs stay in `views/login` slice.
- `/2fa/verify` function goes in `shared/api` or `views/login/api`.

### Token storage

**Preferred: Cookie** — If the stack supports server-side cookies (e.g. Next.js server actions, Remix), use cookies. Frontend token handling is minimal. Server cookie logic in `shared/api`.

**When cookie is not possible:**

| Location | Pros | Cons |
|----------|------|------|
| **shared/api** | API client can read token; refresh via middleware | Token logic mixed with request logic |
| **shared/auth** | Split token management from `shared/api` | Separate segment to maintain |
| **entities/user** | Token + user in one store; clearer for auth logic | Need to expose token to API client |

**Token in shared/api:**  
1) Store token in module scope or reactive store.  
2) Refresh: on 401, call refresh endpoint, store new token, retry.  
3) If token logic grows, move to `shared/auth`.

**Token in entities/user:**  
Expose token to API client via:  
1) Pass token to each request (simple, verbose).  
2) Context / localStorage: key in `shared/api`, store from entity; provider in App layer.  
3) Subscription: push token into API client when it changes.

**Not in views/widgets** — Avoid app-wide token in page/widget slices.

```typescript
// entities/user/model/auth-store.ts
export const useAuthStore = create<{ token: string | null; user: User | null }>((set) => ({
  token: null,
  user: null,
  setAuth: (token, user) => set({ token, user }),
  logout: () => set({ token: null, user: null }),
}));
```

### Logout

1. Send authenticated logout request (e.g. `POST /logout`).
2. Reset token store.

**Placement:**
- All APIs in `shared/api` → put logout in `shared/api/endpoints/logout.ts` next to login.
- Logout button only in header → `widgets/header/api/logout.ts`. Request + store reset can live in that widget's `model/`.

### Automatic logout

Clear token when:
- Logout request fails.
- Refresh request fails.

Put this logic in `entities/user/model` or `shared/auth`.

---

## Types & DTOs

Source: [Types](https://feature-sliced.design/docs/guides/examples/types)

### Avoid shared/types and types segment

Avoid `shared/types` folder and `types` segment. "Types" describes essence (what), not purpose (why). Use purpose-based segments: `shared/api`, `shared/lib`, `model`, etc.

### Utility types

- **Project-wide:** `shared/lib/utility-types` with a README defining what belongs there.
- **Next to usage:** Don't overestimate reusability. Some utility types are fine beside the code that uses them.

```typescript
// shared/lib/utility-types (or type-fest)
type ArrayValues<T extends readonly unknown[]> = T[number];

// views/home/api/ArrayValues.ts — if only used here
type ArrayValues<T extends readonly unknown[]> = T[number];
```

### Business entities and cross-refs

Entities often reference each other (e.g. `Song` has `artists: Array<Artist>`). Two options:

1. **Parametrize:** `Song<ArtistType extends { id: string }>` — works for simple cases like `Cart = { items: Array<Product> }`.
2. **@x cross-import:** Use `entities/song/@x/artist.ts`. See [public-api.md](./public-api.md). Prefer this for tight coupling (e.g. Country/City).

### DTOs and mappers

- **DTOs** next to the request function. In `shared/api` if requests are there; in entity `api/` if requests are there.
- **Mappers** next to DTOs. Transform DTO → frontend type in the same place.

```typescript
// shared/api/endpoints/users.ts
interface UserDTO { id: number; name: string; avatar_url: string; }

function mapUser(dto: UserDTO): User {
  return { id: String(dto.id), name: dto.name, avatar: dto.avatar_url };
}

export async function getUser(id: string) {
  const dto = await apiClient.get<UserDTO>(`/users/${id}`);
  return mapUser(dto.data);
}
```

**Entity DTOs with @x:** When request is in entity and DTO references another entity, use `entities/artist/@x/song` for `ArtistDTO`.

### Nested DTOs

When backend returns multiple entities (e.g. song with full author objects), entities must know each other. Use explicit @x cross-imports, not indirect middleware. Consider normalizr for Redux-style normalization.

### Global types (e.g. Redux)

- **Generic (no app specifics):** `shared/analytics`, `shared/lib`, etc.
- **App-wide (RootState, AppDispatch):** App builds store; Shared needs typed hooks. Use `declare type RootState` / `declare type AppDispatch` globally so `shared/store` can use them without importing from App.

### Enums

Place as close to usage as possible. `ui` for UI-related (toast position), `api` for backend-related (loading state).

### Validation schemas (Zod / Valibot)

- **Backend response** → `api` segment, next to request. Fail request when schema fails.
- **Form input** → `model` or `ui` segment, next to form.

### Component props and context

Keep props/context interface in the same file as the component. If sharing across components, separate file in same folder (e.g. `ui/RecentActionsProps.ts`).

### Ambient declarations (*.d.ts)

- **Vite/ts-reset etc.:** `src/` root or `app/ambient/`.
- **Untyped packages:** `shared/lib/untyped-packages/%LIBRARY_NAME%.d.ts`.

```typescript
// shared/lib/untyped-packages/use-react-screenshot.d.ts
declare module "use-react-screenshot";
```

### Auto-generation

OpenAPI → generated types in `shared/api/openapi/`. Add README for regeneration.

---

## API Requests

Source: [Handling API Requests](https://feature-sliced.design/docs/guides/examples/api-requests)

### shared/api structure

```
shared/api/
  client.ts           # base URL, default headers, serialization
  index.ts
  endpoints/
    login.ts
    users.ts
```

`client.ts` centralizes HTTP setup: base URL, auth headers, JSON (de)serialization. Use axios or fetch wrapper.

### Slice-specific API

If a request is used only by one slice, put it in that slice's `api/` segment. No need to export in the slice's public API.

```typescript
// views/login/api/login.ts
import { client } from '@/shared/api';

export async function login(credentials: LoginCredentials) {
  return client.post('/auth/login', credentials);
}
```

> **Avoid putting API in entities too early.** Backend responses often differ from frontend entity shapes. Keep API in `shared/api` or slice `api/` to transform data; entities stay focused on frontend models.

### OpenAPI / orval / openapi-typescript

Generate types and client into `shared/api/openapi/`. Add README for regeneration.

### TanStack Query (and similar)

When using React Query, keep in `shared`: API types, cache keys, common query/mutation options. See [react-query.md](./react-query.md).

---

## Domain-based files (anti-desegmentation)

Avoid generic `types.ts`, `utils.ts`, `helpers.ts` that mix domains.

```typescript
// ❌ model/types.ts — mixed
export interface DeliveryOption { ... }
export interface UserInfo { ... }

// ✅ model/delivery.ts
export interface DeliveryOption { ... }

// ✅ model/user.ts
export interface UserInfo { ... }
```

See [code-smells.md](./code-smells.md).
