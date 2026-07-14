---
title: Getting Started
category: getting-started
---

# Getting Started

## Installation

Install the packages you need:

```bash
npm install activitylog-core
npm install activitylog-nestjs   # for NestJS apps
npm install activitylog-nextjs   # for Prisma / Drizzle adapters
```

## Quick start (manual logging)

```ts
import {
  SqlExecutorStore,
  createActivityLogger,
  causerRef,
  subjectRef,
} from 'activitylog-core';

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

## Next steps

- [Manual Logging](manual-logging) — deeper API reference
- [NestJS Integration](nestjs-integration) — set up the NestJS module
- [TypeORM Adapter](typeorm-adapter) — automatic entity auditing
- [Prisma Adapter](prisma-adapter) — Prisma client extension
