---
name: creating-hooks
description: Creates custom React hooks for Debrief. Use when building data-fetching hooks, streaming hooks, or browser-API hooks in apps/web/src/hooks/.
---

# Creating Hooks

Custom React hooks for the Debrief web app.

## Directory Structure

```text
apps/web/src/hooks/
└── useMyHook.ts            # Flat file, one hook per file
```

## Data-Fetching Hook Pattern

The most common pattern — fetches from the API on mount and returns `{ data, loading, error }`:

```typescript
import { useEffect, useState } from 'react';

import type { MyType } from '@debrief/shared';
import { API_ROUTE } from '@debrief/shared';

import fetchApi from '@/lib/fetchApi';

interface UseMyDataResult {
  data: MyType | null;
  loading: boolean;
  error: string | null;
}

const useMyData = (): UseMyDataResult => {
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

export default useMyData;
```

For array results, initialise with `[]` instead of `null` and type accordingly:

```typescript
const [items, setItems] = useState<MyItem[]>([]);
```

## Polling Hook Pattern

For data that arrives asynchronously after session start (e.g. background precompute):

```typescript
import { useCallback, useEffect, useState } from 'react';

import type { MyType } from '@debrief/shared';
import { API_ROUTE } from '@debrief/shared';

import fetchApi from '@/lib/fetchApi';

const POLL_INTERVAL_MS = 1500;
const POLL_MAX_DURATION_MS = 90000;

interface UsePolledDataResult {
  data: MyType[];
  loading: boolean;
  error: string | null;
  refetch: () => void;
}

const usePolledData = (id: string | undefined): UsePolledDataResult => {
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

export default usePolledData;
```

## Streaming Hook Pattern

For NDJSON streaming endpoints (voice/reflect). Returns a `useCallback` that the caller invokes per turn:

```typescript
import { useCallback } from 'react';

import type { ReflectRequest } from '@debrief/shared';
import { API_ROUTE } from '@debrief/shared';

export type StreamResult = { content: string; shouldWrapUp: boolean };

const useMyStream = (sessionId: string | undefined) =>
  useCallback(
    async (
      messages: { role: string; content: string }[],
      opts: { onText: (delta: string) => void; signal?: AbortSignal },
    ): Promise<StreamResult | null> => {
      if (!sessionId) {
        return null;
      }

      const { onText, signal } = opts;
      const body = JSON.stringify({ sessionId, messages } satisfies ReflectRequest & { sessionId: string });

      const res = await fetch(API_ROUTE.VOICE_REFLECT, {
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
      let shouldWrapUp = false;

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

          let evt: { type: string; delta?: string; content?: string; shouldWrapUp?: boolean; message?: string };

          try {
            evt = JSON.parse(line) as typeof evt;
          } catch {
            continue;
          }

          if (evt.type === 'text' && evt.delta) {
            onText(evt.delta);
          } else if (evt.type === 'done') {
            content = evt.content ?? '';
            shouldWrapUp = Boolean(evt.shouldWrapUp);
          } else if (evt.type === 'error') {
            throw new Error(evt.message ?? 'stream error');
          }
        }
      }

      return { content, shouldWrapUp };
    },
    [sessionId],
  );

export default useMyStream;
```

## Conventions

- **File naming**: `useMyHook.ts` (camelCase, flat file preferred)
- **Export**: `export default useMyHook` for single-hook files
- **Result interface**: Define an explicit `UseMyHookResult` interface for data-fetching hooks
- **Route constants**: Always use `API_ROUTE` from `@debrief/shared` — never hard-code paths
- **Fetch helper**: Use `fetchApi` from `@/lib/fetchApi` for standard requests (handles credentials, JSON parsing, error throwing)
- **Streaming**: Use raw `fetch` with `credentials: 'include'` for NDJSON streaming endpoints
- **Cleanup**: Always return cleanup functions from useEffect (clear timers, set `cancelled` flags, remove listeners)
- **AbortSignal**: Accept `signal?: AbortSignal` for cancellable async operations
- **Guard clauses**: Return early if required params (like `sessionId`) are undefined

## Checklist

- [ ] Hook file in `apps/web/src/hooks/useXxx.ts`
- [ ] Uses `API_ROUTE` from `@debrief/shared` for endpoints
- [ ] Uses `fetchApi` for standard requests
- [ ] Explicit result interface for data-fetching hooks
- [ ] Proper cleanup (cancelled flags, clearTimeout, removeEventListener)
- [ ] Loading state defaults to `true` for fetch-on-mount hooks
- [ ] Error state typed as `string | null`
- [ ] Default export
- [ ] Blank line before `if` and `return` statements
