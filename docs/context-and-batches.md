---
title: Context and Batches
category: core
---

# Context and Batches

## Request context

Use `runWithContext` around a request or job to supply a causer. It flows through promises and timers via AsyncLocalStorage. The logger uses the context causer only when `.causedBy()` was omitted.

```ts
import { runWithContext } from 'activitylog-core';

runWithContext({ causer: causerRef('User', '123') }, async () => {
  // All activities inside here inherit the causer
  await logger.activity('default').event('viewed').log('Page viewed');
});
```

## Batches

`withBatch` assigns one UUID to a unit of work. Nested calls reuse that UUID.

```ts
import { withBatch } from 'activitylog-core';

await withBatch(async () => {
  // All activities share the same batch_uuid
  await logger.activity('billing').event('subscribed').log('Plan upgrade');
  await logger.activity('billing').event('invoice_created').log('Invoice generated');
});
```

## Queue boundary

For a queue boundary, pass `serializeContext()` with the job payload and restore it with `runWithContext()` in the worker.

```ts
// Producer
const serialized = serializeContext();
await queue.add({ ...payload, _activitylog: serialized });

// Worker
runWithContext(serialized, async () => {
  await processJob();
});
```

## Suppression

- `withoutLogging(fn)` — suppresses activity logging inside `fn`
- `disableLogging()` / `enableLogging()` — global toggle
