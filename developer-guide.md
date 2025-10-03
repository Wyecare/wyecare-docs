# Developer Guide

## Workspace Layout
- `apps/api` – NestJS service (controllers bootstrap via `AppModule`, main entry `apps/api/src/main.ts`).
- `apps/web` – React/Vite frontend consuming shared DTOs.
- `libs/backend` – Domain modules, exported via `libs/backend/src/index.ts`. Includes auth, residents, shifts, incidents, queue, platform, etc.
- `libs/shared` – Shared DTOs, enums, constants for backend + frontend (`libs/shared/src/types/index.ts`).
- `tools/database` – Prisma seed scripts and helpers.
- `terraform` – Infrastructure templates.

## Building & Running
```bash
npm install
npx nx serve api      # dev API with hot reload
npx nx dev web        # dev web (Vite)
npx nx build api      # production API build
npx nx build web      # production web bundle
```
Prisma client: `npx prisma generate`. Run migrations locally against `.env` database with `npx prisma migrate dev` (development only).

## Shared Libraries
Use TypeScript path imports (`@wyecare-solutions/backend`, `@wyecare-solutions/shared`). Library metadata lives in each `libs/**/package.json`. To add a new domain module, scaffold under `libs/backend/src/<domain>` and export it through `libs/backend/src/index.ts`.

## Cron Guards
All cron handlers call `isWorkerProcess()` (`libs/backend/src/lib/worker-context.util.ts`). New scheduled jobs should follow the same pattern to run only when `RUNNER_MODE=worker`.

## Queue Processing
BullMQ queues live under `libs/backend/src/queue`. Workers register processors automatically when the worker service boots. When adding jobs, keep queue names and flows centralized there.

## Seeding & Fixtures
Seed scripts use Prisma (`@prisma/client`). Refer to existing patterns in `tools/database/01-foundation.seed.ts` and `02-production.seed.ts`. Run seeds via `npx tsx` (TypeScript executed in Node).

## Testing & Linting
- Lint all projects: `npx nx lint --all`.
- Tests (where available): `npx nx test <project>`.
- Frontend uses Vite/Test libraries; backend uses Jest (see respective `jest.config.ts`).

## Extending the Platform
- Add new REST endpoints by creating a Nest module/controller/service in `libs/backend` and importing it into `BackendModule` (`libs/backend/src/lib/backend.module.ts`).
- Share DTOs between backend and frontend by updating `libs/shared/src/types` and re-exporting via `libs/shared/src/index.ts`.
- Update infrastructure variables by adding inputs to `terraform/modules/environment/variables.tf` and passing them from environment-specific `main.tf`.

Use this guide alongside the API overview for functional references.
