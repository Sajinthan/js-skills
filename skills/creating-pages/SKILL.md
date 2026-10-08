---
name: creating-pages
description: Creates React pages/views for the Debrief clinical reflection app. Use when adding new pages, views, or frontend routes in apps/web/src.
---

# Creating Pages

Guide for adding new pages in the Debrief React SPA (Vite + React Router).

## Page Location

Each page lives in its own **PascalCase folder** under `apps/web/src/pages/`, with the implementation in `index.tsx`.

```
apps/web/src/pages/
├── Dashboard/
│   └── index.tsx
├── Landing/
│   └── index.tsx
├── ActiveReflection/
│   └── index.tsx
├── ReflectionSession/
│   └── index.tsx
├── ReflectionSummary/
│   └── index.tsx
├── ReflectionHistory/
│   └── index.tsx
├── Settings/
│   └── index.tsx
└── NotFound/
    └── index.tsx
```

Rules:

- Folder name is PascalCase and matches the page component name.
- Implementation file is always `index.tsx`.
- Import using the folder path — `index.tsx` is implicit:

  ```typescript
  import Dashboard from '@/pages/Dashboard';
  import ActiveReflection from '@/pages/ActiveReflection';
  ```

- Co-located helpers (sub-components, hooks, types) used only by the page may live alongside `index.tsx` in the same folder.

## Page File Pattern

```typescript
const Dashboard = () => {
  return (
    <div>
      <h1>Dashboard</h1>
    </div>
  );
};

export default Dashboard;
```

For pages with route params:

```typescript
import { useParams } from "react-router-dom";

const SessionDetail = () => {
  const { id } = useParams<{ id: string }>();

  return (
    <div>
      <h1>Session {id}</h1>
    </div>
  );
};

export default SessionDetail;
```

## Workflow

1. Create a new PascalCase folder in `apps/web/src/pages/` (e.g., `MyPage/`)
2. Add `index.tsx` inside that folder
3. Export the page component as default
4. Register the route in `apps/web/src/routes.tsx` (inside `RequireSignIn` for dashboard pages; see Routing below)
5. Break complex UI into components under `apps/web/src/components/<Name>/index.tsx`

## Key Patterns

### Routing

_Updated 2026-09-29 for Goodo; the rest of this skill still describes Debrief and needs a [[refine-skill]] pass._

Routes live in `apps/web/src/routes.tsx`, written as JSX like debrief-demo's `App.tsx` and turned into route objects with `createRoutesFromElements` (`react-router` v8). `main.tsx` runs them with `createBrowserRouter`, and the tests with `createMemoryRouter` (`test/renderApp.tsx`), so tests go through the real routes, sign-in checks included, and can read `router.state.location`.

- The dashboard has its own domain, so there's no `/app` prefix and no public site here (the site is `apps/site`).
- `RedirectIfSignedIn` wraps sign-in, sign-up and reset; `RequireSignIn` wraps everything else. Both are layout routes: a `<Route element={…}>` with no path whose children render in its `<Outlet>`.
- Dashboard pages go under `<Route path="/" element={<AppShell />}>`; an assistant's tabs under `assistants/:assistantId` (`AssistantLayout`). Build links with the helpers in `config/routes.ts` (`assistantPath(id, 'knowledge')`), never by hand.

```tsx
// apps/web/src/routes.tsx
export const routes = createRoutesFromElements(
  <>
    <Route element={<RequireSignIn />}>
      <Route path="/" element={<AppShell />}>
        <Route index element={<Start />} />
        <Route path="assistants/:assistantId" element={<AssistantLayout />}>
          <Route index element={<AssistantOverview />} />
          <Route path="knowledge" element={<Knowledge />} />
        </Route>
        <Route path="*" element={<NotFound />} />
      </Route>
    </Route>
  </>,
);
```

### Data Fetching

Use the `fetchApi` helper from `@/lib/fetchApi` and the `API_ROUTE` constants from `@debrief/shared` rather than hand-rolled `fetch('/api/...')` calls:

```typescript
import fetchApi from '@/lib/fetchApi';
import { API_ROUTE } from '@debrief/shared';

const sessions = await fetchApi(API_ROUTE.DOCTORS_ME_SESSIONS);
```

For repeated fetches, prefer the existing hooks in `apps/web/src/hooks/` (e.g., `useDoctorStats`, `useReflectionSessions`).

### Styling

- Tailwind CSS utility classes
- `cn()` from `@/lib/utils` for conditional class merging

### Shared Types

Import domain types from the shared package:

```typescript
import type { Consultation, ReflectRequest, ReflectResponse } from '@debrief/shared';
import { API_ROUTE } from '@debrief/shared';
```

### Reflection Session Flow

The main user flow across pages:

1. **Landing** (`/`) — Public marketing page
2. **Dashboard** (`/app`) — View past sessions, click "Start Session"
3. **ReflectionSession** (`/app/session`) — Pick / load a randomly selected consultation, click "Start Reflection"
4. **ActiveReflection** (`/app/session/:sessionId`) — Socratic AI conversation (text + optional voice)
5. **ReflectionSummary** (`/app/summary/:sessionId`) — Themes, domains, areas for reflection
6. **ReflectionHistory** (`/app/history`) — Browse all past sessions
