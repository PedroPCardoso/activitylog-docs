---
title: Prisma Adapter
category: nextjs
---

# Prisma Adapter

The Prisma adapter lives under `activitylog-nextjs/prisma` and provides two modes: a `$extends` client extension (best-effort) plus an explicit transactional helper (`auditedTransaction`) for iff-committed guarantees.

## Setup

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
  dialect: 'sqlite' | 'postgres' | 'mysql';           // explicit dialect
  models?: Record<string, {                           // model metadata
    idField?: string;                                  // default "id"
    relationFields?: readonly string[];
    auditFields?: readonly string[];                   // re-include globally omitted fields
  }>;
  lockForDiff?: false;                                 // Prisma cannot express portable row locks
  tableName?: string;
};
```

Or use a custom store:

```ts
{
  store: myCustomStore;
  storeTransactionMode?: 'none' | 'uses-context';
  tableName?: never;
}
```

## Iff-committed transactions

Use `auditedTransaction` when mutation and Activity must commit or roll back together:

```ts
import { auditedTransaction } from 'activitylog-nextjs/prisma';

await auditedTransaction(prisma, options, async (tx) => {
  await tx.user.update({ where: { id: 1 }, data: { name: 'Atomic' } });
});
```

The helper opens one Prisma interactive transaction and passes a manual audited proxy over that exact `tx` client to the callback. An exception from the callback or logging pipeline rolls all of them back.

## Individual operations

| Prisma operation | Event | Diff source |
|---|---|---|
| `create` | `created` | returned record as `attributes`, empty `old` |
| `update` | `updated` | pre-read record as `old`, returned record as `attributes` |
| `delete` | `deleted` | pre-read/returned record as `old`, empty `attributes` |
| `upsert` | `created` or `updated` | pre-read determines branch |

## Bulk operations

Bulk operations never fabricate per-row old/new values. They emit one Aggregate activity:

```ts
{
  aggregate: true,
  criteria: args.where ?? {},
  changes: args.data,
  affected: result.count,
}
```

### Mapping

| Operation | Event | `criteria` | `changes` | `affected` |
|---|---|---|---|---|
| `createMany` | `created` | `{}` | `{ data, skipDuplicates? }` | `result.count` |
| `createManyAndReturn` | `created` | `{}` | `{ data, skipDuplicates? }` | returned array length |
| `updateMany` | `updated` | `where ?? {}` | `data` | `result.count` |
| `deleteMany` | `deleted` | `where ?? {}` | `{}` | `result.count` |

Aggregate Activities use `subject_type = model` and `subject_id = null`.

## Nested writes

Nested writes are recorded as one Aggregate activity for the top-level model/operation — never decomposed by relation. Detection uses the `relationFields` declared in model config. Nested syntax is recognized only under explicitly listed relation fields.

## Coverage matrix

| Prisma operation | `$extends` client | `auditedTransaction` |
|---|---|---|
| `create` | ✅ best-effort | ✅ iff-committed |
| `update` | ✅ best-effort | ✅ iff-committed |
| `delete` | ✅ best-effort | ✅ iff-committed |
| `upsert` | ✅ best-effort | ✅ iff-committed |
| `createMany` / `updateMany` / `deleteMany` | ✅ best-effort | ✅ iff-committed |
| `*ManyAndReturn` | ✅ best-effort | ✅ iff-committed |
| configured nested writes | ✅ best-effort | ✅ iff-committed |
| unconfigured relation field | ⚠️ not detected | ⚠️ not detected |
| database referential cascade | ❌ | ❌ |

## Value normalization

Before values enter the diff engine, they are normalized:

| Input | Audit representation |
|---|---|
| `bigint` | base-10 string |
| `Date` | ISO-8601 UTC string |
| Prisma `Decimal` | canonical decimal string |
| `Uint8Array` / Node `Buffer` | `{ "$bytes": "<base64>" }` |
| Prisma `DbNull`, `JsonNull`, `AnyNull` | `{ "$prismaNull": "DbNull" }` |
| array | recursively normalized |
| plain object | recursively normalized; undefined omitted |

## Transaction modes

- **Best-effort** (`$extends`): sees top-level operations, but the Activity write is not automatically part of the mutation transaction. Old reads have a race window.
- **Iff-committed** (`auditedTransaction`): owns one interactive transaction. Everything — pre-reads, mutations, Activity writes — uses the same `tx`.
