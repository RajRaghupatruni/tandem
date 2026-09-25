# Tandem production runbook

This runbook deploys the application on one same-origin Render Web Service. PostgreSQL is
Neon, photos are private Backblaze B2 objects, and the web process serves the compiled React
SPA. Do not put real credentials in this file, Git, Vite variables, or a browser bundle.

## 0. Values to decide first

Choose these values before opening provider dashboards:

| Value | Example shape |
| --- | --- |
| Application URL | `https://tandem.example.com` |
| OAuth callback | `https://tandem.example.com/auth/google/callback` |
| Runtime role | `tandem_app` |
| Migration role | `tandem_migrator` |
| B2 bucket | a new private bucket, for example `tandem-production-media` |
| B2 region | the region shown by Backblaze, for example `us-west-004` |

Use the final HTTPS hostname everywhere. Do not use the Render preview URL as the permanent
OAuth callback if a custom domain will be used.

## 1. Neon PostgreSQL

1. Create a Neon project in the desired region and create a production database. Keep Neon’s
   owner/bootstrap connection string private.
2. Use the Neon SQL Editor while connected as the project owner/bootstrap role. Replace the
   two password placeholders in memory only; do not paste them into this document or Git:

```sql
CREATE ROLE tandem_migrator LOGIN PASSWORD '<LONG-RANDOM-MIGRATION-PASSWORD>'
  NOSUPERUSER NOCREATEDB NOCREATEROLE NOBYPASSRLS;
CREATE ROLE tandem_app LOGIN PASSWORD '<LONG-RANDOM-RUNTIME-PASSWORD>'
  NOSUPERUSER NOCREATEDB NOCREATEROLE NOBYPASSRLS;
CREATE ROLE tandem_rls_owner NOLOGIN
  NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION BYPASSRLS;
GRANT tandem_rls_owner TO tandem_migrator;

-- Replace tandem_db with the actual Neon database name.
GRANT CONNECT ON DATABASE tandem_db TO tandem_migrator, tandem_app;
ALTER DATABASE tandem_db OWNER TO tandem_migrator;
GRANT USAGE ON SCHEMA public, app TO tandem_migrator, tandem_app;

-- These are useful when bootstrapping an already-migrated database. The migrations also
-- apply the narrower table/function grants as each table is created.
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO tandem_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO tandem_app;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA app TO tandem_app;
ALTER DEFAULT PRIVILEGES FOR ROLE tandem_migrator IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO tandem_app;
ALTER DEFAULT PRIVILEGES FOR ROLE tandem_migrator IN SCHEMA public
  GRANT USAGE, SELECT ON SEQUENCES TO tandem_app;
```

3. Confirm the flags and identities before deploying:

```sql
SELECT rolname, rolsuper, rolbypassrls, rolcreaterole, rolcreatedb
FROM pg_roles
WHERE rolname IN ('tandem_migrator', 'tandem_app')
ORDER BY rolname;

-- Run while connected with the runtime URL.
SELECT current_user;
```

`tandem_app` must show `rolsuper = false`, `rolbypassrls = false`, and `current_user` must be
`tandem_app`. Never make the runtime role an owner or superuser to work around a grant error.
The application fails closed if its production connection is not the configured non-bypass-RLS
role. If a Neon-hosted role is not allowed to create roles or change database ownership, use
the Neon project owner/admin connection to perform the commands or ask Neon support for the
equivalent role operation; do not weaken `FORCE ROW LEVEL SECURITY`.

## 2. Backblaze B2 private media

1. Create a new B2 bucket and set it to **Private**.
2. In Application Keys, create a bucket-restricted key with only the capabilities needed for
   this service: read files, write files, delete files, and list files if the B2 console requires
   it for a bucket-restricted S3 key. Do not use the master application key.
3. Record the key ID and application key once. Tandem maps them to the S3 access key and secret.
4. Set the S3 endpoint to the B2 S3 endpoint for the bucket’s region, normally:
   `https://s3.<B2-REGION>.backblazeb2.com`. Set `S3_REGION` to the B2 region exactly as shown by
   Backblaze. Do not make the bucket public and do not add a CDN for v1.

Tandem uploads through FastAPI, validates and re-encodes images to WebP, stores opaque
`media/<uuid>.webp` keys, and issues five-minute presigned reads only after membership
authorization. The browser never receives B2 credentials. A presigned-read failure leaves the
memory visible with the Tandem-owned placeholder treatment rather than crashing Timeline.

