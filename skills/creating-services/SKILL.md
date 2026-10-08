---
name: creating-services
description: Creates AWS SDK service wrappers in apps/api/src/services/. Use when adding new AWS integrations or external API clients.
---

# Creating Services

Service files wrap AWS SDK clients and external APIs for the Debrief backend.

## Directory

```text
apps/api/src/services/
├── bedrock.ts          # Claude Converse + streaming
├── polly.ts            # Text-to-speech synthesis
├── deidentify.ts       # Comprehend Medical DetectPHI
├── embeddings.ts       # Titan text embeddings
├── knowledge.ts        # pgvector similarity search
├── haloClient.ts       # External API (Halo PMS)
├── cognitoAuth.ts      # Cognito user management
└── email/              # SES email (folder for templates + sender)
```

## Standard Pattern

Each service file follows this structure:

```typescript
import { MyClient, MyCommand } from '@aws-sdk/client-my-service';

import { credentialsForRole } from '../lib/awsCreds.js';

const client = new MyClient({
  credentials: credentialsForRole('MY_ASSUME_ROLE_ARN'),
});

export const doSomething = async (input: string): Promise<string> => {
  const result = await client.send(
    new MyCommand({ /* params */ }),
  );

  if (!result.Output) {
    throw new Error('MyService returned no output');
  }

  return result.Output;
};
```

## Key Conventions

- **Module-level client** — instantiate the SDK client once at the top of the file, not per-call
- **`credentialsForRole`** — use `../lib/awsCreds.js` for cross-account assume-role. Returns `undefined` when the env var is unset (local dev uses default chain)
- **Named exports** — export individual functions, not a class or default
- **Throw on empty responses** — check the response payload and throw a descriptive error if the service returned nothing
- **Config from env** — read model IDs, voice IDs, regions from `process.env` with sensible defaults via `??`
- **Type imports from shared** — use `@debrief/shared` for domain types that cross the API/web boundary

## Credentials Pattern

```typescript
import { credentialsForRole } from '../lib/awsCreds.js';

// Returns undefined in local dev (falls back to default credential chain)
// Returns an STS assume-role provider in Lambda (env var set by Terraform)
const client = new MyClient({
  credentials: credentialsForRole('MY_ASSUME_ROLE_ARN'),
});
```

Valid env var names: `BEDROCK_ASSUME_ROLE_ARN`, `POLLY_ASSUME_ROLE_ARN`, `SES_ASSUME_ROLE_ARN`.

## Streaming Pattern

For services that produce streaming output (e.g. Bedrock ConverseStream):

```typescript
export async function* myStream(opts: MyOpts): AsyncGenerator<string, void, void> {
  const response = await client.send(new MyStreamCommand({ /* params */ }));

  if (!response.stream) {
    throw new Error('Service returned no stream');
  }

  for await (const event of response.stream) {
    const delta = event.contentBlockDelta?.delta?.text;

    if (delta) {
      yield delta;
    }
  }
}
```

## External API Pattern (non-AWS)

For third-party HTTP APIs:

```typescript
const BASE_URL = process.env.HALO_API_URL ?? 'https://api.example.com';

export const fetchSomething = async (id: string): Promise<MyType> => {
  const res = await fetch(`${BASE_URL}/resource/${id}`, {
    headers: { Authorization: `Bearer ${process.env.HALO_API_KEY}` },
  });

  if (!res.ok) {
    throw new Error(`Halo API ${res.status}: ${await res.text()}`);
  }

  return res.json() as Promise<MyType>;
};
```

## Checklist

- [ ] File in `apps/api/src/services/` (camelCase)
- [ ] Module-level client instantiation
- [ ] Uses `credentialsForRole` for AWS services needing cross-account access
- [ ] Config values from `process.env` with `??` defaults
- [ ] Named function exports (not default, not class)
- [ ] Throws descriptive errors on empty/failed responses
- [ ] Types imported from `@debrief/shared` where applicable
- [ ] `.js` extension on relative imports (ESM)
