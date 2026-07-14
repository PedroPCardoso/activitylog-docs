---
title: Context and Batches
category: core
---

# Context and Batches

## Request context

Use `runWithContext` around a request or job to supply a causer. It flows through promises and timers via AsyncLocalStorage. The logger resolves the context causer when `.causedBy()` was omitted.

```ts
import { runWithContext, causerRef } from 'activitylog-core';

runWithContext({ causer: causerRef('User', '123') }, async () => {
  await logger.activity('default').event('viewed').log('Page viewed');
  await innerTask(); // still inside the same context
});
```

## Causer resolution precedence

1. Explicit `.causedBy(ref)` on the builder
2. Context from `runWithContext`
3. Resolver configured in the store/logger options
4. `null` (system causer)

## Batches

`withBatch` assigns one UUID to a unit of work. Nested calls reuse that UUID.

```ts
import { withBatch } from 'activitylog-core';

await withBatch(async () => {
  await logger.activity('billing').event('subscribed').log('Plan upgrade');
  await logger.activity('billing').event('invoice_created').log('Invoice generated');
  // Both share the same batch_uuid
});
```

## Queue boundary

AsyncLocalStorage does not cross queue boundaries automatically. Pass `serializeContext()` with the job payload and restore it with `runWithContext()` in the worker.

```ts
// Producer
const serialized = serializeContext();
await queue.add({ ...payload, _activitylog: serialized });

// Worker
import { runWithContext } from 'activitylog-core';
runWithContext(serialized, async () => {
  await processJob();
});
```

A worker that receives no context logs a null/system causer.

## Suppression

- `withoutLogging(fn)` — suppresses activity logging inside `fn`
- `disableLogging()` / `enableLogging()` — global toggle
