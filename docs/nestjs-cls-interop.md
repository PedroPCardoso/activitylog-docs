---
title: nestjs-cls Interoperability
category: nestjs
---

# Interoperability with nestjs-cls

`activitylog` owns its AsyncLocalStorage because its batch and logging controls must also work outside NestJS. An application that already uses `nestjs-cls` therefore has two independent ALS stores. They can be nested safely, but they should not become two independent sources of user identity.

Use `nestjs-cls` as the source of truth and read it from activitylog's lazy `causerResolver`. This keeps both packages decoupled and avoids copying the user when middleware starts, before authentication guards have run.

## Configuration

Mount the `nestjs-cls` middleware before `ActivityLogModule`:

```ts
import { Module } from '@nestjs/common';
import { ClsModule, ClsServiceManager, type ClsStore } from 'nestjs-cls';
import { ActivityLogModule, causerRef } from 'activitylog-nestjs';

interface AppClsStore extends ClsStore {
  user?: { id: string };
}

@Module({
  imports: [
    ClsModule.forRoot({
      global: true,
      middleware: { mount: true },
    }),
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

`ClsServiceManager.getClsService()` is the supported nestjs-cls escape hatch for reading the active store outside Nest's injection context. The callback is executed by activitylog only when an Activity is logged and the builder did not already choose an explicit or anonymous causer.

## Authentication guard

The authentication guard should write the user once, to `nestjs-cls`:

```ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private readonly cls: ClsService<AppClsStore>) {}

  canActivate(context: ExecutionContext): boolean {
    const user = authenticate(context.switchToHttp().getRequest());
    this.cls.set('user', user);
    return true;
  }
}
```

Do not also copy the same user into activitylog context. The lazy resolver reads the current CLS value, including changes made after middleware initialization.

## Context boundaries

The resulting nesting is:

```
ClsMiddleware.run(request)
  └─ ActivityLogMiddleware.run(request)
       └─ guards → controller → services → activity().log()
```

Both stores are active inside that chain and both are released when it ends. They do not share data automatically.

## Queue boundary

Neither ALS crosses a queue automatically. Materialize the causer when creating a job payload:

```ts
const cls = ClsServiceManager.getClsService<AppClsStore>();
const user = cls.get('user');
const activitylogContext = runWithContext(
  { causer: user ? causerRef('User', user.id) : null },
  () => serializeContext(),
);
```

The worker restores only the serialized activitylog context with `runWithContext(activitylogContext, handler)`. A worker that receives no context logs a null/system causer.
