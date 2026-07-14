---
title: Querying Activities
category: core
---

# Querying Activities

`activityQuery(store)` provides immutable typed scopes for filtering activities.

## Basic usage

```ts
import { activityQuery } from 'activitylog-core';

const query = activityQuery(store);
const result = await query
  .inLog('billing')
  .forSubject(subjectRef('Order', '42'))
  .paginate(20, null);

console.log(result.items);
console.log(result.nextCursor); // nullable, for the next page
```

## Scopes

| Scope | Description |
|---|---|
| `.inLog(name)` | Filter by log name |
| `.forSubject(ref)` | Filter by subject type + id |
| `.causedBy(ref)` | Filter by causer type + id |
| `.forEvent(event)` | Filter by event name |
| `.forBatch(uuid)` | Filter by batch UUID |
| `.between(from, to)` | Filter by time range |
| `.whereProperty(path, value)` | Filter inside JSON properties |

## Pagination

`.paginate(limit, cursor)` uses the stable `createdAt, id` cursor and returns `items` plus a nullable `nextCursor`.

```ts
// First page
const page1 = await query.paginate(20, null);

// Second page
const page2 = await query.paginate(20, page1.nextCursor);
```

## Aggregates

Call `.withAggregates(false)` to exclude aggregate bulk-operation activities from results.
