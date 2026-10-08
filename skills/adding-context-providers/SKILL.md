---
name: adding-context-providers
description: Create or modify a React Context provider in the dashboard (apps/web). Use when adding global state (current user, theme, feature flags, etc.) that needs to flow through the component tree.
---

# Adding Context Providers

React Contexts in the dashboard follow a small folder-per-context layout under `apps/web/src/context/`, with the provider, the context object and the consumer hook split across files.

## Folder Layout

```text
apps/web/src/context/
└── CurrentUserContext/
    ├── index.tsx           # <CurrentUserProvider>: owns state + fetch logic
    └── useCurrentUser.ts   # createContext() + useCurrentUser() consumer hook
```

Rules:

- **Folder is PascalCase** (`CurrentUserContext/`, `ThemeContext/`).
- **`index.tsx`** exports the provider component (e.g. `CurrentUserProvider`); this is what `App.tsx` mounts.
- **`useXxx.ts`** exports the `Context` object and the `useXxx()` hook; components import the hook from here, never the raw context.
- Splitting the provider and the hook into two files keeps the provider's `useEffect` logic out of components that only consume the value, and avoids circular imports when the provider needs the context type.

## Pattern

```typescript
// apps/web/src/context/CurrentUserContext/useCurrentUser.ts
import { createContext, useContext } from 'react';

import type { User, UserUpdate } from '@repo/shared';

export interface CurrentUserContextValue {
  user: User | null;
  loading: boolean;
  error: string | null;
  updateUser: (data: UserUpdate) => Promise<void>;
}

export const CurrentUserContext = createContext<CurrentUserContextValue>({
  user: null,
  loading: true,
  error: null,
  updateUser: async () => {},
});

export const useCurrentUser = () => useContext(CurrentUserContext);
```

```typescript
// apps/web/src/context/CurrentUserContext/index.tsx
import { useEffect, useState } from 'react';

import { API_ROUTE, type User, type UserUpdate } from '@repo/shared';

import { fetchApi } from '@/lib/fetch-api';

import { CurrentUserContext } from './useCurrentUser';

export const CurrentUserProvider = ({ children }: { children: React.ReactNode }) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetchApi<User>(API_ROUTE.USERS_ME)
      .then(setUser)
      .catch((e: Error) => setError(e.message))
      .finally(() => setLoading(false));
  }, []);

  const updateUser = async (data: UserUpdate) => {
    const updated = await fetchApi<User>(API_ROUTE.USERS_ME, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });

    setUser(updated);
  };

  return (
    <CurrentUserContext.Provider value={{ user, loading, error, updateUser }}>
      {children}
    </CurrentUserContext.Provider>
  );
};
```

Consumers import only the hook:

```typescript
import { useCurrentUser } from '@/context/CurrentUserContext/useCurrentUser';

export const Settings = () => {
  const { user, updateUser } = useCurrentUser();
  // ...
};
```

## Default Context Value

Provide a sensible default in `createContext(...)` so the hook never returns `undefined` and components don't need a "did you forget the provider" check. Functions in the default value should be no-op `async () => {}` to keep the type signature.

## Mounting the Provider

Mount providers in `apps/web/src/App.tsx` above the routes. Order them outermost → innermost; data providers (e.g. `CurrentUserProvider`) typically wrap the router, while UI providers (e.g. `ThemeProvider`) sit closer to the root.

Signed-out routes should not assume an authenticated context. Gate authenticated trees with the `RequireSignIn` layout route ([[creating-pages]]) so the provider's `null` state is only seen on signed-out pages.

## What NOT to Do

- Don't put global state in `localStorage`. Auth is BFF-cookie based and the API is the source of truth; use a context backed by `fetchApi` instead.
- Don't introduce a context for component-local state. Promote to context only when 2 or more unrelated components need the same data.
- Don't talk to the auth provider directly from a context. Go through `@/lib/auth`, which calls the API (`/api/auth/*`).

## Checklist

- [ ] Folder under `apps/web/src/context/<PascalCaseName>/`
- [ ] `index.tsx` exports the `<XxxProvider>` component
- [ ] `useXxx.ts` exports the `Context` object and `useXxx()` hook
- [ ] Default value supplied to `createContext` so the hook is non-null
- [ ] Data fetching uses `fetchApi` + `API_ROUTE` (no hard-coded paths)
- [ ] Provider mounted in `App.tsx`
- [ ] Authenticated trees still gated by `RequireSignIn`
