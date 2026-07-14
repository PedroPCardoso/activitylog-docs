---
title: Prisma Adapter
category: nextjs
---

# Prisma Adapter

The Prisma adapter lives under `activitylog-nextjs/prisma` and works via a `$extends` client extension plus an explicit transactional helper.

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

## Iff-committed transactions

Use `auditedTransaction` when mutation and activity must commit or roll back together:

```ts
import { auditedTransaction } from 'activitylog-nextjs/prisma';

await auditedTransaction(prisma, options, async (tx) => {
  await tx.user.update({ where: { id: 1 }, data: { name: 'Atomic' } });
});
```

## Individual operations

| Operation | Event | Diff |
|---|---|---|
| `create` | `created` | `attributes` from returned record |
| `update` | `updated` | `old` from pre-read, `attributes` from returned record |
| `delete` | `deleted` | `old` from pre-read |
| `upsert` | `created` or `updated` | pre-read determines branch |

## Bulk operations

Bulk operations emit one aggregate activity per top-level call:

```ts
{
  aggregate: true,
  criteria: args.where ?? {},
  changes: args.data,
  affected: result.count,
}
```

Aggregate activities use `subject_type = model` and `subject_id = null`.

## Nested writes

Nested writes are recorded as one aggregate activity for the top-level model/operation — never decomposed by relation. Detection uses the `relationFields` declared in model config.

## Coverage matrix

| Operation | `$extends` client | `auditedTransaction` |
|---|---|---|
| `create` | ✅ best-effort | ✅ iff-committed |
| `update` | ✅ best-effort | ✅ iff-committed |
| `delete` | ✅ best-effort | ✅ iff-committed |
| `upsert` | ✅ best-effort | ✅ iff-committed |
| `createMany` / `updateMany` / `deleteMany` | ✅ best-effort | ✅ iff-committed |
| configured nested writes | ✅ best-effort | ✅ iff-committed |
| database referential cascade | ❌ | ❌ |

## Value normalization

Before entering the diff engine, values are normalized:

| Input | Audit value |
|---|---|
| `bigint` | base-10 string |
| `Date` | ISO-8601 UTC string |
| `Decimal` | canonical decimal string |
| `Uint8Array` / `Buffer` | `{ "$bytes": "<base64>" }` |
| Prisma null sentinels | `{ "$prismaNull": "DbNull" }` |
