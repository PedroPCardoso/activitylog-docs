# activitylog

**ORM-agnostic entity audit trail for TypeScript**, with the DX of [spatie/laravel-activitylog](https://github.com/spatie/laravel-activitylog). The core knows nothing about any ORM or NestJS — adapters are first-class citizens.

> **Status:** production-ready. Core (manual logging, context, query API), NestJS module, TypeORM, and Prisma adapters are published. Drizzle adapter coming next.

## The bet

- **ORM-agnostic core + first-class adapters** — TypeORM, Prisma, Drizzle.
- **`iff-committed`** — an activity persists *if and only if* the mutation that caused it commits (when transactional). The audit trail never orphans or drops a record.
- **Causer resolved automatically** from request context (AsyncLocalStorage).
- **Honest coverage** — where a guarantee isn't possible (e.g., bulk/nested writes), it's declared in a coverage matrix, never faked.
- **Redaction on by default** — passwords, tokens, and PII don't leak into the audit trail.

## Packages

| Package | What |
|---|---|
| `activitylog-core` | Agnostic core: logger, store, diff, context, query API |
| `activitylog-nestjs` | NestJS module + TypeORM adapter (subpath) |
| `activitylog-nextjs` | Prisma + Drizzle adapters (subpaths) |

## Contents

- [Getting Started](docs/getting-started)
- [Manual Logging](docs/manual-logging)
- [Context and Batches](docs/context-and-batches)
- [NestJS Integration](docs/nestjs-integration)
- [TypeORM Adapter](docs/typeorm-adapter)
- [Prisma Adapter](docs/prisma-adapter)
- [Querying Activities](docs/querying-activities)
- [Schema and Migrations](docs/schema-and-migrations)
- [Architecture Decisions](docs/architecture-decisions)
- [Releasing](docs/releasing)

## License

[MIT](LICENSE)
