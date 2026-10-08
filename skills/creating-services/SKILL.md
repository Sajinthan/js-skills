---
name: creating-services
description: Creates service modules in apps/<service>/src/services/ (apps/api, apps/worker and any other backend app) that wrap an external SDK or HTTP API behind a factory. Use when adding a cloud SDK integration, a third-party API client, or any external client a route or job calls.
---

# Creating Services

A service wraps one external SDK or HTTP API behind a small typed interface. Routes and jobs call the service, never the SDK. Database access is not a service: it goes through `@repo/db` ([[prisma-and-migrations]]).

## Where things go

| Concern | Location |
|---|---|
| Service | `apps/<service>/src/services/<name>.ts` |
| Tests | `apps/<service>/src/services/__tests__/<name>.test.ts` ([[testing]]) |
| Config | the app's env schema (`src/env.ts`, via `parseEnv` from `@repo/shared`) |
| Wiring | `src/server.ts` creates each service once and passes it in |
| Types that cross the API/web boundary | `@repo/shared` |

## Factory pattern

Each file exports an interface and a `create<Name>` factory. The factory builds the client once, defines each function as a named `const` typed from the interface, and returns them by name. Never write function bodies inside the returned object ([[coding-style]]). AWS S3 as the example:

```ts
import { GetObjectCommand, NoSuchKey, PutObjectCommand, S3Client } from '@aws-sdk/client-s3';

export interface FileStore {
  put: (key: string, text: string) => Promise<void>;
  get: (key: string) => Promise<string | null>;
}

export interface FileStoreOptions {
  bucket: string;
  region: string;
}

export const createFileStore = ({ bucket, region }: FileStoreOptions): FileStore => {
  const client = new S3Client({ region });

  const put: FileStore['put'] = async (key, text) => {
    await client.send(new PutObjectCommand({ Bucket: bucket, Key: key, Body: text }));
  };

  const get: FileStore['get'] = async key => {
    try {
      const object = await client.send(new GetObjectCommand({ Bucket: bucket, Key: key }));

      if (!object.Body) {
        throw new Error('S3 returned an object with no body');
      }

      return await object.Body.transformToString('utf-8');
    } catch (error) {
      if (error instanceof NoSuchKey) {
        return null;
      }

      throw error;
    }
  };

  return { put, get };
};
```

`server.ts` calls it once: `createFileStore({ bucket: env.FILES_BUCKET, region: env.AWS_REGION })`.

## Key conventions

- **One client per factory call**: create the SDK client inside the factory, never per call.
- **Config is passed in**: the factory takes an options object that `server.ts` fills from the env schema. A service never reads `process.env`, so tests pass their own options.
- **Named exports**: the factory and its interface; no class, no default export.
- **Throw on empty responses**: check the payload and throw a descriptive error when the provider returned nothing.
- **Map the errors you expect** (not found, throttled) to a value the caller can act on, such as `null`; rethrow the rest.
- **Types from shared**: use `@repo/shared` for domain types that cross the API/web boundary.

## Credentials (AWS example)

When a service assumes a role in another AWS account, pass a credentials provider to the client. A helper such as `credentialsForRole(roleArn)` in `src/lib/` returns `undefined` when no role is configured, so local dev falls back to the default credential chain:

```ts
const client = new S3Client({ region, credentials: credentialsForRole(roleArn) });
```

## Streaming pattern

For an SDK that streams output, the interface returns an `AsyncGenerator`, and the factory holds a generator function in a named `const`:

```ts
export interface TextStream {
  stream: (input: string) => AsyncGenerator<string, void, void>;
}

export const createTextStream = ({ region }: TextStreamOptions): TextStream => {
  const client = new MyClient({ region });

  const stream: TextStream['stream'] = async function* (input) {
    const response = await client.send(new MyStreamCommand({ input }));

    if (!response.stream) {
      throw new Error('Service returned no stream');
    }

    for await (const event of response.stream) {
      const delta = event.contentBlockDelta?.delta?.text;

      if (delta) {
        yield delta;
      }
    }
  };

  return { stream };
};
```

## HTTP API pattern

For a third-party HTTP API, the base URL and key are options, and the response is parsed with Zod at the edge:

```ts
export interface IssueTracker {
  getIssue: (id: string) => Promise<Issue>;
}

export const createIssueTracker = ({ baseUrl, apiKey }: IssueTrackerOptions): IssueTracker => {
  const getIssue: IssueTracker['getIssue'] = async id => {
    const res = await fetch(`${baseUrl}/issues/${encodeURIComponent(id)}`, {
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (!res.ok) {
      throw new Error(`Issue tracker API ${res.status}: ${await res.text()}`);
    }

    return issueSchema.parse(await res.json());
  };

  return { getIssue };
};
```

## Checklist

- [ ] File in `apps/<service>/src/services/`, test in `services/__tests__/`
- [ ] Exports an interface and a `create<Name>` factory; no class, no default export
- [ ] Client created once inside the factory; each function a named `const` returned by name
- [ ] Config passed in as options from the env schema; no `process.env` in the service
- [ ] Throws descriptive errors on empty or failed responses
- [ ] External responses parsed with Zod; shared types from `@repo/shared`
- [ ] Created once in `server.ts` and passed to whatever uses it
