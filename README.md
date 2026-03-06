# Supabase Self-Hosted on EC2 — RDS & Services Setup Guide

> This document covers everything done to configure AWS RDS PostgreSQL and set up the 4 Supabase services (Kong, GoTrue, PostgREST, postgres-meta) using Podman Quadlets on Ubuntu EC2. Use this as a from-scratch reference.

---

## Architecture Overview

```
Internet → Kong (port 8000) → GoTrue      (auth.v1)    → RDS PostgreSQL
                             → PostgREST  (rest/v1)    → RDS PostgreSQL
                             → Meta       (/pg)         → RDS PostgreSQL
```

**Services:**
- `supabase-kong` — API gateway, handles routing + API key auth
- `supabase-auth` — GoTrue, handles authentication (email, OAuth, JWT)
- `supabase-rest` — PostgREST, exposes your database as a REST API
- `supabase-meta` — postgres-meta, database introspection for PostgREST

**Infrastructure:**
- EC2: Ubuntu, t3.large, 50GB EBS, Podman 4.9.3
- RDS: PostgreSQL 15.17, db.t4g.micro, same VPC as EC2, no public access

---

## Part 1 — RDS Creation (AWS Console)

1. Go to **RDS → Create Database**
2. Engine: **PostgreSQL 15.17-R1** (do NOT use PG16, PG17, or PG18 — compatibility issues with Supabase migrations)
3. Template: Free tier or Production
4. Instance class: `db.t4g.micro` (testing) or `db.t3.medium` (production)
5. Storage: 20GB gp3, enable autoscaling
6. **VPC: same VPC as your EC2** — critical
7. **Public access: NO** — critical
8. VPC Security Group: allow inbound port `5432` from EC2's security group ID only
9. Initial database name: `postgres`
10. Note down: endpoint, master username, master password

---

## Part 2 — Connect to RDS from EC2

Install psql client on EC2:

```bash
sudo apt-get install -y postgresql-client
```

Connect using master credentials:

```bash
psql "host=YOUR_RDS_ENDPOINT port=5432 dbname=postgres user=YOUR_MASTER_USER sslmode=require"
```

---

## Part 3 — Enable PostgreSQL Extensions

Run inside psql after connecting:

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

> **Note:** `pgjwt` is NOT available on AWS RDS. This is fine — newer GoTrue handles JWT in the application layer, not the database.

---

## Part 4 — Create Required Database Roles

These roles are used by the 4 Supabase services to connect to RDS. Use a strong password without special characters (`@`, `£`, `^`, `#` etc.) to avoid URI encoding issues in connection strings.

```sql
-- Roles PostgREST uses (no login, just permission groups)
CREATE ROLE anon NOLOGIN NOINHERIT;
CREATE ROLE authenticated NOLOGIN NOINHERIT;
CREATE ROLE service_role NOLOGIN NOINHERIT BYPASSRLS;

-- Login role PostgREST connects as
CREATE ROLE authenticator NOINHERIT LOGIN PASSWORD 'YOUR_PASSWORD';
GRANT anon TO authenticator;
GRANT authenticated TO authenticator;
GRANT service_role TO authenticator;

-- Login role GoTrue connects as
CREATE ROLE supabase_auth_admin NOINHERIT CREATEROLE LOGIN PASSWORD 'YOUR_PASSWORD';

-- Login role postgres-meta connects as
CREATE ROLE supabase_admin LOGIN PASSWORD 'YOUR_PASSWORD';
GRANT ALL PRIVILEGES ON DATABASE postgres TO supabase_admin;
```

---

## Part 5 — Grant Master User Membership (RDS Specific)

RDS master user is not a true superuser, so we need to grant it membership to assign schema ownership:

```sql
GRANT supabase_auth_admin TO YOUR_MASTER_USERNAME;
```

Replace `YOUR_MASTER_USERNAME` with your RDS master username (e.g. `supabasepostgres`).

---

## Part 6 — Create Required Schemas

```sql
-- Auth schema owned by supabase_auth_admin (GoTrue will create its tables here)
CREATE SCHEMA IF NOT EXISTS auth AUTHORIZATION supabase_auth_admin;

-- Extensions schema
CREATE SCHEMA IF NOT EXISTS extensions;

-- Grant permissions
GRANT ALL ON SCHEMA auth TO supabase_auth_admin;
GRANT ALL ON SCHEMA public TO supabase_auth_admin;
GRANT ALL ON SCHEMA extensions TO supabase_auth_admin;
GRANT USAGE ON SCHEMA public TO anon, authenticated, service_role;
GRANT ALL PRIVILEGES ON DATABASE postgres TO supabase_admin;
```

---

