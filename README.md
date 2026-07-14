# activitylog

**ORM-agnostic entity audit trail for TypeScript**, with the DX of [spatie/laravel-activitylog](https://github.com/spatie/laravel-activitylog). The core knows nothing about any ORM or NestJS — adapters are first-class.

> **Status:** 1.0 — Core (manual logging, context, query API), NestJS module, TypeORM, and Prisma adapters are published. Drizzle adapter in preview.

## The bet

- **ORM-agnostic core + first-class adapters** — TypeORM, Prisma, Drizzle.
- **`iff-committed`** — an activity persists *if and only if* the mutation that caused it commits (when transactional). The audit trail never orphans or drops a record.
- **Causer resolved automatically** from request context (AsyncLocalStorage).
- **Honest coverage** — where a guarantee isn't possible (e.g. bulk/nested writes), it's declared in a coverage matrix, never faked.
- **Redaction on by default** — passwords, tokens and PII don't leak into the audit trail.

## Packages

| Package | Contents |
|---|---|
| `activitylog-core@1` | Agnostic core: logger, store, diff, context, query API |
| `activitylog-nestjs@1` | NestJS module + TypeORM adapter (subpath) |
| `activitylog-nextjs@1` | Prisma + Drizzle adapters (subpaths) |

## Quick start

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

## Contents

- [Getting Started](docs/getting-started)
- [Manual Logging](docs/manual-logging)
- [Context and Batches](docs/context-and-batches)
- [NestJS Integration](docs/nestjs-integration)
- [nestjs-cls Interoperability](docs/nestjs-cls-interop)
- [TypeORM Adapter](docs/typeorm-adapter)
- [Prisma Adapter](docs/prisma-adapter)
- [Querying Activities](docs/querying-activities)
- [Schema and Migrations](docs/schema-and-migrations)
- [Architecture Decisions](docs/architecture-decisions)
- [Deliberate Divergences from Spatie](docs/divergences-from-spatie)
- [Releasing](docs/releasing)

## License

[MIT](LICENSE)
