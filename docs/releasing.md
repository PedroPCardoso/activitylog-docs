---
title: Releasing
category: reference
---

# Releasing

Releases use the two-phase Changesets workflow on `main`.

## Workflow

1. Feature PRs add or update files in `.changeset/`.
2. A push to `main` runs build and the installed-consumer smoke tests.
3. `changesets/action` updates `changeset-release/main` and opens/updates the version PR.
4. Merging that version PR runs the workflow again; with no pending changesets, the action calls `npm run release` and creates GitHub releases for newly published package versions.

The package manifests set `publishConfig.access: public` and `publishConfig.provenance: true`. The workflow grants only the release job `id-token: write`, allowing npm to attach GitHub Actions provenance to published artifacts.

Unreleased workspaces remain `private: true` even when they already have a package skeleton.

## Repository prerequisites

- Add an npm automation/granular access token as the repository secret `NPM_TOKEN`. The workflow maps it to both `NPM_TOKEN` and `NODE_AUTH_TOKEN`.
- In **Settings → Actions → General → Workflow permissions**, enable **Allow GitHub Actions to create and approve pull requests** so `changesets/action` can create the version PR.
- Keep the `main` environment and branch protections compatible with the release workflow's `contents: write`, `pull-requests: write`, and `id-token: write` permissions.

If the repository intentionally disallows bot-created PRs, a maintainer may open the PR manually from `changeset-release/main` to `main`.

## Verification

Before merging a version PR:

```sh
npm ci
npm run build
npm run smoke:nestjs
npm run smoke:prisma
npx changeset status
```

After the core/NestJS publish run succeeds, verify their exact public versions:

```sh
npm view activitylog-core version dist.integrity dist.attestations
npm view activitylog-nestjs version dist.integrity dist.attestations
npm view activitylog-nextjs version dist.integrity dist.attestations
```

Install the public versions into clean NestJS and Prisma consumers before closing the release issue.
