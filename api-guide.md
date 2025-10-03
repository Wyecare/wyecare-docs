# API & Workflow Overview

## Authentication
- **Login**: `POST /auth/login` (`libs/backend/src/auth/auth.controller.ts:18-30`). Returns access + refresh tokens (`AuthService.login` in `libs/backend/src/auth/auth.service.ts`).
- **Accept Invitation**: `POST /auth/accept-invitation` – organization users set password via emailed token.
- **Refresh**: `POST /auth/refresh` – exchange refresh token for new access token.
- All protected routes require `Authorization: Bearer <token>`; guards enforce permissions (`libs/backend/src/auth/guards`).

## Platform Administration
- Platform admins (seeded via `00-platform-admin.seed.ts`) authenticate and manage organizations.
- **Organization Invitations**: `libs/backend/src/platform/organization-invitation.controller.ts` – invites organization admins by email.
- **Organization Setup**: `organization-setup.controller.ts` – mark org as configured, upload logos, set preferences.
- **Platform Users/Organizations**: `platform-users.*`, `platform-organizations.*` manage multi-tenant registry.

## Organization Operations
- **Users**: `libs/backend/src/users` – CRUD, role assignment, invitations; email credentials send via `MailerModule`.
- **Residents**: `libs/backend/src/residents` – register residents, care plans, tasks, SSE updates (`resident-task-queue` module).
- **Shifts & Time Tracking**: `libs/backend/src/shifts` – templates, scheduling, time entries, payroll snapshots.
- **Documents**: `libs/backend/src/documents` – upload, tag, verify compliance documents.
- **Incidents**: `libs/backend/src/incidents` – file, assign, resolve incidents.
- **Dashboard**: `libs/backend/src/dashboard` – aggregated metrics (queried via controllers).

## Realtime & Background
- SSE for task updates originates in `resident-task-queue` module; worker cron keeps statuses accurate.
- BullMQ queue processors handle asynchronous jobs (e.g., notifications) – located under `libs/backend/src/queue`.

## API Documentation (Swagger)
Run the API and visit `/api/docs` (enable `NODE_ENV=development` or log in with valid credentials). Swagger auto-generates from Nest decorators in controllers.

## Role & Permission Model
- Permissions defined in `libs/shared/src/types/index.ts` and seeded by `01-foundation.seed.ts`.
- Roles (e.g., `organization_admin`, `nurse`, `care_assistant`) include permission lists; assigned via `userRoles` relations.
- Platform roles (`PlatformRole` enum) govern platform-level access (super admin, platform admin, accountant, support manager).

For full payloads and DTOs, reference `libs/shared/src/types` and inspect the Swagger schema.
