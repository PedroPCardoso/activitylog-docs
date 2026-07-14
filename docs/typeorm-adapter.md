---
title: TypeORM Adapter
category: nestjs
---

# TypeORM Adapter

The `activitylog-nestjs/typeorm` subpath audits entity lifecycle operations through a TypeORM subscriber. Only entities whose changes should be logged need to be decorated.

## Entity decorator

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

## Register the subscriber

After the TypeORM `DataSource` is initialized, register one subscriber:

```ts
import { registerActivityLogSubscriber } from 'activitylog-nestjs/typeorm';

await dataSource.initialize();
registerActivityLogSubscriber(dataSource, { store });
```

Registration happens after initialization because TypeORM rebuilds its configured subscriber list while initializing. Nest applications can perform this step from a provider's `onModuleInit()`.

## Diffs

`repository.save()` creates `created` or `updated` activities. `repository.remove()` and `repository.softRemove()` create `deleted` activities. Each activity uses the entity class name as `subject.type` and the primary column value as `subject.id`:

```ts
{
  attributes: { status: 'paid' },
  old: { status: 'pending' },
}
```

Created activities have empty `old`; hard-deleted activities have empty `attributes`. Soft-delete diffs contain the changed delete-date column. Composite primary keys are outside the 1.0 contract and are skipped.

## auditedUpdate

TypeORM does not provide a reliable old entity to subscribers for `Repository.update()` or update QueryBuilder calls. Use the explicit helper for single-row criteria updates:

```ts
import { auditedUpdate } from 'activitylog-nestjs/typeorm';

await auditedUpdate(
  dataSource.getRepository(Order),
  { id: orderId },
  { status: 'paid' },
);
```

The helper opens or nests a transaction, reads the matching row, performs the update, re-reads it by primary key, calculates the diff and persists the Activity before committing. No match returns `affected: 0` and does not create an Activity.

Use `lockForDiff: true` when the database supports pessimistic write locks and exact old values under concurrent writers justify the contention cost.

## Coverage matrix

| TypeORM operation | Automatic | Supported path | Limitation |
|---|---|---|---|
| `save()` insert | ✅ | `@LogsActivity()` + subscriber | Activity uses event manager when TypeORM supplies the entity |
| `save()` update | ✅ | `@LogsActivity()` + subscriber | Old/new diff for full and partial saves |
| `remove()` | ✅ | `@LogsActivity()` + subscriber | Logged when TypeORM supplies `databaseEntity` |
| `softRemove()` | ✅ | `@LogsActivity()` + subscriber | Delete-date diff when entity and old state are supplied |
| `repository.update()` | ⚠️ | `auditedUpdate()` | Direct call is intentionally not logged |
| update QueryBuilder | ⚠️ | `auditedUpdate()` | Direct QueryBuilder is intentionally not logged |
| bulk update | ❌ | explicit Aggregate activity | Per-row expansion is not fabricated |
| entity-less cascade | ❌ | explicit manual activity | No reliable subject or old state |
