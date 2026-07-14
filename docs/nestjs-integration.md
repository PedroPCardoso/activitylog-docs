---
title: NestJS Integration
category: nestjs
---

# NestJS Integration

`activitylog-nestjs` exposes a global module and an injectable facade for NestJS applications.

## Global setup

```ts
import { Module } from '@nestjs/common';
import { ActivityLogModule } from 'activitylog-nestjs';

@Module({
  imports: [ActivityLogModule.forRoot({ store })],
})
export class AppModule {}
```

`forRoot()` applies the request middleware once. The middleware opens AsyncLocalStorage before guards run, while the logger resolves `request.user` lazily after a guard has populated it.

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

A plain `request.user` with an `id` becomes a `User` causer. Set `type` on the user object, return a class instance, or pass `causerResolver` to `forRoot()` when the application uses a different identity shape:

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
import { ActivityLogModule } from 'activitylog-nestjs';

@Module({
  imports: [ActivityLogModule.forFeature({
    logOnly: ['name', 'status'],
  })],
})
export class BillingModule {}
```

### Precedence

Options composition follows a strict chain:

**Call > Feature > Root > Defaults**

- Call-time: `.activity('billing', { logOnly: ['name'] })`
- Feature module: `ActivityLogModule.forFeature(options)`
- Root module: `ActivityLogModule.forRoot(options)`
- Built-in defaults

Merge is shallow; arrays are replaced, not merged.

## Interceptor alternative

`ActivityLogInterceptor` is exported as a secondary integration for applications that cannot use middleware. Use either HTTP integration as the context boundary.

```ts
import { ActivityLogInterceptor } from 'activitylog-nestjs';

@Controller()
@UseInterceptors(ActivityLogInterceptor)
export class OrdersController {}
```

## TypeORM adapter

The TypeORM adapter lives under `activitylog-nestjs/typeorm`. See [TypeORM Adapter](typeorm-adapter).
