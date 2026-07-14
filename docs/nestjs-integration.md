---
title: NestJS Integration
category: nestjs
---

# NestJS Integration

`activitylog-nestjs` exposes a global module and an injectable façade.

## Setup

```ts
import { Module } from '@nestjs/common';
import { ActivityLogModule } from 'activitylog-nestjs';

@Module({
  imports: [ActivityLogModule.forRoot({ store })],
})
export class AppModule {}
```

`forRoot()` applies the request middleware once. The middleware opens AsyncLocalStorage before guards run, while the logger resolves `request.user` lazily after a guard has had a chance to populate it.

## Usage in services

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

## Causer resolution

A plain `request.user` with an `id` becomes a `User` causer. Set `type` on the user object, return a class instance, or pass `causerResolver` to `forRoot()` when your application uses a different identity shape:

```ts
ActivityLogModule.forRoot({
  store,
  causerResolver: (request) =>
    request.user
      ? { type: 'Account', id: request.user.accountId }
      : null,
});
```

## Feature modules

Feature modules can set local defaults:

```ts
ActivityLogModule.forFeature({
  logOnly: ['name', 'status'],
});
```

Precedence: call > feature > root > default.

## Interceptor alternative

`ActivityLogInterceptor` is exported as a secondary integration for applications that cannot use middleware. Use either HTTP integration as the context boundary.

## nestjs-cls interop

Applications that already keep their authenticated user in `nestjs-cls` can use it as the single identity source without adding a runtime dependency. See the [nestjs-cls recipe](nestjs-cls) for the lazy resolver pattern, middleware ordering, and queue-boundary caveat.

## TypeORM adapter

The TypeORM adapter lives under the `activitylog-nestjs/typeorm` subpath. See [TypeORM Adapter](typeorm-adapter).
