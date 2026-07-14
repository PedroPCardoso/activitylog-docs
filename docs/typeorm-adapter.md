---
title: TypeORM Adapter
category: nestjs
---

# TypeORM Adapter

The `activitylog-nestjs/typeorm` subpath audits entity lifecycle operations through a TypeORM subscriber.

## Entity decorator

Decorate only the entities whose changes should be logged:

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

## Diffs and options

`repository.save()` creates `created` or `updated` activities. `repository.remove()` and `repository.softRemove()` create `deleted` activities.

```ts
{
  attributes: { status: 'paid' },
  old: { status: 'pending' },
}
```

Created activities have empty `old`; hard-deleted activities have empty `attributes`. Composite primary keys are outside the 0.x contract and are skipped.

## auditedUpdate

TypeORM does not provide a reliable old entity for `Repository.update()` or update QueryBuilder calls. Use the explicit helper for one-row criteria updates:

```ts
import { auditedUpdate } from 'activitylog-nestjs/typeorm';

await auditedUpdate(
  dataSource.getRepository(Order),
  { id: orderId },
  { status: 'paid' },
);
```

The helper opens or nests a transaction, reads the matching row, performs the update, re-reads it, calculates the diff, and persists the activity before committing.

## Coverage matrix

| Operation | Auto | Supported path |
|---|---|---|
| `save()` insert | ✅ | `@LogsActivity()` + subscriber |
| `save()` update | ✅ | `@LogsActivity()` + subscriber |
| `remove()` | ✅ | `@LogsActivity()` + subscriber |
| `softRemove()` | ✅ | `@LogsActivity()` + subscriber |
| `repository.update()` | ⚠️ | Use `auditedUpdate()` |
| update QueryBuilder | ⚠️ | Use `auditedUpdate()` |
| bulk update | ❌ | Explicit aggregate activity |
| entity-less cascade | ❌ | Explicit manual activity |
