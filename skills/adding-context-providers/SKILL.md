---
name: adding-context-providers
description: Create or modify a React Context provider for the Debrief web app. Use when adding global state (current doctor, theme, feature flags, etc.) that needs to flow through the component tree.
---

# Adding Context Providers

React Contexts in Debrief follow a small folder-per-context layout under [apps/web/src/context/](file:///Users/sajinthanjanahiram/Projects/debrief-demo/apps/web/src/context/) with the provider, the context object, and the consumer hook split across files.

## Folder Layout

```text
apps/web/src/context/
└── DoctorContext/
    ├── index.tsx        # <DoctorProvider> — owns state + fetch logic
    └── useDoctor.ts     # createContext() + useDoctor() consumer hook
```

Rules:

- **Folder is PascalCase** (`DoctorContext/`, `ThemeContext/`).
- **`index.tsx`** exports the provider component (e.g., `DoctorProvider`) — this is what `App.tsx` mounts.
- **`useXxx.ts`** exports the `Context` object and the `useXxx()` hook — components import the hook from here, never the raw context.
- Splitting the provider and the hook into two files keeps the provider's `useEffect` logic out of components that only consume the value, and avoids circular imports when the provider needs the context type.

## Pattern

```typescript
// apps/web/src/context/DoctorContext/useDoctor.ts
import { createContext, useContext } from 'react';

import type { Doctor, DoctorUpdate } from '@debrief/shared';

export interface DoctorContextValue {
  doctor: Doctor | null;
  loading: boolean;
  error: string | null;
  updateDoctor: (data: DoctorUpdate) => Promise<void>;
}

export const DoctorContext = createContext<DoctorContextValue>({
  doctor: null,
  loading: true,
  error: null,
  updateDoctor: async () => {},
});

export const useDoctor = () => useContext(DoctorContext);
```

```typescript
// apps/web/src/context/DoctorContext/index.tsx
import { useEffect, useState } from 'react';

import { API_ROUTE, type Doctor, type DoctorUpdate } from '@debrief/shared';

import fetchApi from '@/lib/fetchApi';

import { DoctorContext } from './useDoctor';

export const DoctorProvider = ({ children }: { children: React.ReactNode }) => {
  const [doctor, setDoctor] = useState<Doctor | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    fetchApi<Doctor>(API_ROUTE.DOCTORS_ME)
      .then(setDoctor)
      .catch((e: Error) => setError(e.message))
      .finally(() => setLoading(false));
  }, []);

  const updateDoctor = async (data: DoctorUpdate) => {
    const updated = await fetchApi<Doctor>(API_ROUTE.DOCTORS_ME, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });

    setDoctor(updated);
  };

  return (
    <DoctorContext.Provider value={{ doctor, loading, error, updateDoctor }}>
      {children}
    </DoctorContext.Provider>
  );
};
```

Consumers import only the hook:

```typescript
import { useDoctor } from '@/context/DoctorContext/useDoctor';

const Settings = () => {
  const { doctor, updateDoctor } = useDoctor();
  // ...
};
```

## Default Context Value

Provide a sensible default in `createContext(...)` so the hook never returns `undefined` and components don't need a "did you forget the provider" check. Functions in the default value should be no-op `async () => {}` to keep the type signature.

## Mounting the Provider

Mount providers in [apps/web/src/App.tsx](file:///Users/sajinthanjanahiram/Projects/debrief-demo/apps/web/src/App.tsx) above the routes. Order them outermost → innermost; data providers (e.g. `DoctorProvider`) typically wrap the router, while UI providers (e.g. `ThemeProvider`) sit closer to the root.

Public-facing routes should not assume an authenticated context — gate authenticated trees with `RequireAuth` / `RequireDoctor` (see `apps/web/src/components/RequireDoctor/`) so the provider's `null` state is only seen on public pages.

## What NOT to Do

- Don't put global state in `localStorage` — auth is BFF-cookie based and the API is the source of truth. Use a context backed by `fetchApi` instead.
- Don't introduce a context for component-local state — promote to context only when ≥2 unrelated components need the same data.
- Don't talk to Cognito directly from a context — go through `apps/web/src/lib/auth.ts`, which calls the BFF (`/api/auth/*`).

## Checklist

- [ ] Folder under `apps/web/src/context/<PascalCaseName>/`
- [ ] `index.tsx` exports the `<XxxProvider>` component
- [ ] `useXxx.ts` exports the `Context` object and `useXxx()` hook
- [ ] Default value supplied to `createContext` so the hook is non-null
- [ ] Data fetching uses `fetchApi` + `API_ROUTE` (no hard-coded paths)
- [ ] Provider mounted in `App.tsx`
- [ ] Authenticated trees still gated by `RequireAuth` / `RequireDoctor`