## 3. Google OAuth / OpenID Connect

1. In Google Cloud Console, create or select a project and configure the OAuth consent screen.
   Use the production app name and support/contact information. Keep the app restricted to the
   intended testers until the real-user list is ready.
2. Create a **Web application** OAuth client.
3. Add the exact authorized JavaScript origin:
   `https://tandem.example.com`
4. Add the exact authorized redirect URI:
   `https://tandem.example.com/auth/google/callback`
   There is no trailing slash and no localhost URI in the production client.
5. Store the client ID and secret in Render only. Tandem requests `openid email profile`, accepts
   only a verified Google subject/email, stores no Google access token, and creates a server-side
   session whose cookie is HttpOnly, Secure, SameSite=Lax, and opaque.

The invite reference is retained in browser session storage across the OAuth redirect. Verify the
invite flow before distributing an invitation. Do not put an invitation reference in logs or an
email body.

## 4. TMDb

1. Create or use the TMDb account and generate an API Read Access Token (Bearer token), not a
   browser-exposed key.
2. Confirm the service’s use complies with TMDb terms and includes any required attribution in
   the product/legal copy.
3. Put the token in Render as `TMDB_API_TOKEN`. The backend adapter owns the token, limits the
   query, times out, normalizes responses, and returns a controlled provider error.

## 5. Geoapify

1. Create a Geoapify project and API key with the minimum place-search permissions needed.
2. Restrict the key to the production service where Geoapify permits it; it is still kept server-
   side because the frontend must not expose provider credentials.
3. Put it in Render as `GEOAPIFY_API_KEY`. Do not configure or reintroduce Foursquare.

## 6. Resend (deferred for v1)

Outbound email is intentionally disabled at the Tandem v1 launch. Do not add Resend secrets to
the Render web service or hourly cron job while `EMAIL_DELIVERY_ENABLED=false`; the cron still
runs because it generates in-app On This Day notifications. The email preference, outbox, and
adapter remain in the codebase for a later re-enable.

1. In Resend, add the sending domain and complete its DNS verification (SPF/DKIM; publish the
   recommended DMARC policy for the domain).
2. Create a restricted sending API key. Set `RESEND_FROM_EMAIL` to a verified address such as
   `Tandem <hello@example.com>`.
3. Tandem emails contain only a generic “a memory is waiting” message and a link to
   `APPLICATION_URL`; they do not contain notes, titles, photos, or location data.
4. Test with a designated test recipient first. Keep production and development Resend API keys
   and sender domains separate. A development environment must not hold the production key.

## 7. Render Web Service

1. Push this branch or merge it to the intended deployment branch. In Render choose **New →
   Blueprint**, select the repository and confirm the root `render.yaml`.
2. Confirm the Blueprint creates only:
   - `tandem-web`, a Docker web service on the paid smallest always-on 512 MB plan (`starter`);
   - `tandem-anniversary`, a Docker cron job on the smallest cron plan.

   It must not create Render Postgres, Redis/Key Value, a second public frontend, or a separate
   backend service.
3. In the `tandem-web` Environment page, set these values. Render supplies `PORT` itself.

```text
APP_ENV=production
LOG_LEVEL=INFO
DATABASE_RUNTIME_ROLE=tandem_app
DATABASE_RLS_OWNER_ROLE=tandem_rls_owner
DATABASE_URL=postgresql+psycopg://tandem_app:<password>@<neon-host>/<database>?sslmode=require&channel_binding=require
MIGRATION_DATABASE_URL=postgresql+psycopg://tandem_migrator:<password>@<neon-host>/<database>?sslmode=require&channel_binding=require
APPLICATION_URL=https://tandem.example.com
GOOGLE_CLIENT_ID=<client-id>
GOOGLE_CLIENT_SECRET=<client-secret>
GOOGLE_REDIRECT_URI=https://tandem.example.com/auth/google/callback
OAUTH_SESSION_SECRET=<dedicated-random-secret>
TMDB_API_TOKEN=<tmdb-bearer-token>
GEOAPIFY_API_KEY=<geoapify-key>
S3_ENDPOINT_URL=https://s3.<b2-region>.backblazeb2.com
S3_BUCKET=<private-b2-bucket>
S3_ACCESS_KEY_ID=<bucket-key-id>
S3_SECRET_ACCESS_KEY=<bucket-application-key>
S3_REGION=<b2-region>
EMAIL_DELIVERY_ENABLED=false
```

