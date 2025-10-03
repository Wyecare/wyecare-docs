# Data Seeding Guide

The repository provides TypeScript seed scripts under `tools/database`. Execute them with `npx tsx` after running Prisma migrations (`npx prisma migrate deploy`). Seeds are idempotent courtesy of Prisma `upsert` calls.

## Seed Pipeline
1. **Platform Admins** – `tools/database/00-platform-admin.seed.ts`
   - Creates platform super admin, platform admin, accountant, support manager (passwords printed to console).
   - Permissions per role defined in `DEFAULT_PLATFORM_USERS` and `PLATFORM_PERMISSIONS`.
2. **Foundation Data** – `tools/database/01-foundation.seed.ts`
   - Seeds permissions, roles, document tags, task templates, shift templates, staffing requirements using shared constants (from `@wyecare-solutions/shared`).
3. **Production Baseline** – `tools/database/02-production.seed.ts`
   - Creates a clean organization (`slug: your-organization`), marks it `isSetup=false`, and an initial admin user (`admin@yourorganization.com / Admin123!`).
4. **Optional Datasets**
   - Development: `tools/database/03-development.seed.ts` – multi-region org, rich staff/resident data.
   - Demo: `tools/database/04-demo.seed.ts` – curated demo scenarios (check README for credentials).

## Execution Example
```bash
npx prisma migrate deploy
npx tsx tools/database/00-platform-admin.seed.ts
npx tsx tools/database/01-foundation.seed.ts
npx tsx tools/database/02-production.seed.ts
```
To add demo data:
```bash
npx tsx tools/database/03-development.seed.ts
```

## Shared Constants
Seed scripts depend on enums/arrays exported by `libs/shared`:
- Permissions, roles, document tags (`libs/shared/src/types/index.ts`).
- Shift templates, staffing requirements (`libs/shared/src/types/index.ts` under `DEFAULT_SHIFT_TEMPLATES`, etc.).

Keep seeds synchronized with these constants; update `libs/shared` when changing permission or role definitions.
