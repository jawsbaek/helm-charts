# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Helm chart for deploying Umami (a privacy-focused web analytics platform) on Kubernetes. The chart packages Umami v3.x with PostgreSQL database dependency and provides extensive configuration options for Kubernetes deployment.

**Note:** Chart version 8.0.0+ uses Umami v3, which only supports PostgreSQL. MySQL support was removed in Umami v3.

## Common Commands

### Testing and Validation
- `helm lint .` - Lint the chart for syntax and validation errors
- `helm template my-release . -f values.yaml` - Render templates locally to debug
- `helm template my-release . --debug` - Debug template rendering with verbose output

### Installation and Upgrades
- `helm install my-release . -f values.yaml` - Install the chart locally
- `helm upgrade my-release . -f values.yaml` - Upgrade an existing release
- `helm uninstall my-release` - Remove the release

### Dependency Management
- `helm dependency update` - Update chart dependencies (PostgreSQL subchart)
- `helm dependency list` - List chart dependencies

### Documentation
- Edit `README.md.gotmpl` (not `README.md` directly) - the README.md is generated from this template
- Use `helm-docs` or similar tooling to regenerate README.md from the template

## Architecture

### Chart Structure

The chart follows standard Helm conventions with these key components:

**Core Resources** (in `templates/`):
- `deployment.yaml` - Main Umami application deployment
- `service.yaml` - Service exposing Umami on port 3000
- `ingress.yaml` - Optional Ingress for external access
- `route.yaml` - Optional Gateway API HTTPRoute (BETA feature)
- `hpa.yaml` - Horizontal Pod Autoscaler configuration

**Configuration Resources**:
- `app-secret.yaml` - Secret for Umami's APP_SECRET (random string for unique value generation)
- `db-secret.yaml` - Constructed database URL secret (unless using existing secret)
- `configmap.yaml` - Custom tracking script ConfigMap (optional)
- `serviceaccount.yaml` - ServiceAccount for the deployment

**Migration**:
- `job.yaml` - Pre-install/pre-upgrade Job for v1→v2 database migration (enabled via `umami.migration.v1v2.enabled`)

### Database Architecture

**As of chart version 8.0.0 (Umami v3): PostgreSQL only**

The chart supports two database configurations:

1. **Bundled PostgreSQL** (default): Bitnami PostgreSQL subchart (`postgresql.enabled: true`)
2. **External PostgreSQL**: User-managed PostgreSQL database via `externalDatabase.*` values

**MySQL Removed:** Umami v3 removed MySQL support. The `mysql.*` values are deprecated and non-functional.

Database connection handling:
- Helper functions in `_helpers.tpl` construct the database URL for PostgreSQL
- The URL format is: `postgresql://<username>:<password>@<hostname>:<port>/<database>`
- Generated or existing secret provides `DATABASE_URL` environment variable to the deployment
- Database migrations from v2 to v3 run automatically on first pod startup

### Template Helpers (`_helpers.tpl`)

Critical helper functions that construct configuration:

- `umami.database.*` - Suite of functions that abstract database connection details for PostgreSQL or external databases
  - `umami.database.hostname` - Resolves to subchart service name or external hostname
  - `umami.database.port` - Database port from subchart or external config
  - `umami.database.url` - Constructs full PostgreSQL connection string
  - `umami.database.secretName` - Returns existing secret name or generates one
- `umami.appSecret.secretName` - Resolves app secret from existing or generates
- Standard helpers: `umami.fullname`, `umami.labels`, `umami.selectorLabels`, etc.

These helpers ensure the chart works consistently regardless of which PostgreSQL database option is chosen.

### Environment Variables

The deployment configures Umami via environment variables (see `templates/deployment.yaml:41-112`):
- Core: `DATABASE_URL`, `APP_SECRET`, `PORT`, `HOSTNAME`
- Features: `DISABLE_LOGIN`, `DISABLE_TELEMETRY`, `CLOUD_MODE`, `TRACKER_SCRIPT_NAME`
- Debugging: `LOG_QUERY`
- Additional custom variables via `extraEnv` array

Note: `DISABLE_LOGIN` can be removed from the deployment via `umami.removeDisableLoginEnv: true` (default) as it caused issues in some setups.

## Important Notes

### Database Password Handling
- Passwords must be set in `values.yaml` under the appropriate section: `postgresql.auth.password` or `externalDatabase.auth.password`
- The chart constructs the database URL including the password and stores it in a Kubernetes Secret
- Never commit actual passwords to version control; use `--set` flags or separate values files

### Subchart Version Updates
Major version updates of the PostgreSQL subchart may require migration steps. See the "Upgrading the Chart" section in README.md for subchart-specific upgrade notes.

### Custom Tracking Script
Enable via `umami.customScript.enabled: true` to mount a custom JavaScript tracking script. The script is stored in a ConfigMap and mounted at `/app/public/script.js` (configurable via `mountPath`).

### Gateway API Support (BETA)
The `route.yaml` template provides Gateway API HTTPRoute support, but this is marked as BETA and may change. It coexists with traditional Ingress support.

## Upgrading from v2 to v3

### Chart v7.x to v8.x (Umami v2 to v3)

This is a **breaking change** upgrade:

1. **MySQL users must migrate to PostgreSQL first**
   - Follow: https://docs.umami.is/docs/guides/migrate-mysql-postgresql
   - Complete migration before upgrading chart

2. **Backup your database before upgrading**
   ```bash
   pg_dump -h <host> -U <user> <database> > umami_backup.sql
   ```

3. **Database schema migration is automatic**
   - v2→v3 migrations run automatically on first startup
   - Monitor pod logs during first boot after upgrade
   - Check for migration success: `kubectl logs <pod-name> | grep migration`

4. **MySQL subchart removal**
   - The `mysql` dependency is completely removed from Chart.yaml
   - All `mysql.*` values are ignored
   - Clean up old MySQL resources manually if needed

### Rollback Procedure

If upgrade fails:
```bash
# Restore database backup
psql -h <host> -U <user> <database> < umami_backup.sql

# Rollback helm release
helm rollback <release-name>
```

## Chart Values Structure

Key value sections:
- `image.*` - Container image configuration (registry: ghcr.io, repository: umami-software/umami, tag: v3.0.3)
- `umami.*` - Umami application settings (all the `DISABLE_*`, `FORCE_SSL`, etc. flags)
- `umami.appSecret.*` - App secret configuration (existing or generated)
- `umami.migration.*` - Database migration job control
- `postgresql.*` - PostgreSQL subchart configuration (deprecated: `mysql.*`)
- `externalDatabase.*` - External database connection details
- `database.*` - Secret configuration for database URL
- `ingress.*` / `route.*` - External access configuration
- Standard Kubernetes: `replicaCount`, `resources`, `nodeSelector`, `tolerations`, `affinity`, `autoscaling.*`
