# BoloDesk on Railway

Deploy `guptaakashgupta/chatwoot`, branch `pilot/india`, to
`https://bolodesk.askpocket.in`. Build this repository, not the upstream Chatwoot
image, so the pilot branding is included. Commit and push the branding assets,
installation configuration and deployment changes before connecting GitHub.

## Services

Create one project with a production environment. Keep the application, database
and Redis in the same region, preferably Singapore for this pilot.

| Service | Source | Configuration |
| --- | --- | --- |
| Postgres | PostgreSQL template with pgvector | Persistent volume; enable scheduled backups and point-in-time recovery |
| Redis | Redis template | Persistent volume; configure `maxmemory-policy noeviction` for Sidekiq |
| Uploads | Railway storage bucket | Private bucket shared by web and worker |
| Web | GitHub `pilot/india` | Dockerfile `docker/Dockerfile`; settings below |
| Worker | Same repository and branch | Same Dockerfile; settings below |

The database must provide `vector`, `pg_stat_statements`, `pg_trgm`, and
`pgcrypto`, as required by `db/schema.rb`. Confirm the chosen template supports
all four before initializing the application. Use the database's private URL.

Use Railway's service settings for the build and deploy configuration. Do not
create a legacy `railway.json`/`railway.toml`: Railway documents these as
deprecated in favor of its newer Infrastructure as Code workflow.

## Shared application variables

Set these on both Web and Worker; secrets belong in Railway, never in Git.
Reference the actual service names when using Railway variable references.

```dotenv
RAILWAY_DOCKERFILE_PATH=docker/Dockerfile
RAILS_ENV=production
NODE_ENV=production
INSTALLATION_ENV=docker
RAILS_LOG_TO_STDOUT=true
RAILS_SERVE_STATIC_FILES=true
FRONTEND_URL=https://bolodesk.askpocket.in
FORCE_SSL=true
ENABLE_ACCOUNT_SIGNUP=false
WEB_CONCURRENCY=0
RAILS_MAX_THREADS=5
SIDEKIQ_CONCURRENCY=5
ACTIVE_STORAGE_SERVICE=s3_compatible
```

Also configure:

| Variable | Value |
| --- | --- |
| `SECRET_KEY_BASE` | One generated secret, identical for web and worker |
| `DATABASE_URL` | Reference Postgres's private `DATABASE_URL` |
| `REDIS_URL` | Reference Redis's authenticated private `REDIS_URL` |
| `STORAGE_ACCESS_KEY_ID` | Uploads bucket access key |
| `STORAGE_SECRET_ACCESS_KEY` | Uploads bucket secret key |
| `STORAGE_REGION` | Region from the bucket's credentials |
| `STORAGE_BUCKET_NAME` | Bucket name from its credentials |
| `STORAGE_ENDPOINT` | Base S3 endpoint from its credentials, without the bucket prefix |

Leave `STORAGE_FORCE_PATH_STYLE` unset: Railway buckets use virtual-hosted URLs.
Do not use local attachment storage: web and worker have separate filesystems.
Configure bucket CORS for `https://bolodesk.askpocket.in` if browser direct uploads
are used, following Railway's bucket documentation.

Configure a verified email sender and your email provider's `SMTP_ADDRESS`,
`SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_AUTHENTICATION`,
`SMTP_ENABLE_STARTTLS_AUTO`, and `MAILER_SENDER_EMAIL`. Match TLS settings to the
provider. Do not launch invitations or password recovery without testing delivery.

GitHub builds supply `RAILWAY_GIT_COMMIT_SHA`; the Dockerfile records this in
`.git_sha`. For CLI uploads or local worktree builds, explicitly supply the
source commit as that build argument/variable because `.git` is not portable.

## Web settings

- Root directory: repository root.
- Pre-deploy command: `env POSTGRES_STATEMENT_TIMEOUT=600s bundle exec rails db:chatwoot_prepare`
- Start command: `sh -c 'exec bundle exec puma -C config/puma.rb -b tcp://0.0.0.0:$PORT'`
- Healthcheck path: `/health`; timeout: 300 seconds.
- Keep the service always running; disable sleeping/serverless mode.
- Use Railway's injected `PORT`; Puma reads it from the environment.

The pre-deploy task initializes a fresh database or migrates an existing one.
Only Web runs this task. HTTP health probes are excluded from Rails's HTTPS
redirect; other application routes retain forced HTTPS and secure cookies.

## Worker settings

- Start command: `bundle exec sidekiq -C config/sidekiq.yml`
- No pre-deploy command, HTTP healthcheck or public domain.
- Keep the service always running; disable sleeping/serverless mode.
- Deploy the worker after the first successful Web database preparation.

Start with room for roughly 2 GB each for Web and Worker, then size from actual
memory and queue metrics. Set usage alerts in Railway. Subsequent releases should
prepare the database through Web before deploying the matching worker revision.

## Domain and launch checks

1. Add `bolodesk.askpocket.in` to Web's custom domains. Configure the exact DNS
   records Railway supplies, and wait for certificate issuance.
2. Complete the initial administrator onboarding. Keep public account signup
   disabled; see the pilot branding guide for existing database branding updates.
3. Verify `/health`, login, BoloDesk branding, an inbox conversation, a background
   job and outbound email.
4. Upload an attachment, redeploy both application services, and verify the
   attachment is still downloadable and its preview works.
5. Verify database backup scheduling and perform a restore into an isolated
   database before accepting production customer data. Configure uptime monitoring
   separately: Railway's deployment healthcheck is not ongoing monitoring.

Application rollback does not roll back database migrations. Check schema
compatibility before rolling back either service; restore data only through a
deliberate recovery procedure.

## References

- [Docker builds](https://docs.railway.com/builds/dockerfiles)
- [Database responsibilities](https://docs.railway.com/databases)
- [Pre-deploy commands](https://docs.railway.com/deployments/pre-deploy-command)
- [Storage buckets](https://docs.railway.com/storage-buckets)
- [Healthchecks](https://docs.railway.com/deployments/healthchecks)
- [Config as Code deprecation](https://docs.railway.com/config-as-code/reference)
