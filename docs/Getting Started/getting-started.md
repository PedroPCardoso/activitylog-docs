---
title: Complete Guide — activitylog
excerpt: >-
  ORM-agnostic entity audit trail for TypeScript, with the DX of
  spatie/laravel-activitylog. Core, NestJS, TypeORM and Prisma adapters.
hidden: false
---
> **ORM-agnostic entity audit trail for TypeScript**, with the DX of
> [spatie/laravel-activitylog](https://github.com/spatie/laravel-activitylog).  
> The core knows nothing about any ORM or NestJS — adapters are first-class.

---

## Table of Contents

- [Installation](#installation)
- [Manual Logging](#manual-logging)
- [Context and Batches](#context-and-batches)
- [NestJS Integration](#nestjs-integration)
- [nestjs-cls Interoperability](#nestjs-cls-interoperability)
- [TypeORM Adapter](#typeorm-adapter)
- [Prisma Adapter](#prisma-adapter)
- [Querying Activities](#querying-activities)
- [Schema and Migrations](#schema-and-migrations)
- [Deliberate Divergences from Spatie](#deliberate-divergences-from-spatie)
- [Releasing](#releasing)
- [API Architecture](#api-architecture)
- [Error Reference](#error-reference)

---

## Installation

```bash
npm install activitylog-core
npm install activitylog-nestjs   # for NestJS applications
npm install activitylog-nextjs   # for Prisma / Drizzle adapters
```

Peer dependencies depend on the adapter:
- `@nestjs/common` + `@nestjs/core` for `activitylog-nestjs`
- `typeorm` for `activitylog-nestjs/typeorm`
- `@prisma/client` >=7.8 for `activitylog-nextjs/prisma`

---

## Manual Logging

`activitylog-core` accepts an executor supplied by your database client or ORM. It does not load a database driver itself, so the same store works behind an ORM transaction.

### Basic usage

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

### Aliases

- `.on()` — alias for `.performedOn()`
- `.by()` — alias for `.causedBy()`
- `.byAnonymous()` — removes an explicit causer, falling back to context

### Timestamps

`createdAt` defaults to the application clock in UTC with millisecond precision.

```ts
await logger
  .activity('orders')
  .event('created')
  .createdAt(new Date('2025-01-01T00:00:00.000Z'))
  .log('Order created');
```

### Log options

`logOptions` control redaction, hooks, and diff filtering. Redaction is **deny-by-default**: sensitive field names (passwords, tokens, secrets, authorization data, email addresses) are masked before persistence.

```ts
const logger = createActivityLogger({
  store,
  logOptions: {
    redact: ['password', 'token', 'secret'], // default list; override explicitly
    dontSubmitEmptyLogs: true,                // skip persist when diff is empty
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

### Transactional writes

Pass an executor bound to an existing database transaction as the second `persist` argument. The core never commits or rolls back a transaction itself.

```ts
await logger
  .activity('orders')
  .event('created')
  .log('Order created', transactionalExecutor);
```

---

## Context and Batches

### Request context

Use `runWithContext` around a request or job to supply a causer. It flows through promises and timers via AsyncLocalStorage.

```ts
import { runWithContext, causerRef } from 'activitylog-core';

runWithContext({ causer: causerRef('User', '123') }, async () => {
  await logger.activity('default').event('viewed').log('Page viewed');
});
```

### Causer resolution precedence

1. Explicit `.causedBy(ref)` on the builder
2. Context from `runWithContext`
3. Resolver configured in store options
4. `null` (system causer)

### Batches

`withBatch` assigns one UUID to a unit of work. Nested calls reuse that UUID.

```ts
import { withBatch } from 'activitylog-core';

await withBatch(async () => {
  await logger.activity('billing').event('subscribed').log('Plan upgrade');
  await logger.activity('billing').event('invoice_created').log('Invoice generated');
});
```

### Queue boundary

AsyncLocalStorage does not cross queue boundaries. Pass `serializeContext()` with the job payload and restore it in the worker.

```ts
// Producer
const serialized = serializeContext();
await queue.add({ ...payload, _activitylog: serialized });

// Worker
runWithContext(serialized, async () => {
  await processJob();
});
```

A worker that receives no context logs a null/system causer.

### Suppression

- `withoutLogging(fn)` — suppresses activity logging inside `fn`
- `disableLogging()` / `enableLogging()` — global toggle

---

## NestJS Integration

### Global setup

`activitylog-nestjs` exposes a global module and an injectable facade.

```ts
import { Module } from '@nestjs/common';
import { ActivityLogModule } from 'activitylog-nestjs';

@Module({
  imports: [ActivityLogModule.forRoot({ store })],
})
export class AppModule {}
```

`forRoot()` applies the request middleware once. The middleware opens AsyncLocalStorage before guards run, while the logger resolves `request.user` lazily after a guard has populated it.

### Usage in services

```ts
import { Injectable } from '@nestjs/common';
import { ActivityLogService } from 'activitylog-nestjs';

@Injectable()
export class OrdersService {
  constructor(private readonly activityLog: ActivityLogService) {}

  async record(): Promise<void> {
    await this.activityLog
      .activity('orders')
      .event('updated')
      .log('Order updated');
  }
}
```

### Causer resolution

A plain `request.user` with an `id` becomes a `User` causer. Pass `causerResolver` to `forRoot()` when the application uses a different identity shape:

```ts
ActivityLogModule.forRoot({
  store,
  causerResolver: (request) =>
    request.user
      ? { type: 'Account', id: request.user.accountId }
      : null,
});
```

### Feature modules

```ts
import { ActivityLogModule } from 'activitylog-nestjs';

@Module({
  imports: [ActivityLogModule.forFeature({
    logOnly: ['name', 'status'],
  })],
})
export class BillingModule {}
```

### Option precedence

**Call > Feature > Root > Defaults**

```ts
// call-site overrides
.activity('billing', { logOnly: ['name'] })

// forFeature options
ActivityLogModule.forFeature({ logOnly: ['name', 'status'] })

// forRoot options
ActivityLogModule.forRoot({ store, logOnly: ['name', 'status'] })

// built-in defaults
```

Merge is shallow; arrays are replaced, not merged.

### Interceptor alternative

```ts
import { ActivityLogInterceptor } from 'activitylog-nestjs';

@Controller()
@UseInterceptors(ActivityLogInterceptor)
export class OrdersController {}
```

Use either middleware (recommended) or the interceptor as the HTTP context boundary.

---

## nestjs-cls Interoperability

`activitylog` owns its AsyncLocalStorage; an application already using `nestjs-cls` has two independent ALS stores. Use `nestjs-cls` as the source of truth and read it from activitylog's lazy `causerResolver`.

### Configuration

```ts
import { Module } from '@nestjs/common';
import { ClsModule, ClsServiceManager, type ClsStore } from 'nestjs-cls';
import { ActivityLogModule, causerRef } from 'activitylog-nestjs';

interface AppClsStore extends ClsStore {
  user?: { id: string };
}

@Module({
  imports: [
    ClsModule.forRoot({ global: true, middleware: { mount: true } }),
    ActivityLogModule.forRoot({
      store,
      causerResolver: () => {
        const user = ClsServiceManager.getClsService<AppClsStore>().get('user');
        return user ? causerRef('User', user.id) : null;
      },
    }),
  ],
})
export class AppModule {}
```

### Context nesting

```
ClsMiddleware.run(request)
  └─ ActivityLogMiddleware.run(request)
       └─ guards → controller → services → activity().log()
```

### Queue boundary

Materialize the causer when creating a job payload:

```ts
const cls = ClsServiceManager.getClsService<AppClsStore>();
const user = cls.get('user');
const activitylogContext = runWithContext(
  { causer: user ? causerRef('User', user.id) : null },
  () => serializeContext(),
);
```

---

## TypeORM Adapter

The `activitylog-nestjs/typeorm` subpath audits entity lifecycle operations through a TypeORM subscriber.

### Entity decorator

```ts
import { Column, Entity, PrimaryGeneratedColumn } from 'typeorm';
import { LogsActivity } from 'activitylog-nestjs/typeorm';

@LogsActivity({
  logOnly: ['name', 'status'],
  logOnlyDirty: true,
})
@Entity()
export class Order {
  @PrimaryGeneratedColumn()
  id!: number;

  @Column()
  name!: string;

  @Column()
  status!: string;
}
```

### Register the subscriber

```ts
import { registerActivityLogSubscriber } from 'activitylog-nestjs/typeorm';

await dataSource.initialize();
registerActivityLogSubscriber(dataSource, { store });
```

### Diffs

`repository.save()` creates `created` or `updated` activities. `repository.remove()` and `repository.softRemove()` create `deleted` activities.

```ts
{
  attributes: { status: 'paid' },
  old: { status: 'pending' },
}
```

Created activities have empty `old`; hard-deleted activities have empty `attributes`. Composite primary keys are outside the 1.0 contract.

### auditedUpdate

TypeORM does not provide a reliable old entity for `Repository.update()` or update QueryBuilder calls.

```ts
import { auditedUpdate } from 'activitylog-nestjs/typeorm';

await auditedUpdate(
  dataSource.getRepository(Order),
  { id: orderId },
  { status: 'paid' },
);
```

The helper opens or nests a transaction, reads the row, updates, re-reads, diffs, and persists before committing. Use `lockForDiff: true` for pessimistic locks under concurrent writers.

### Coverage matrix

| TypeORM operation | Automatic | Supported path |
|---|---|---|
| `save()` insert | ✅ | `@LogsActivity()` + subscriber |
| `save()` update | ✅ | `@LogsActivity()` + subscriber |
| `remove()` | ✅ | `@LogsActivity()` + subscriber |
| `softRemove()` | ✅ | `@LogsActivity()` + subscriber |
| `repository.update()` | ⚠️ | `auditedUpdate()` |
| update QueryBuilder | ⚠️ | `auditedUpdate()` |
| bulk update | ❌ | explicit Aggregate activity |
| entity-less cascade | ❌ | explicit manual activity |

---

## Prisma Adapter

The Prisma adapter lives under `activitylog-nextjs/prisma` and provides two modes: a `$extends` client extension (best-effort) and `auditedTransaction` (iff-committed).

### Setup

```ts
import { prismaActivityLog } from 'activitylog-nextjs/prisma';

const options = {
  dialect: 'sqlite' as const,
  models: {
    User: { idField: 'id', relationFields: ['posts'] },
  },
};

const activityPrisma = prismaActivityLog(prisma, options);
await activityPrisma.user.update({ where: { id: 1 }, data: { name: 'Updated' } });
```

### Options

```ts
type PrismaActivityLogOptions = {
  dialect: 'sqlite' | 'postgres' | 'mysql';
  models?: Record<string, {
    idField?: string;               // default "id"
    relationFields?: readonly string[];
    auditFields?: readonly string[];
  }>;
  lockForDiff?: false;
  tableName?: string;
};
```

Or use a custom store:

```ts
{
  store: myCustomStore;
  storeTransactionMode?: 'none' | 'uses-context';
}
```

### auditedTransaction

```ts
import { auditedTransaction } from 'activitylog-nextjs/prisma';

await auditedTransaction(prisma, options, async (tx) => {
  await tx.user.update({ where: { id: 1 }, data: { name: 'Atomic' } });
});
```

The helper opens one Prisma interactive transaction and passes a manual audited proxy over that exact `tx` client. An exception rolls back both mutation and Activity.

### Individual operations

| Operation | Event | Diff source |
|---|---|---|
| `create` | `created` | `attributes` from returned record |
| `update` | `updated` | `old` from pre-read, `attributes` from returned record |
| `delete` | `deleted` | `old` from pre-read |
| `upsert` | `created` or `updated` | pre-read determines branch |

### Bulk operations

Bulk operations emit one Aggregate activity per top-level call:

```ts
{
  aggregate: true,
  criteria: args.where ?? {},
  changes: args.data,
  affected: result.count,
}
```

Aggregate activities use `subject_type = model` and `subject_id = null`.

### Nested writes

Nested writes are recorded as one Aggregate activity for the top-level model. Detection uses the `relationFields` declared in model config.

### Transaction modes

- **Best-effort** (`$extends`): sees top-level operations, but the Activity write is not automatically part of the mutation transaction.
- **Iff-committed** (`auditedTransaction`): owns one interactive transaction. Everything uses the same `tx`.

### Value normalization

| Input | Audit representation |
|---|---|
| `bigint` | base-10 string |
| `Date` | ISO-8601 UTC string |
| `Decimal` | canonical decimal string |
| `Uint8Array` / `Buffer` | `{ "$bytes": "<base64>" }` |
| Prisma null sentinels | `{ "$prismaNull": "DbNull" }` |

### Coverage matrix

| Operation | `$extends` | `auditedTransaction` |
|---|---|---|
| `create` | ✅ best-effort | ✅ iff-committed |
| `update` | ✅ best-effort | ✅ iff-committed |
| `delete` | ✅ best-effort | ✅ iff-committed |
| `upsert` | ✅ best-effort | ✅ iff-committed |
| `createMany`/`updateMany`/`deleteMany` | ✅ best-effort | ✅ iff-committed |
| configured nested | ✅ best-effort | ✅ iff-committed |
| referential cascade | ❌ | ❌ |

---

## Querying Activities

`activityQuery(store)` provides immutable typed scopes for filtering activities.

### Basic usage

```ts
import { activityQuery } from 'activitylog-core';

const query = activityQuery(store);

const result = await query
  .inLog('billing')
  .forSubject(subjectRef('Order', '42'))
  .causedBy(causerRef('User', 'admin'))
  .forEvent('updated')
  .forBatch(batchUuid)
  .between(fromDate, toDate)
  .paginate(20, null);
```

### Scopes

| Scope | Description |
|---|---|
| `.inLog(name)` | Filter by log name |
| `.forSubject(ref)` | Filter by subject type + id |
| `.causedBy(ref)` | Filter by causer type + id |
| `.forEvent(event)` | Filter by event name |
| `.forBatch(uuid)` | Filter by batch UUID |
| `.between(from, to)` | Filter by time range |
| `.whereProperty(path, value)` | Filter inside JSON properties |

### Pagination

`.paginate(limit, cursor)` uses the stable `createdAt, id` cursor.

```ts
const page1 = await query.paginate(20, null);
const page2 = await query.paginate(20, page1.nextCursor);
```

### Aggregates

Call `.withAggregates(false)` to exclude aggregate bulk-operation activities.

---

## Schema and Migrations

`activitylog-core` exports `ACTIVITY_LOG_MIGRATIONS` for PostgreSQL, MySQL and SQLite.

```ts
import { ACTIVITY_LOG_MIGRATIONS } from 'activitylog-core';
await database.execute(ACTIVITY_LOG_MIGRATIONS.postgres);
```

### Common schema

| Column | Type | Notes |
|---|---|---|
| `id` | `bigint` (default) or `uuid` v7 | Configurable |
| `log_name` | `varchar(255)` | Channel/bucket, indexed |
| `description` | `text` | Human-readable |
| `subject_type` | `varchar(255)` | Opaque identity string |
| `subject_id` | `varchar(255)` | Stringified PK |
| `causer_type` | `varchar(255)` | Opaque identity string |
| `causer_id` | `varchar(255)` | Stringified PK |
| `event` | `varchar(255)` | Verb |
| `properties` | `jsonb` (PG), `JSON` (MySQL), `TEXT` (SQLite) | Diff + metadata |
| `batch_uuid` | `uuid` | Batch grouping |
| `created_at` | `timestamptz(3)` | Application-generated UTC |

### Indexes

All variants include indexes on `log_name`, `(subject_type, subject_id)`, `(causer_type, causer_id)`.

### ORM references

**TypeORM:**
```ts
@Entity({ name: 'activity_log' })
export class ActivityLogEntity {
  @PrimaryGeneratedColumn({ type: 'bigint' }) id!: string;
  @Column({ name: 'log_name' }) logName!: string;
  @Column('text') description!: string;
  @Column({ name: 'subject_type', nullable: true }) subjectType!: string | null;
  @Column({ name: 'subject_id', nullable: true }) subjectId!: string | null;
  @Column({ name: 'causer_type', nullable: true }) causerType!: string | null;
  @Column({ name: 'causer_id', nullable: true }) causerId!: string | null;
  @Column({ nullable: true }) event!: string | null;
  @Column({ type: 'jsonb' }) properties!: Record<string, unknown>;
  @Column({ name: 'batch_uuid', nullable: true }) batchUuid!: string | null;
  @Column({ name: 'created_at', type: 'timestamptz', precision: 3 }) createdAt!: Date;
}
```

**Prisma:**
```prisma
model ActivityLog {
  id          BigInt   @id @default(autoincrement())
  logName     String   @map("log_name")
  description String
  subjectType String?  @map("subject_type")
  subjectId   String?  @map("subject_id")
  causerType  String?  @map("causer_type")
  causerId    String?  @map("causer_id")
  event       String?
  properties  Json
  batchUuid   String?  @map("batch_uuid")
  createdAt   DateTime @map("created_at") @db.Timestamptz(3)

  @@index([logName])
  @@index([subjectType, subjectId])
  @@index([causerType, causerId])
  @@map("activity_log")
}
```

**Drizzle:**
```ts
export const activityLog = pgTable('activity_log', {
  id: bigserial('id', { mode: 'bigint' }).primaryKey(),
  logName: varchar('log_name', { length: 255 }).notNull(),
  description: text('description').notNull(),
  subjectType: varchar('subject_type', { length: 255 }),
  subjectId: varchar('subject_id', { length: 255 }),
  causerType: varchar('causer_type', { length: 255 }),
  causerId: varchar('causer_id', { length: 255 }),
  event: varchar('event', { length: 255 }),
  properties: jsonb('properties').notNull(),
  batchUuid: varchar('batch_uuid', { length: 255 }),
  createdAt: timestamp('created_at', { withTimezone: true, precision: 3 }).notNull(),
}, (table) => [
  index('idx_activity_log_log_name').on(table.logName),
  index('idx_activity_log_subject').on(table.subjectType, table.subjectId),
  index('idx_activity_log_causer').on(table.causerType, table.causerId),
]);
```

---

## Deliberate Divergences from Spatie

Spatie's Laravel package is the functional inspiration, not a promise that TypeScript ORMs expose the same lifecycle hooks as Eloquent.

### ORM event coverage is explicit

TypeORM and Prisma do not expose equivalent state for every write shape. Activitylog publishes an adapter coverage matrix instead of claiming that one global hook sees everything.

- **TypeORM:** `save`, `remove`, `softRemove` are subscriber conveniences. `auditedUpdate()` is the iff-committed alternative for `Repository.update()`.
- **Prisma:** `$extends` is best-effort. `auditedTransaction()` supplies the iff-committed path.

### Properties use a stable diff envelope

Automatic entity changes store `properties.attributes` and `properties.old`. Creates have empty `old`; hard deletes have empty `attributes`.

### Redaction is built in and enabled by default

Sensitive field names are deeply redacted before persistence. Spatie does not have this.

### Identity is ORM-agnostic

Opaque type + string-compatible single primary key. Composite keys are outside the 1.0 contract.

### Transactions are claimed only where they are controlled

The core never starts or commits transactions. Adapters claim iff-committed only on explicit paths.

---

## Releasing

Releases use the two-phase Changesets workflow on `main`.

### Workflow

1. Feature PRs add/update `.changeset/` files.
2. A push to `main` runs build and consumer smoke tests.
3. `changesets/action` updates `changeset-release/main` and opens the version PR.
4. Merging the version PR publishes all changed packages to npm.

### Repository prerequisites

- Add `NPM_TOKEN` as a repository secret.
- Enable **Allow GitHub Actions to create and approve pull requests** in Settings.
- Keep `main` protections compatible with the workflow permissions.

### Verification

```sh
npm ci
npm run build
npm run smoke:nestjs
npm run smoke:prisma
npx changeset status
```

After publish:

```sh
npm view activitylog-core version dist.integrity dist.attestations
npm view activitylog-nestjs version dist.integrity dist.attestations
npm view activitylog-nextjs version dist.integrity dist.attestations
```

---

## API Architecture

### Key decisions (D1–D23)

| # | Decision |
|---|---|
| D1 | **iff-committed** is the defining invariant. |
| D2 | **Core agnostic of ORM + first-class adapters.** |
| D3 | iff-committed only guaranteed by explicit transactional helpers. |
| D4 | Identity: opaque string types + varchar ids. |
| D5 | Context via ALS singleton in core. |
| D6 | ALS opened early (middleware), causer resolved late. |
| D7 | Queue boundary = explicit contract (`serializeContext()`). |
| D8 | Old values without lock by default; `lockForDiff` opt-in. |
| D9 | Aggregate = 1 Activity with `subject_id=null`. |
| D10 | `bigint` PK default, `uuid` v7 configurable. |
| D11 | `properties` uses best JSON type per dialect. |
| D12 | Outbox deferred beyond 1.0. |
| D13 | Injection choke point via `assertSafeIdentifier`. |
| D14 | Redaction deny-by-default (A09). |
| D15 | Exactly 3 packages under `activitylog-` prefix. |
| D16 | Write pipeline: diff → beforePersist → redaction → persist. |
| D17 | 1.0 anchored on TypeORM; Prisma follows in same major. |
| D18 | Query API: strong types for fixed columns. |
| D19 | `created_at` is UTC generated by the application. |
| D20 | `LogOptions` composes by precedence. |
| D21 | Prisma adapter: explicit dialect, manual tx proxy. |
| D22 | Bulk/nested Prisma: one Aggregate per top-level op. |
| D23 | Prisma adapter: minimal public metadata, portable normalization. |

### Domain glossary

| Term | Definition |
|---|---|
| **Activity** | Immutable record that something happened to an entity. |
| **Subject** | The entity the activity happened to (`performedOn`). |
| **Causer** | Who caused the activity (`causedBy`). |
| **Type** | Opaque identity string — core never interprets it. |
| **Morph map** | String↔entity map to control `*_type` values. |
| **Reference** | Typed `(type, id)` pair (`SubjectRef` / `CauserRef`). |
| **Log name** | Channel/bucket the activity belongs to. |
| **Batch** | Logical unit of work sharing a `batch_uuid`. |
| **Diff** | `{ attributes, old }` — new and previous state. |
| **Aggregate activity** | Single activity for bulk/nested operations. |
| **Store** | Where activities are persisted (`ActivityStore`). |
| **Redaction** | Masking sensitive fields before persistence. |

---

## Error Reference

All exceptions extend `ActivityLogError` and carry a stable `code`:

| Exception | Code | Cause |
|---|---|---|
| `ValidationError` | `VALIDATION_ERROR` | Invalid options |
| `InvalidIdentifierException` | `INVALID_IDENTIFIER` | Unsafe table/column name |
| `QueryExecutionError` | `QUERY_EXECUTION_ERROR` | Driver error during SQL execution |
| `ConfigurationError` | `CONFIGURATION_ERROR` | Unsupported dialect / invalid config |
| `RedactionError` | `REDACTION_ERROR` | Redaction pipeline failure |
