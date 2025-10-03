# Operations Runbook

## Services
- **API Cloud Run Service** (`terraform/modules/environment/main.tf:408-506`): Handles HTTP traffic, scales to zero when idle.
- **Worker Cloud Run Service** (`terraform/modules/environment/main.tf:520-579`): Keeps `min_instances=1`, runs cron jobs and BullMQ processors.
- **Cloud SQL (Postgres)**: Provisioned via Terraform (name `wyecare-sql`).
- **Memorystore (Redis)**: Used for queues/cache (`libs/backend/src/queue`).
- **Firebase Storage/Hosting**: File storage abstraction (`libs/backend/src/storage`) and frontend hosting.

## Environment Variables
Injected through Secret Manager references:
- `NODE_ENV`, `APP_TIMEZONE`, `DATABASE_URL`, `JWT_SECRET`, `REDIS_URL`, `MAIL_FROM`, optional `FRONTEND_URL` (API only).
- Worker adds `RUNNER_MODE=worker` and reuses `DATABASE_URL`, `JWT_SECRET`, `REDIS_URL`.
- SMTP secrets (`smtp-host`, `smtp-user`, `smtp-pass`) load via Secret Manager; ensure versions exist before sending emails.

## Cron & Background Jobs
Cron handlers live in:
- `libs/backend/src/residents/task-queue/task-batch-processor.service.ts`
- `libs/backend/src/platform/usage-tracking.service.ts`

Both call `isWorkerProcess()` (`libs/backend/src/lib/worker-context.util.ts`) at runtime. Only the worker service (with `RUNNER_MODE=worker`) executes cron logic. If the worker is scaled to zero or fails, automation stops—monitor logs and set alerts on Cloud Run.

## Database Maintenance
To run migrations or seeds manually:
1. Start Cloud SQL proxy: `cloud-sql-proxy <connection> --unix-socket /tmp &`
2. Export `DATABASE_URL` from Secret Manager.
3. Run `npx prisma migrate deploy` or any seed script (`tools/database/...`).

## Secrets Rotation
Add new versions via:
```bash
echo -n 'new-value' | gcloud secrets versions add <secret-name> --data-file=- --project <project>
```
Cloud Run reads `latest`, so rotate during low traffic and redeploy services to pick up changes if necessary.

## Monitoring & Logging
- Use Cloud Run logs for API/worker service health (`gcloud run services logs read <service>`).
- Cloud SQL insights for slow queries (enable query stats).
- Memorystore metrics via Cloud Monitoring.
- Firebase hosting logs via Firebase console for frontend deployments.

## Scaling
- API: `min_instances_api = 0`, `max_instances_api = 3` by default. Adjust in `terraform/environments/<env>/main.tf` if needed.
- Worker: `min_instances_worker = 1`, `max_instances_worker = 2`. Increase for heavy cron load; ensure cron handlers remain idempotent.

## Disaster Recovery
- Cloud SQL backups (enable automated backups and point-in-time recovery).
- Secrets stored in Secret Manager (versioned).
- Artifact Registry stores container images; tag releases accordingly.

## Common Tasks
- **Redeploy API/Worker**: push to `dev` or `prod` branch; GitHub workflow builds & deploys.
- **Seed New Environment**: follow Migration & Seeding steps above; confirm platform admins exist (`tools/database/00-platform-admin.seed.ts` prints credentials).
- **Update Frontend**: adjust `.github/actions/deploy-frontend` or run Firebase CLI manually.

Keep this runbook alongside the deployment guide for on-call and customer operations teams.
