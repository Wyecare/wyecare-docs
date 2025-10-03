# Deployment Guide (Licensed Edition)

This guide walks through setting up Wyecare in your own Google Cloud project using the provided Terraform templates. Adapt these steps if you deploy to a different cloud.

## 1. Prerequisites
- Node.js 20 and npm (`package.json: engines not pinned but workspace builds with Node20`).
- Terraform ≥ 1.7 (`terraform/environments/dev/main.tf` requires it).
- Google Cloud SDK (`gcloud`) authenticated to the target project.
- Permissions to create Cloud Run, Cloud SQL, Memorystore, Secret Manager, Artifact Registry, Firebase.

## 2. Install Dependencies
```bash
npm install
```
This installs all workspace packages (API, web, backend libs, Prisma).

## 3. Configure Secrets
The Terraform module expects Secret Manager entries; create versions for each secret before running `apply`:

```bash
PROJECT_ID=wyecare-dev-app # change per environment

echo -n 'postgres://user:pass@host/db' | gcloud secrets versions add database-url --data-file=- --project $PROJECT_ID
echo -n 'your-jwt-secret'        | gcloud secrets versions add jwt-secret     --data-file=- --project $PROJECT_ID
echo -n 'redis://host:6379'      | gcloud secrets versions add redis-url     --data-file=- --project $PROJECT_ID
echo -n 'Wyecare Solutions'      | gcloud secrets versions add mail-from     --data-file=- --project $PROJECT_ID
# Optional frontend URL
# echo -n 'https://app.example.com' | gcloud secrets versions add frontend-url --data-file=- --project $PROJECT_ID
# SMTP (required for email invites)
echo -n 'smtp.yourhost.com'      | gcloud secrets versions add smtp-host     --data-file=- --project $PROJECT_ID
echo -n 'noreply@example.com'    | gcloud secrets versions add smtp-user     --data-file=- --project $PROJECT_ID
echo -n 'app-password-here'      | gcloud secrets versions add smtp-pass     --data-file=- --project $PROJECT_ID
```

## 4. Terraform Setup
Each environment folder (`terraform/environments/dev` or `prod`) wraps the shared module (`terraform/modules/environment`). The module provisions:
- Cloud Run services (API + worker)
- Cloud SQL (Postgres) with private VPC connector
- Memorystore (Redis)
- Secret Manager secrets (referenced in the Cloud Run env blocks `terraform/modules/environment/main.tf:430-506`)
- Artifact Registry
- Firebase (project + hosting site)

**Run Terraform**:
```bash
cd terraform/environments/dev
terraform init
terraform plan -var "github_repository=<owner/repo>"
terraform apply -var "github_repository=<owner/repo>" -auto-approve
```
Replace `dev` with `prod` for the production project. The module outputs Cloud Run URLs, artifact repo, Firebase site (`terraform/environments/dev/main.tf:41-66`, `outputs.tf`).

## 5. Database Migration & Seeding
After infrastructure is up, run migrations and seeds against Cloud SQL. Use the Cloud SQL Auth Proxy:

```bash
CONNECTION=$(gcloud sql instances describe wyecare-sql --project $PROJECT_ID --format='value(connectionName)')
cloud-sql-proxy "$CONNECTION" --unix-socket /tmp &
export DATABASE_URL=$(gcloud secrets versions access latest --secret=database-url --project $PROJECT_ID)

npx prisma migrate deploy
npx tsx tools/database/00-platform-admin.seed.ts
npx tsx tools/database/01-foundation.seed.ts
npx tsx tools/database/02-production.seed.ts
```
This creates platform roles/users -> core permissions -> a clean organization with setup admin (`tools/database/02-production.seed.ts:42-119`).

Optional demo data:
```bash
npx tsx tools/database/03-development.seed.ts
# or npx tsx tools/database/04-demo.seed.ts
```

## 6. Frontend Deployment
The GitHub workflow (`.github/workflows/deploy.yml`) invokes `.github/actions/deploy-frontend`, which:
1. Writes `apps/web/.env` with the API URL/timezone.
2. Runs `npx nx build web --configuration=production --skip-nx-cache`.
3. Calls `FirebaseExtended/action-hosting-deploy@v0` with the configured service account to push to the `live` channel.

To deploy manually, follow the same steps:
```bash
cat > apps/web/.env <<EOF
VITE_APP_API_URL=<api-url>
VITE_APP_TIMEZONE=Asia/Kolkata
EOF
npx nx build web --configuration=production
firebase hosting:deploy --project $PROJECT_ID --only <firebase_site_id>
```
Authenticate with Firebase (`firebase login`) and supply the same service account used in CI if you rely on `FirebaseExtended/action-hosting-deploy@v0` locally.

## 7. Accessing the Platform
- **API Docs**: `https://<api-url>/api/docs`
- **Platform Admin Login**: Use credentials shown after `00-platform-admin.seed.ts` executes (e.g., `admin@wyecare.solutions`).
- **Organization Setup**: Platform admin invites org admin via platform endpoints; org admin logs in, completes organization profile (`libs/backend/src/platform/organization-setup.service.ts`).

## 8. Production Considerations
- Ensure `RUNNER_MODE=worker` remains set on the worker Cloud Run service so cron executes only there (`terraform/modules/environment/main.tf:530-534`).
- Configure SMTP secrets before sending invites.
- Monitor Cloud Run logs and Cloud SQL metrics (enable alerts to catch cron failures).
- For non-GCP deployments, adapt the Terraform module or recreate equivalent infrastructure (e.g., AWS ECS + RDS + ElastiCache).

Proceed to the Operations Runbook for runtime procedures.
