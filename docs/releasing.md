---
title: Releasing
category: reference
---

# Releasing

Releases use the two-phase Changesets workflow on `main`.

## Workflow

1. Feature PRs add or update files in `.changeset/`.
2. A push to `main` runs build and consumer smoke tests.
3. `changesets/action` updates `changeset-release/main` and opens the version PR.
4. Merging the version PR publishes all changed packages to npm.

## Repository prerequisites

- Add an npm automation token as `NPM_TOKEN` repository secret.
- Enable **Allow GitHub Actions to create and approve pull requests** in Settings → Actions.
- Keep `main` protections compatible with `contents: write`, `pull-requests: write`, and `id-token: write`.

## Verification

Before merging a version PR:

```sh
npm ci
npm run build
npm run smoke:nestjs
npm run smoke:prisma
npx changeset status
```

After publish:

```sh
npm view activitylog-core version dist.integrity dist.attestations
npm view activitylog-nestjs version dist.integrity dist.attestations
npm view activitylog-nextjs version dist.integrity dist.attestations
```
