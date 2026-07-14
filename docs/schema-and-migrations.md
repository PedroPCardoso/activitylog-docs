---
title: Schema and Migrations
category: core
---

# Schema and Migrations

`activitylog-core` exports `ACTIVITY_LOG_MIGRATIONS` for PostgreSQL, MySQL, and SQLite. The library does not execute these migrations — apply the matching SQL through your application's migration system.

```ts
import { ACTIVITY_LOG_MIGRATIONS } from 'activitylog-core';

await database.execute(ACTIVITY_LOG_MIGRATIONS.postgres);
```

## Common schema

All variants share:

| Column | Type | Notes |
|---|---|---|
| `id` | `bigint` (default) or `uuid` v7 | Configurable |
| `log_name` | `varchar(255)` | Channel/bucket |
| `description` | `text` | Human-readable |
| `subject_type` | `varchar(255)` | Opaque identity string |
| `subject_id` | `varchar(255)` | Stringified PK |
| `causer_type` | `varchar(255)` | Opaque identity string |
| `causer_id` | `varchar(255)` | Stringified PK |
| `event` | `varchar(255)` | Verb |
| `properties` | `jsonb` / `JSON` / `TEXT` | Diff + metadata |
| `batch_uuid` | `uuid` | Batch grouping |
| `created_at` | `timestamptz(3)` / UTC | Application-generated |

## Indexes

All variants include indexes on:
- `log_name`
- `(subject_type, subject_id)`
- `(causer_type, causer_id)`

## ORM-specific references

- **TypeORM**: `@Entity({ name: 'activity_log' })` with `jsonb` properties and `timestamptz` timestamp
- **Prisma**: `model ActivityLog` with `@map("activity_log")`, Json properties, `@db.Timestamptz(3)`
- **Drizzle**: `pgTable('activity_log', ...)` with `jsonb` and `timestamp(3)` columns

See the full ORM schema references in the [GitHub repo](https://github.com/PedroPCardoso/activitylog/tree/main/docs).
