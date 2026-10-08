---
name: adding-components
description: Creates React components for the dashboard in apps/web. Use when building new UI components or reusable elements in apps/web/src/components.
---

# Adding Components

How to add React components to the dashboard (`apps/web`).

## File Location

Each component lives in its own **PascalCase folder** under `apps/web/src/components/`, with the implementation in `index.tsx`.

```
apps/web/src/components/
├── ProjectCard/
│   └── index.tsx
├── TaskList/
│   └── index.tsx
└── ui/                  # shadcn primitives: flat files, not folders
    ├── button.tsx
    └── dialog.tsx
```

Rules:

- Folder name is PascalCase and matches the component name (e.g. `ProjectCard/`).
- Implementation file is always `index.tsx` inside the folder.
- Import using the folder path; `index.tsx` is implicit:

  ```typescript
  import { ProjectCard } from '@/components/ProjectCard';
  import { TaskList, TaskListItem } from '@/components/TaskList';
  ```

- **Exception:** shadcn/ui primitives in `apps/web/src/components/ui/` stay as flat `kebab-case.tsx` files (e.g. `button.tsx`, `sheet.tsx`).
- Co-located helpers (sub-components, hooks, types) may live alongside `index.tsx` in the same folder when used only by that component.

## Component Pattern

```typescript
interface ProjectCardProps {
  name: string;
  description: string;
  onClick?: () => void;
}

export const ProjectCard = ({ name, description, onClick }: ProjectCardProps) => {
  return (
    <div className="rounded-lg border p-4" onClick={onClick}>
      <h3 className="font-semibold">{name}</h3>
      <p className="text-muted-foreground">{description}</p>
    </div>
  );
};
```

## Workflow

1. Create a new PascalCase folder in `apps/web/src/components/` (e.g. `ProjectCard/`)
2. Add `index.tsx` inside that folder
3. Define a TypeScript interface for props
4. Export the component as a named `const` arrow function
5. Import via the folder path: `import { ProjectCard } from '@/components/ProjectCard';`

## Key Patterns

### State & Data Fetching

- Use React hooks (`useState`, `useEffect`) for local state
- Fetch from the API through the Vite proxy (`/api/...`), using `fetchApi` from `@/lib/fetch-api` or a hook from `src/hooks/` ([[creating-hooks]])

### Styling

- Tailwind CSS utility classes
- Use `cn()` from `@/lib/utils` for conditional class merging:

```typescript
import { cn } from '@/lib/utils';

<div className={cn('p-4', isActive && 'bg-primary text-primary-foreground')} />
```

### Shared Types

Import domain types from the shared package:

```typescript
import type { Project, Task } from '@repo/shared';
import { API_ROUTE } from '@repo/shared';
```

## Checklist

- [ ] PascalCase folder under `apps/web/src/components/` with `index.tsx`
- [ ] Props interface defined
- [ ] Named `const` arrow export, no default export
- [ ] Conditional classes merged with `cn()`, domain types from `@repo/shared`