Leave `ALLOW_NON_PRODUCTION_EMAILS` unset/false outside a deliberately isolated email test. To
later enable outbound email, set `EMAIL_DELIVERY_ENABLED=true` and add both `RESEND_API_KEY` and
`RESEND_FROM_EMAIL` (or the compatibility alias `RESEND_FROM_ADDRESS`) for a verified sending
domain; production configuration rejects the launch if either credential is missing. The adapter
refuses delivery in development/test by default, even if a Resend key is accidentally present.

Use Render’s secret input for every angle-bracket secret. `MIGRATION_DATABASE_URL` is needed by
the pre-deploy command and is not used by the web runtime. The service’s runtime calls
`assert_runtime_role` against `DATABASE_URL` during startup.

Before applying migration `0015_rls_helper_role`, provision the helper owner through a Neon
administrative connection because `tandem_migrator` intentionally has `NOCREATEROLE`:

The role must be created as follows (the literal `NOLOGIN` is required), and only the migration
role may be granted membership:

```sql
CREATE ROLE tandem_rls_owner NOLOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE NOREPLICATION BYPASSRLS;
GRANT tandem_rls_owner TO tandem_migrator;
```

Do not grant this role to `tandem_app`, do not give it a password, and do not make either login
role `BYPASSRLS`. Migration `0015` verifies these attributes, grants the helper only its bounded
source-table privileges, transfers the named helper functions, and revokes temporary schema
`CREATE` afterward.

`OAUTH_SESSION_SECRET` is a separate randomly generated secret used only by Authlib’s short-lived
`tandem_oauth_session` cookie. It must not reuse the Google client secret, either database URL
credential, or any Tandem authentication token. The web service requires it at startup; the
anniversary cron does not construct the OAuth web middleware and does not need this variable.

4. Add the custom domain, complete DNS, and wait for Render TLS. Keep Render’s automatic HTTPS
   redirect enabled. Do not add an application HTTP→HTTPS redirect that could loop behind TLS
   termination.
5. Confirm the service uses the Dockerfile at the repository root, has health check path
    `/api/health`, and starts with the image command. The Uvicorn process trusts forwarded
    protocol/client headers only from private infrastructure CIDRs (`127.0.0.1`, `10/8`,
    `172.16/12`, and `192.168/16`), never from arbitrary public callers. Confirm the Render
    proxy reaches the service from one of those ranges before deployment; if Render documents a
    different private proxy range, set the image's forwarded-header allowlist to that bounded
    range. No Vite server is involved in production.

## 8. Render Cron Job

The Blueprint schedules the existing bounded worker hourly with:

```text
python -m app.workers.anniversary
```

Set the cron job’s `DATABASE_URL`, `DATABASE_RUNTIME_ROLE`, `APPLICATION_URL`, and
`EMAIL_DELIVERY_ENABLED=false`. The other provider/B2 variables are included so the strict
production settings contract remains identical; the worker does not call those providers or
Resend at launch.
Do not set `MIGRATION_DATABASE_URL` on the cron job. The worker uses `tandem_app` and its explicit
`app.worker_mode` RLS policies. Its PostgreSQL unique idempotency key and claim-before-send
transaction protect against duplicate generation; a process crash after Resend accepts a message
can still produce an at-least-once duplicate.

The hourly cadence is sufficient for this small audience. The worker checks each user’s IANA
timezone and preferred delivery hour; it is not tied to the Render server timezone.

## 9. Migrations and deploy order

Render runs this pre-deploy command before routing traffic to the new web image:

```text
alembic upgrade head
```

It uses `MIGRATION_DATABASE_URL`, so migration logs appear in the Render deploy log and a failed
migration blocks the deployment. Confirm the migration head after the first deploy:

```text
alembic current
alembic check
```

The V1 final migration head is `0016_v1_completion`. Never run migrations from FastAPI lifespan,
and never point `MIGRATION_DATABASE_URL` at the runtime role. Schema downgrades are not a routine
rollback: future migrations may be destructive and application code is not necessarily backward
compatible. For a bad application image, redeploy the previous image/commit without downgrading.
For a bad schema change, stop writes, take a backup, and execute a reviewed forward repair or a
carefully tested downgrade with the migration identity.

## 10. First login and first Tandem