## Part 7 — Grant Permissions on Tables and Sequences

```sql
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA auth TO supabase_auth_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA auth TO supabase_auth_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO supabase_auth_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO supabase_auth_admin;

-- Cover future tables created by GoTrue migrations
ALTER DEFAULT PRIVILEGES IN SCHEMA auth GRANT ALL ON TABLES TO supabase_auth_admin;
ALTER DEFAULT PRIVILEGES IN SCHEMA auth GRANT ALL ON SEQUENCES TO supabase_auth_admin;
```

---

## Part 8 — Grant Additional Role Permissions

```sql
GRANT ALL PRIVILEGES ON DATABASE postgres TO supabase_auth_admin;
ALTER ROLE supabase_auth_admin CREATEDB CREATEROLE;
```

---

## Part 9 — Set Search Path for GoTrue Role

This is critical — GoTrue looks for its tables without schema prefix. This tells PostgreSQL to look in `auth` schema by default for this role:

```sql
ALTER ROLE supabase_auth_admin SET search_path TO auth, public;
```

---

## Part 10 — Fix GoTrue Migration Compatibility (PG15 Issue)

GoTrue v2.151.0 has a migration (`20221208132122`) that uses `uuid = text` comparison which fails on PostgreSQL 15. Since the table is empty on fresh install, just mark the migration as done:

```sql
INSERT INTO auth.schema_migrations (version) VALUES ('20221208132122')
ON CONFLICT DO NOTHING;
```

> **Why:** GoTrue tracks completed migrations in `auth.schema_migrations`. By inserting this version, GoTrue skips the broken migration on startup. The migration was a data backfill for old data that doesn't exist on a fresh install anyway.

---

## Part 11 — Verify Everything

```sql
-- Check all roles exist
\du

-- Check schemas exist
\dn

-- Check auth tables after GoTrue first start
\dt auth.*
```

Expected auth tables after GoTrue runs:

```
auth.audit_log_entries
auth.flow_state
auth.identities
auth.instances
auth.mfa_amr_claims
auth.mfa_challenges
auth.mfa_factors
auth.one_time_tokens
auth.refresh_tokens
auth.saml_providers
auth.saml_relay_states
auth.schema_migrations
auth.sessions
auth.sso_domains
auth.sso_providers
auth.users
```

---

## Part 12 — Environment Variables (.env)

Location on EC2: `/etc/supabase/.env`

Permissions: `sudo chmod 600 /etc/supabase/.env`

```bash
# ============================
# POSTGRES / RDS
# ============================
POSTGRES_HOST=YOUR_RDS_ENDPOINT
POSTGRES_PORT=5432
POSTGRES_DB=postgres
POSTGRES_PASSWORD=YOUR_PASSWORD

# PostgREST — connects as 'authenticator' role
PGRST_DB_URI=postgres://authenticator:YOUR_PASSWORD@YOUR_RDS_ENDPOINT:5432/postgres?sslmode=require
PGRST_DB_SCHEMAS=public,extensions

# GoTrue — connects as 'supabase_auth_admin' role
GOTRUE_DB_DATABASE_URL=postgres://supabase_auth_admin:YOUR_PASSWORD@YOUR_RDS_ENDPOINT:5432/postgres?sslmode=require

# ============================
# JWT KEYS (generate these)
# ============================
JWT_SECRET=your-super-secret-jwt-token-minimum-32-chars
GOTRUE_JWT_SECRET=your-super-secret-jwt-token-minimum-32-chars
ANON_KEY=eyJ...      # JWT signed with JWT_SECRET, role: anon
SERVICE_ROLE_KEY=eyJ...  # JWT signed with JWT_SECRET, role: service_role

# ============================
# SITE / API URLs
# ============================
SITE_URL=https://yourapp.com
GOTRUE_SITE_URL=https://yourapp.com
API_EXTERNAL_URL=https://api.yourapp.com
ADDITIONAL_REDIRECT_URLS=

# ============================
# GOOGLE OAUTH
# ============================
GOOGLE_ENABLED=true
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret

# ============================
# FACEBOOK OAUTH
# ============================
FACEBOOK_ENABLED=true
FACEBOOK_CLIENT_ID=your-facebook-app-id
FACEBOOK_CLIENT_SECRET=your-facebook-app-secret

# ============================
# SMTP via Resend
# ============================
SMTP_HOST=smtp.resend.com
SMTP_PORT=465
SMTP_USER=resend
SMTP_PASS=re_your-resend-api-key
SMTP_ADMIN_EMAIL=no-reply@yourapp.com
SMTP_SENDER_NAME=YourApp

# ============================
# META SERVICE
# ============================
PG_META_DB_HOST=YOUR_RDS_ENDPOINT
PG_META_DB_PORT=5432
PG_META_DB_NAME=postgres
PG_META_DB_USER=supabase_admin
PG_META_DB_PASSWORD=YOUR_PASSWORD
```

