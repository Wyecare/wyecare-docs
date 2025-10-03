# Wyecare Platform Overview

## Purpose
Wyecare is a multi-tenant care-home operating system that unifies resident care management with workforce and compliance tooling. The licensed package contains the full Nx workspace so you can run the platform in your own cloud environment and extend it for your organization or customers.

### Core Domains
- **Resident Care & Tasking** – Real-time task queues, cron-driven status sync, SSE updates (`libs/backend/src/residents/task-queue/task-batch-processor.service.ts`).
- **Workforce & Scheduling** – Shift templates, staffing requirements, payroll snapshots, swaps (`libs/backend/src/shifts`).
- **Compliance & Incidents** – Document management, verification, and incident workflows (`libs/backend/src/documents`, `libs/backend/src/incidents`).
- **Platform Administration** – Multi-tenant setup, organization provisioning, invitations, subscription tracking (`libs/backend/src/platform`).

### Architecture
- **API Service** – NestJS app (`apps/api/src/main.ts`) exposing REST + Swagger (`/api/docs`).
- **Worker Service** – Uses the same image but runs cron/BullMQ processors only when `RUNNER_MODE=worker` (guards in `libs/backend/src/residents/task-queue/task-batch-processor.service.ts` and `libs/backend/src/platform/usage-tracking.service.ts`).
- **Data Layer** – PostgreSQL via Prisma (`prisma/schema.prisma`), Redis via BullMQ, storage via Firebase (upload abstraction under `libs/backend/src/storage`).
- **Frontend** – React/Vite app (`apps/web`) consuming shared DTOs from `libs/shared`.
- **Infrastructure Template** – Terraform module (`terraform/modules/environment`) that provisions Cloud Run, Cloud SQL, Memorystore, Secret Manager, Artifact Registry, Firebase Hosting.

### Multi-Tenancy & Access Control
- Platform admins (seeded by `tools/database/00-platform-admin.seed.ts`) invite organization admins using platform invitation endpoints (`libs/backend/src/platform/organization-invitation.controller.ts`).
- Organization admins configure regions, invite staff, assign roles/permissions (roles + permissions originate from `libs/shared/src/types/index.ts` and seeded by `tools/database/01-foundation.seed.ts`).

### Deliverables in the Licensed Package
1. **Source workspace**: backend, frontend, shared libs, Prisma schema, seeds.
2. **Dockerfile**: single image used for both API and worker.
3. **Terraform**: reference GCP deployment (dev/prod templates + reusable module).
4. **Seed scripts**: platform + production + demo datasets (`tools/database`).
5. **Documentation**: deployment guide, operations runbook, developer guide, API overview, data seeding instructions (this `docs/licensed` collection).

Use the remaining documents in this folder to deploy, operate, extend, and integrate the platform.
