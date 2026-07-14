---
title: Manual Logging
category: core
---

# Manual Logging

`activitylog-core` accepts an executor supplied by your database client or ORM. It does not load a database driver itself, so the same store works behind an ORM transaction and preserves the transaction boundary.

## Basic usage

```ts
const logger = createActivityLogger({ store });

await logger
  .activity('billing')
  .performedOn(subjectRef('Order', order.id))
  .causedBy(causerRef('User', userId))
  .withProperties({ plan: 'pro' })
  .event('subscribed')
  .log('Subscription created');
```

## Aliases

- `.on()` — alias for `.performedOn()`
- `.by()` — alias for `.causedBy()`
- `.byAnonymous()` — removes an explicit causer (falls back to context)

## Timestamps

`createdAt` defaults to the application clock in UTC with millisecond precision. Override with `.createdAt(date)` to supply a logical event time.

## Log options

`logOptions` redact nested sensitive property names before persistence. The default list includes passwords, tokens, secrets, authorization data, and email addresses. Override with a replacement list or `redact: false` only when that policy is explicitly appropriate for your application.

### beforePersist hook

A `beforePersist` hook can enrich an activity before redaction. Its output is also redacted.

```ts
const logger = createActivityLogger({
  store,
  logOptions: {
    beforePersist: (activity) => ({
      ...activity,
      properties: { ...activity.properties, enriched: true },
    }),
  },
});
```

## Transactional writes

Pass an executor bound to an existing ORM transaction as the second `persist` argument. The core never commits or rolls back a transaction itself.

```ts
await logger
  .activity('orders')
  .event('created')
  .log('Order created', transactionalExecutor);
```
