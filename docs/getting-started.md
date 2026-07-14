---
title: Getting Started
category: getting-started
---

# Getting Started

## Installation

Install the packages you need:

```bash
npm install activitylog-core
npm install activitylog-nestjs   # for NestJS applications
npm install activitylog-nextjs   # for Prisma / Drizzle adapters
```

## Minimal setup (manual logging)

`activitylog-core` accepts an executor supplied by your database client or ORM. It does not load a database driver itself, so the same store works behind an ORM transaction and preserves the transaction boundary.

```ts
import { SqlExecutorStore, createActivityLogger, causerRef, subjectRef } from 'activitylog-core';

const store = new SqlExecutorStore({
  dataSource: {
    dialect: 'postgres',
    execute: async (sql, params) => pool.query(sql, params).then((r) => r.rows),
  },
});

const logger = createActivityLogger({ store });

await logger
  .activity('billing')
  .performedOn(subjectRef('Order', order.id))
  .causedBy(causerRef('User', userId))
  .withProperties({ plan: 'pro' })
  .event('subscribed')
  .log('Subscription created');
```

`createdAt` defaults to the application clock in UTC with millisecond precision. Use `.createdAt(date)` to override.

## Next steps

- [Manual Logging](manual-logging) — deeper API reference (aliases, hooks, redact, options)
- [Context and Batches](context-and-batches) — ALS, causer propagation, batch UUID
- [NestJS Integration](nestjs-integration) — set up the NestJS module
- [TypeORM Adapter](typeorm-adapter) — automatic entity auditing with `@LogsActivity()`
- [Prisma Adapter](prisma-adapter) — Prisma `$extends` client extension
