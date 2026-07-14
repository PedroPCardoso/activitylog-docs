---
title: Manual Logging
category: core
---

# Manual Logging

The fluent API is the primary way to log activities directly.

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
- `.byAnonymous()` — removes an explicit causer, falling back to context resolution

## Timestamps

`createdAt` defaults to the application clock in UTC with millisecond precision.

```ts
await logger
  .activity('orders')
  .event('created')
  .createdAt(new Date('2025-01-01T00:00:00.000Z'))
  .log('Order created');
```

## Log options

`logOptions` control redaction, hooks, and diff filtering. Redaction is deny-by-default: sensitive field names (passwords, tokens, secrets, authorization data, email addresses) are masked before persistence.

```ts
const logger = createActivityLogger({
  store,
  logOptions: {
    redact: ['password', 'token', 'secret'],  // default list; override explicitly
    dontSubmitEmptyLogs: true,                 // skip persist when diff is empty
  },
});
```

Pass `redact: false` only when that policy is explicitly appropriate for your application.

### beforePersist hook

A `beforePersist` hook can enrich an activity. Its output is also redacted — redaction is the final guard.

```ts
const logger = createActivityLogger({
  store,
  logOptions: {
    beforePersist: (activity, context) => {
      return { ...activity, properties: { ...activity.properties, enriched: true } };
    },
  },
});
```

### descriptionForEvent

```ts
const logger = createActivityLogger({
  store,
  logOptions: {
    descriptionForEvent: ({ event, subject, diff }) => {
      if (event === 'created') return `New ${subject.type} created`;
      return `${subject.type} ${event}`;
    },
  },
});
```

## Transactional writes

Pass an executor bound to an existing database transaction as the second `persist` argument. The core never commits or rolls back a transaction itself.

```ts
await logger
  .activity('orders')
  .event('created')
  .log('Order created', transactionalExecutor);
```