> **Important:** Passwords in connection string URIs (`PGRST_DB_URI`, `GOTRUE_DB_DATABASE_URL`) must NOT contain special characters like `@`, `£`, `^`, `#`, `(`, `)`. Use only letters, numbers, and underscores/hyphens.

---

## Part 13 — Kong Configuration

Location: `/etc/supabase/volumes/kong/kong.yml`

The `ANON_KEY` and `SERVICE_ROLE_KEY` must be hardcoded directly in this file — Kong 2.8.1 does not support environment variable substitution in declarative config.

Replace `${ANON_KEY}` and `${SERVICE_ROLE_KEY}` with actual JWT values:

```bash
sudo sed -i 's|\${ANON_KEY}|YOUR_ACTUAL_ANON_KEY|' /etc/supabase/volumes/kong/kong.yml
sudo sed -i 's|\${SERVICE_ROLE_KEY}|YOUR_ACTUAL_SERVICE_ROLE_KEY|' /etc/supabase/volumes/kong/kong.yml
```

Also remove `hide_groups_not_found: true` from all ACL plugin configs — not supported in Kong 2.8.1.

---

## Part 14 — Starting the Services

```bash
# Reload systemd to pick up quadlet files
sudo systemctl daemon-reload

# Start in this order
sudo systemctl start supabase-meta
sudo systemctl start supabase-auth
sudo systemctl start supabase-rest
sudo systemctl start supabase-kong

# Check all 4 are running
sudo systemctl status supabase-meta supabase-auth supabase-rest supabase-kong --no-pager
```

---

## Part 15 — Testing

```bash
# Set anon key
ANON_KEY=$(sudo grep ^ANON_KEY /etc/supabase/.env | cut -d= -f2)

# Test auth health
curl -s http://localhost:8000/auth/v1/health -H "apikey: $ANON_KEY"

# Test signup
curl -s -X POST http://localhost:8000/auth/v1/signup \
  -H "apikey: $ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"TestPass123!"}'

# Test REST API
curl -s http://localhost:8000/rest/v1/ -H "apikey: $ANON_KEY"
```

Successful signup returns a full user JSON object with `id`, `email`, `role`, and `identities`.

---

## Troubleshooting Reference

| Error | Cause | Fix |
|---|---|---|
| `no pg_hba.conf entry, no encryption` | `sslmode=disable` rejected by RDS | Use `sslmode=require` in connection strings |
| `password authentication failed` | Wrong password in `.env` or role password not set | Run `ALTER ROLE ... PASSWORD` in psql |
| `relation "identities" does not exist` | GoTrue searching wrong schema | Run `ALTER ROLE supabase_auth_admin SET search_path TO auth, public` |
| `operator does not exist: uuid = text` | PG15 migration compatibility | Insert migration version into `auth.schema_migrations` |
| `permission denied for schema public` | Missing grants | Run GRANT commands in Part 6 and 7 |
| `hide_groups_not_found: unknown field` | Kong 2.8.1 doesn't support this field | Remove from all ACL configs in `kong.yml` |
| `DNS resolution failed` in Kong | Cached old container IP | Restart Kong after restarting any other service |
| `No API key found in request` | Kong reading `${ANON_KEY}` literally | Hardcode actual JWT values in `kong.yml` |

---

## Service Image Versions

| Service | Image | Version |
|---|---|---|
| Kong | `docker.io/kong` | `2.8.1` |
| GoTrue | `docker.io/supabase/gotrue` | `v2.151.0` |
| PostgREST | `docker.io/postgrest/postgrest` | `v12.2.0` |
| postgres-meta | `docker.io/supabase/postgres-meta` | `v0.84.2` |

---

## Key Notes

- **RDS PostgreSQL version:** Use 15.x only. PG16+ and PG18 have compatibility issues with Supabase migrations and extensions.
- **No SUPERUSER on RDS:** AWS RDS doesn't allow `ALTER ROLE ... SUPERUSER`. Use `CREATEDB CREATEROLE` instead.
- **pgjwt not available on RDS:** Not needed — GoTrue handles JWT at the application layer.
- **Kong DNS caching:** If you restart GoTrue or PostgREST, always restart Kong too — it caches container IPs.
- **Passwords in URIs:** Never use special characters in passwords that appear in connection string URIs.
- **All state in RDS:** Containers are fully stateless. All auth data lives in RDS. Containers can be destroyed and recreated freely.
