# FSD Code Examples

Full code examples for Feature-Sliced Design. See [SKILL.md](../SKILL.md) for overview.

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
```

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
