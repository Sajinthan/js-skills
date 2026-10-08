---
name: adding-components
description: Creates React components for the Debrief clinical reflection app. Use when building new UI components, widgets, or reusable elements in apps/web/src/components.
---

# Adding Components

Guide for creating React components in the Debrief React SPA.

## File Location

Each component lives in its own **PascalCase folder** under `apps/web/src/components/`, with the implementation in `index.tsx`.

```
apps/web/src/components/
├── ChatBottomBar/
│   └── index.tsx
├── ConsultationPanel/
│   └── index.tsx
├── TextChatView/
│   └── index.tsx
└── ui/                  # shadcn primitives — flat files, not folders
    ├── button.tsx
    └── dialog.tsx
```

Rules:

- Folder name is PascalCase and matches the component name (e.g., `ChatBottomBar/`).
- Implementation file is always `index.tsx` inside the folder.
- Import using the folder path — the `index.tsx` is implicit:

  ```typescript
  import ChatBottomBar from '@/components/ChatBottomBar';
  import ConsultationPanel, { ConsultationPanelContent } from '@/components/ConsultationPanel';
  ```

- **Exception:** shadcn/ui primitives in `apps/web/src/components/ui/` stay as flat `kebab-case.tsx` files (e.g., `button.tsx`, `sheet.tsx`).
- Co-located helpers (sub-components, hooks, types) may live alongside `index.tsx` in the same folder when used only by that component.

## Component Pattern

```typescript
interface CaseCardProps {
  title: string;
  summary: string;
  onClick?: () => void;
}

const CaseCard = ({ title, summary, onClick }: CaseCardProps) => {
  return (
    <div className="rounded-lg border p-4" onClick={onClick}>
      <h3 className="font-semibold">{title}</h3>
      <p className="text-muted-foreground">{summary}</p>
    </div>
  );
};

export default CaseCard;
```

## Workflow

1. Create a new PascalCase folder in `apps/web/src/components/` (e.g., `MyWidget/`)
2. Add `index.tsx` inside that folder
3. Define a TypeScript interface for props
4. Export the component as default
5. Import via the folder path: `import MyWidget from '@/components/MyWidget';`

## Key Patterns

### State & Data Fetching

- Use React hooks (`useState`, `useEffect`) for local state
- Fetch from the API via the Vite proxy (`/api/...`)

### Styling

- Tailwind CSS utility classes
- Use `cn()` from `apps/web/src/lib/utils` for conditional class merging:

```typescript
import { cn } from "../lib/utils";

<div className={cn("p-4", isActive && "bg-primary text-primary-foreground")} />
```

### Shared Types

Import domain types from the shared package:

```typescript
import type { Consultation, ReflectRequest, ReflectResponse } from '@debrief/shared';
import { API_ROUTE } from '@debrief/shared';
```

### Key Component Patterns

**TextChatView** — Reflection chat transcript:

- GP messages right-aligned, AI messages left-aligned
- Renders an array of `ChatMessage` items

**ChatBottomBar** — Composer for chat input:

- Text input with send button
- Optional mic with auto-submit on silence (Web Speech API)
- Parent passes `startMicRef` to imperatively open the mic after AI audio ends

**ConsultationPanel** — De-identified consultation display:

- Card-based layout with consultation sections
- Uses `Consultation` type from `@debrief/shared`
- Also exports `ConsultationPanelContent` for embedding inside a `Sheet`

**VoiceModeView** / **WaveformVisualiser** — Voice-mode session UI:

- Pulse / waveform visualisation while AI is speaking or user is recording

**PreSessionScreen** — Pre-session loading & countdown
