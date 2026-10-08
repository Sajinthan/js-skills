---
name: creating-hooks
description: Creates custom React hooks for the dashboard in apps/web. Use when building data-fetching hooks, polling hooks, streaming hooks, or browser-API hooks in apps/web/src/hooks/.
---

# Creating Hooks

Custom React hooks for the dashboard (`apps/web`).

## Directory Structure

```text
apps/web/src/hooks/
└── useMyHook.ts            # Flat file, one hook per file
```

## Data-Fetching Hook Pattern

The most common pattern — fetches from the API on mount and returns `{ data, loading, error }`:

```typescript
import { useEffect, useState } from 'react';

import type { MyType } from '@repo/shared';
import { API_ROUTE } from '@repo/shared';

import { fetchApi } from '@/lib/fetch-api';

interface UseMyDataResult {
  data: MyType | null;
  loading: boolean;
  error: string | null;
}

export const useMyData = (): UseMyDataResult => {
  const [data, setData] = useState<MyType | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const controller = new AbortController();

    fetchApi<MyType>(API_ROUTE.MY_ENDPOINT, { signal: controller.signal })
      .then(setData)
      .catch((e: Error) => {
        if (controller.signal.aborted) {
          return;
        }
        setError(e.message);
      })
      .finally(() => {
        if (!controller.signal.aborted) {
          setLoading(false);
        }
      });

    return () => controller.abort();
  }, []);

  return { data, loading, error };
};
```

For array results, initialise with `[]` instead of `null` and type accordingly:

```typescript
const [items, setItems] = useState<MyItem[]>([]);
```

## Polling Hook Pattern

For data that arrives asynchronously after the first request (e.g. the output of a background job):

```typescript
import { useCallback, useEffect, useState } from 'react';

import type { MyType } from '@repo/shared';
import { API_ROUTE } from '@repo/shared';

import { fetchApi } from '@/lib/fetch-api';

const POLL_INTERVAL_MS = 1500;
const POLL_MAX_DURATION_MS = 90000;

interface UsePolledDataResult {
  data: MyType[];
  loading: boolean;
  error: string | null;
  refetch: () => void;
}

export const usePolledData = (id: string | undefined): UsePolledDataResult => {
  const [data, setData] = useState<MyType[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [tick, setTick] = useState(0);

  const refetch = useCallback(() => {
    setError(null);
    setLoading(true);
    setTick(t => t + 1);
  }, []);

  useEffect(() => {
    if (!id) {
      return;
    }

    let cancelled = false;
    let pollTimer: ReturnType<typeof setTimeout> | null = null;
    const startedAt = Date.now();

    const fetchOnce = async (): Promise<void> => {
      try {
        const result = await fetchApi<MyType[]>(API_ROUTE.MY_ENDPOINT.replace(':id', id));

        if (cancelled) {
          return;
        }
        setData(result);

        const shouldKeepPolling = result.length === 0 && Date.now() - startedAt < POLL_MAX_DURATION_MS;

        if (shouldKeepPolling) {
          pollTimer = setTimeout(fetchOnce, POLL_INTERVAL_MS);
        } else {
          setLoading(false);
        }
      } catch (e) {
        if (cancelled) {
          return;
        }
        setData([]);
        setError((e as Error).message);
        setLoading(false);
      }
    };

    void fetchOnce();

    return () => {
      cancelled = true;

      if (pollTimer) {
        clearTimeout(pollTimer);
      }
    };
  }, [id, tick]);

  return { data, loading, error, refetch };
};
```

## Streaming Hook Pattern

For NDJSON streaming endpoints (e.g. an AI-generated project summary streamed as text). Returns a `useCallback` that the caller invokes per request:

```typescript
import { useCallback } from 'react';

import type { SummaryRequest } from '@repo/shared';
import { API_ROUTE } from '@repo/shared';

export type StreamResult = { content: string };

export const useSummaryStream = (projectId: string | undefined) =>
  useCallback(
    async (
      prompt: string,
      opts: { onText: (delta: string) => void; signal?: AbortSignal },
    ): Promise<StreamResult | null> => {
      if (!projectId) {
        return null;
      }

      const { onText, signal } = opts;
      const body = JSON.stringify({ prompt } satisfies SummaryRequest);

      const res = await fetch(API_ROUTE.PROJECT_SUMMARY_STREAM.replace(':projectId', projectId), {
        method: 'POST',
        credentials: 'include',
        headers: { 'content-type': 'application/json' },
        body,
        signal,
      });

      if (!res.ok || !res.body) {
        throw new Error(`stream http ${res.status}`);
      }

      const reader = res.body.getReader();
      const decoder = new TextDecoder();
      let buffered = '';
      let content = '';

      while (true) {
        if (signal?.aborted) {
          break;
        }

        const { value, done } = await reader.read();

        if (done) {
          break;
        }

        buffered += decoder.decode(value, { stream: true });

        let nl: number;

        while ((nl = buffered.indexOf('\n')) >= 0) {
          const line = buffered.slice(0, nl).trim();
          buffered = buffered.slice(nl + 1);

          if (!line) {
            continue;
          }

          let evt: { type: string; delta?: string; content?: string; message?: string };

          try {
            evt = JSON.parse(line) as typeof evt;
          } catch {
            continue;
          }

          if (evt.type === 'text' && evt.delta) {
            onText(evt.delta);
          } else if (evt.type === 'done') {
            content = evt.content ?? '';
          } else if (evt.type === 'error') {
            throw new Error(evt.message ?? 'stream error');
          }
        }
      }

      return { content };
    },
    [projectId],
  );
```

## Conventions

- **File naming**: `useMyHook.ts` (camelCase, flat file, one hook per file)
- **Export**: named export, `export const useMyHook = …`
- **Result interface**: Define an explicit `UseMyHookResult` interface for data-fetching hooks
- **Route constants**: Always use `API_ROUTE` from `@repo/shared`; never hard-code paths
- **Fetch helper**: Use `fetchApi` from `@/lib/fetch-api` for standard requests (handles credentials, JSON parsing, error throwing)
- **Streaming**: Use raw `fetch` with `credentials: 'include'` for NDJSON streaming endpoints
- **Cleanup**: Always return cleanup functions from `useEffect` (clear timers, set `cancelled` flags, remove listeners)
- **AbortSignal**: Accept `signal?: AbortSignal` for cancellable async operations
- **Guard clauses**: Return early if required params (like `projectId`) are undefined

## Checklist

- [ ] Hook file in `apps/web/src/hooks/useXxx.ts`
- [ ] Uses `API_ROUTE` from `@repo/shared` for endpoints
- [ ] Uses `fetchApi` for standard requests
- [ ] Explicit result interface for data-fetching hooks
- [ ] Proper cleanup (cancelled flags, clearTimeout, removeEventListener)
- [ ] Loading state defaults to `true` for fetch-on-mount hooks
- [ ] Error state typed as `string | null`
- [ ] Named export
- [ ] Blank line before `if` and `return` statements
