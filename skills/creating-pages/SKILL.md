---
name: creating-pages
description: Creates React pages and routes for the dashboard in apps/web (Vite + React Router). Use when adding new pages, views, or frontend routes in apps/web/src.
---

# Creating Pages

How to add pages to the dashboard (`apps/web`, Vite + React Router).

## Page Location

Each page lives in its own **PascalCase folder** under `apps/web/src/pages/`, with the implementation in `index.tsx`.

```
apps/web/src/pages/
├── Projects/
│   └── index.tsx
├── ProjectTasks/
│   └── index.tsx
├── Settings/
│   └── index.tsx
└── NotFound/
    └── index.tsx
```

Rules:

- Folder name is PascalCase and matches the page component name.
- Implementation file is always `index.tsx`.
- Import using the folder path; `index.tsx` is implicit:

  ```typescript
  import { Projects } from '@/pages/Projects';
  import { ProjectTasks } from '@/pages/ProjectTasks';
  ```

- Co-located helpers (sub-components, hooks, types) used only by the page may live alongside `index.tsx` in the same folder.

## Page File Pattern

```typescript
export const Projects = () => {
  return (
    <div>
      <h1>Projects</h1>
    </div>
  );
};
```

For pages with route params:

```typescript
import { useParams } from 'react-router';

export const ProjectOverview = () => {
  const { projectId } = useParams<{ projectId: string }>();

  return (
    <div>
      <h1>Project {projectId}</h1>
    </div>
  );
};
```

## Workflow

1. Create a new PascalCase folder in `apps/web/src/pages/` (e.g. `MyPage/`)
2. Add `index.tsx` inside that folder
3. Export the page as a named `const` arrow function
4. Register the route in `apps/web/src/routes.tsx` (inside `RequireSignIn` for dashboard pages; see Routing below)
5. Break complex UI into components under `apps/web/src/components/<Name>/index.tsx` ([[adding-components]])

## Key Patterns

### Routing

Routes live in `apps/web/src/routes.tsx`, written as JSX and turned into route objects with `createRoutesFromElements`. `main.tsx` runs them with `createBrowserRouter`, and the tests with `createMemoryRouter` (`test/renderApp.tsx`), so tests go through the real routes, sign-in checks included, and can read `router.state.location`.

- The dashboard has no public pages (the marketing site is `apps/site`), so routes need no `/app` prefix.
- `RedirectIfSignedIn` wraps sign-in, sign-up and reset; `RequireSignIn` wraps everything else. Both are layout routes: a `<Route element={…}>` with no path whose children render in its `<Outlet>`.
- Dashboard pages go under `<Route path="/" element={<AppShell />}>`; a project's tabs under `projects/:projectId` (`ProjectLayout`). Build links with the helpers in `config/routes.ts` (`projectPath(id, 'tasks')`), never by hand.

```tsx
// apps/web/src/routes.tsx
export const routes = createRoutesFromElements(
  <>
    <Route element={<RedirectIfSignedIn />}>
      <Route path="sign-in" element={<SignIn />} />
    </Route>
    <Route element={<RequireSignIn />}>
      <Route path="/" element={<AppShell />}>
        <Route index element={<Projects />} />
        <Route path="projects/:projectId" element={<ProjectLayout />}>
          <Route index element={<ProjectOverview />} />
          <Route path="tasks" element={<ProjectTasks />} />
        </Route>
        <Route path="*" element={<NotFound />} />
      </Route>
    </Route>
  </>,
);
```

### Data Fetching

Use the `fetchApi` helper from `@/lib/fetch-api` and the `API_ROUTE` constants from `@repo/shared` rather than hand-rolled `fetch('/api/...')` calls:

```typescript
import { API_ROUTE } from '@repo/shared';

import { fetchApi } from '@/lib/fetch-api';

const projects = await fetchApi(API_ROUTE.PROJECTS);
```

For repeated fetches, prefer the existing hooks in `apps/web/src/hooks/` (e.g. `useProjects`) or add one ([[creating-hooks]]).

### Styling

- Tailwind CSS utility classes
- `cn()` from `@/lib/utils` for conditional class merging

### Shared Types

Import domain types from the shared package:

```typescript
import type { Project, Task } from '@repo/shared';
import { API_ROUTE } from '@repo/shared';
```

## Checklist

- [ ] PascalCase folder under `apps/web/src/pages/` with `index.tsx`
- [ ] Named `const` arrow export, no default export
- [ ] Route registered in `routes.tsx` under the right layout route
- [ ] Links built with the `config/routes.ts` helpers
- [ ] Data through `fetchApi` + `API_ROUTE` or a hook