1. Open the final application URL and choose **Continue with Google**.
2. The first verified Google account becomes a normal user; there is no hidden admin account.
3. Create the first Tandem and record its name/timezone.
4. Invite User B by email from the Tandem flow. User B must sign in with the exact invited email.
5. Keep User C authenticated but outside the Tandem for the authorization smoke test.

## 11. Verification after deployment

Run these checks with the final hostname:

```text
GET /healthz                         -> 200 {"status":"ok"}
GET /api/health                     -> 200 and database="ok"
GET /api/readyz                     -> 200 and database="ok"
GET /timeline                       -> 200 HTML containing the SPA shell
GET /assets/<known-hash>.js         -> 200 with long immutable cache
GET /assets/does-not-exist.js       -> 404 (never SPA index)
GET /api/not-a-route                -> 404 JSON (never SPA index)
```

Also verify the response includes `X-Request-ID`, CSP, Referrer-Policy, X-Content-Type-Options,
frame protection, and Permissions-Policy. Production unsafe requests without the exact app
Origin/Referer receive 403. Check Render logs for JSON request records with route, status,
latency, provider classification, and correlation ID but no session, invitation, OAuth, photo,
note, or API-key values.

Run the repeatable user journey in [PRODUCTION_SMOKE_CHECKLIST.md](PRODUCTION_SMOKE_CHECKLIST.md).

## 12. Backup and recovery

PostgreSQL and B2 are separate recovery domains. A `pg_dump` contains metadata, memberships,
memories, and object keys, but not photo bytes. A B2 bucket copy contains photo bytes, but is not
useful alone without the PostgreSQL metadata and authorization state.

At least weekly, and before a migration, make a protected custom-format dump using the migration
identity (never print the URL):

```powershell
pg_dump --format=custom --no-owner --file=tandem-YYYYMMDD.dump "$env:MIGRATION_DATABASE_URL"
```

Verify that the dump file exists, is encrypted or stored in an access-controlled backup location,
and can be listed with `pg_restore --list`. Keep at least one copy outside the Render workspace.
To restore to a disposable Neon branch first:

```powershell
createdb --maintenance-db="$env:MIGRATION_DATABASE_URL" restore_check
pg_restore --no-owner --dbname="$env:RESTORE_DATABASE_URL" tandem-YYYYMMDD.dump
```

Use Neon’s branch/restore controls for point-in-time recovery when available, then run `alembic
upgrade head` and verify RLS before cutover. Do not restore a dump directly over production without
an incident decision and a new pre-restore backup.

For B2, use the Backblaze B2 CLI or S3-compatible tooling to copy the private bucket to a separate
private recovery bucket on a regular cadence, for example `b2 sync b2://<bucket> <backup-dir>`.
Test restoring a sample object and confirm the application can create a fresh presigned URL. B2’s
durability is not the same thing as an independent point-in-time backup; deletions and bad writes
must be covered by retention/versioning or a separate bucket policy chosen in the B2 console.

## 13. Privacy lifecycle audit and known gap

Implemented now:

- members can hard-delete memories; media objects are deleted before metadata is committed;
- members can delete individual photos;
- owners can remove members and members can leave when ownership rules permit;
- each user can disable On This Day resurfacing in Settings; outbound email is deferred for v1;
- non-members receive the same 404-style Tandem boundary and cannot read private media.

Not implemented yet and therefore a P0 follow-up before promising data-portability compliance:

- there is no self-service account export endpoint/UI;
- there is no self-service account deletion endpoint/UI that safely removes the user’s sessions,
  preferences, memberships, invitations, authored memories, and corresponding B2 objects;
- leave/remove and account deletion are currently API capabilities rather than polished Settings
  UI flows.

This is deliberately recorded rather than hidden. Memory and photo deletion are available now,
but account export/deletion remains the exact P0 gap to close before a formal privacy promise.

## 14. Cost expectation

At current published prices, Render’s smallest paid always-on 512 MB web plan is about **$7/month**;
the hourly 512 MB cron is usage-priced and is typically well under $1/month for a short run, but
will vary with runtime. Resend can remain on its free transactional tier at this volume if its
limits fit, and B2’s first 10 GB is currently free with usage-based charges thereafter. Neon is
usage-based/plan-dependent, so set a spending alert and confirm the selected Neon plan before
launch. A reasonable initial envelope is **about $7–$15/month plus Neon usage**, excluding a custom
domain and any paid email/storage overage. Recheck provider pricing before purchase.
