# Umami v3.0.3 Upgrade Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Upgrade the Umami Helm chart from v2.20.1 (PostgreSQL) to v3.0.3, removing MySQL support and updating all configurations for v3 compatibility.

**Architecture:** This is a breaking change upgrade that removes MySQL database support entirely. Umami v3 only supports PostgreSQL. The upgrade involves removing the MySQL subchart dependency, updating image tags, removing MySQL-related helpers and templates, updating documentation, and incrementing the chart version to reflect breaking changes (major version bump).

**Tech Stack:** Helm 3, Umami v3.0.3, PostgreSQL (Bitnami subchart v18.2.0), Kubernetes manifests

**Key Changes from Research:**
- **BREAKING**: MySQL no longer supported in Umami v3 - PostgreSQL only
- Image tag changes from `postgresql-v2.20.1` to `v3.0.3`
- Environment variables remain compatible (DATABASE_URL, APP_SECRET, etc.)
- Database migrations run automatically on first v3 startup
- New features: Updated UI, universal filters, segments/cohorts, links and pixels tracking

**Sources:**
- [Umami v3.0.0 Release Discussion](https://github.com/umami-software/umami/discussions/3685)
- [Upgrading Self-hosted Umami to v3](https://deepakness.com/blog/upgrading-umami-v3/)
- [Umami Environment Variables](https://umami.is/docs/environment-variables)
- [Umami v3 Release Info](https://github.com/umami-software/umami/releases)

---

## Task 1: Update Chart Metadata

**Files:**
- Modify: `Chart.yaml`
- Modify: `CHANGELOG.md`

**Step 1: Update Chart.yaml with new versions**

Update the chart version (major bump due to breaking changes) and app version:

```yaml
# Change line 34
appVersion: v3.0.3

# Change line 55 (increment major version from 7.3.0 to 8.0.0)
version: 8.0.0
```

**Step 2: Remove MySQL dependency from Chart.yaml**

Remove the MySQL dependency block (lines 40-43):

```yaml
# DELETE these lines:
- condition: mysql.enabled
  name: mysql
  repository: oci://registry-1.docker.io/bitnamicharts
  version: 14.0.3
```

Keep only the PostgreSQL dependency.

**Step 3: Update annotations for breaking change**

Update the `artifacthub.io/changes` annotation in Chart.yaml (lines 3-5):

```yaml
  artifacthub.io/changes: |
    - kind: changed
      description: "BREAKING: Upgraded to Umami v3.0.3 - MySQL support removed, PostgreSQL only"
    - kind: removed
      description: "Removed MySQL subchart dependency - Umami v3 requires PostgreSQL"
    - kind: changed
      description: "Updated default image tag to v3.0.3"
```

**Step 4: Update CHANGELOG.md**

Add new entry at the top of CHANGELOG.md:

```markdown
## 8.0.0 (2026-01-09)

### Breaking Changes

- **Upgraded to Umami v3.0.3**: MySQL support has been completely removed. Umami v3 only supports PostgreSQL.
- **Removed MySQL subchart dependency**: The `mysql.*` values are deprecated and non-functional.
- **Migration required**: Users on MySQL must migrate their database to PostgreSQL before upgrading. See [MySQL to PostgreSQL Migration Guide](https://docs.umami.is/docs/guides/migrate-mysql-postgresql).

### Changed

- Updated default image tag from `postgresql-v2.20.1` to `v3.0.3`
- Updated appVersion to v3.0.3

### New Features in Umami v3

- Redesigned user interface with improved navigation
- Universal filters applied via query strings for shareable URLs
- Segments and cohorts for saved filter sets and user tracking
- Links and pixels for external tracking capabilities
- Dedicated admin dashboard for managing users, websites, and teams
```

**Step 5: Run helm lint**

```bash
helm lint .
```

Expected: Should still pass with 0 failures

**Step 6: Commit chart metadata updates**

```bash
git add Chart.yaml CHANGELOG.md
git commit -m "chore: bump chart to v8.0.0 with Umami v3.0.3, remove MySQL dependency"
```

---

## Task 2: Update Default Values

**Files:**
- Modify: `values.yaml`

**Step 1: Update default image tag**

Change the default image tag (line 15):

```yaml
  # -- Overrides the image tag
  tag: "v3.0.3"
```

**Step 2: Deprecate MySQL configuration**

Replace the entire `mysql:` section (lines 233-242) with deprecation notice:

```yaml
mysql:
  # -- DEPRECATED: MySQL is no longer supported in Umami v3. This section is kept for backwards compatibility but will have no effect. Use PostgreSQL instead.
  # @ignored
  enabled: false
  auth:
    database: mychart
    password: mychart
    username: mychart
```

**Step 3: Update externalDatabase type description**

Update line 257 comment:

```yaml
  # -- Type of database (only postgresql supported in Umami v3)
  type: postgresql
```

**Step 4: Add migration warning comment**

Add a comment block at the top of the umami section (before line 166):

```yaml
# Umami v3 Configuration
# NOTE: Umami v3 only supports PostgreSQL. If upgrading from v2 with MySQL,
# you must migrate to PostgreSQL first: https://docs.umami.is/docs/guides/migrate-mysql-postgresql
# Database migrations from v2 to v3 run automatically on first startup.
umami:
```

**Step 5: Run helm template test**

```bash
helm template test-release . --set postgresql.enabled=true
```

Expected: Should render without errors

**Step 6: Commit values updates**

```bash
git add values.yaml
git commit -m "feat: update default image to v3.0.3, deprecate MySQL configuration"
```

---

## Task 3: Update Template Helpers

**Files:**
- Modify: `templates/_helpers.tpl`

**Step 1: Remove MySQL database type logic**

Update the `umami.database.type` helper (lines 135-143) to only return postgresql:

```go
{{/*
Get the type of the external database
*/}}
{{- define "umami.database.type" -}}
  {{- if .Values.postgresql.enabled -}}
    {{- "postgresql" -}}
  {{- else if .Values.externalDatabase.type -}}
    {{- printf "%s" (tpl .Values.externalDatabase.type $) -}}
  {{- end -}}
{{- end -}}
```

Remove the `else if .Values.mysql.enabled` block (lines 138-139).

**Step 2: Remove MySQL hostname logic**

Update the `umami.database.hostname` helper (lines 67-78) to remove MySQL:

```go
{{/*
Return the hostname of the database to use
*/}}
{{- define "umami.database.hostname" -}}
  {{- if .Values.postgresql.enabled -}}
    {{- printf "%s" (include "postgresql.v1.primary.fullname" .Subcharts.postgresql) -}}
  {{- else -}}
    {{- printf "%s" (tpl .Values.externalDatabase.hostname $) -}}
  {{- end -}}
{{- end -}}
```

Remove lines 73-74 (MySQL case).

**Step 3: Remove MySQL port logic**

Update the `umami.database.port` helper (lines 80-91) to remove MySQL:

```go
{{/*
Return database service port
*/}}
{{- define "umami.database.port" -}}
  {{- if .Values.postgresql.enabled -}}
    {{- printf "%s" (include "postgresql.v1.service.port" .Subcharts.postgresql) -}}
  {{- else -}}
    {{- printf "%s" (tpl (toString .Values.externalDatabase.port) $) -}}
  {{- end -}}
{{- end -}}
```

Remove lines 86-87 (MySQL case).

**Step 4: Remove MySQL database name logic**

Update the `umami.database.database` helper (lines 93-104) to remove MySQL:

```go
{{/*
Return the name for the database to use
*/}}
{{- define "umami.database.database" -}}
  {{- if .Values.postgresql.enabled -}}
    {{- printf "%s" (include "postgresql.v1.database" .Subcharts.postgresql) -}}
  {{- else -}}
    {{- printf "%s" (tpl .Values.externalDatabase.auth.database $) -}}
  {{- end -}}
{{- end -}}
```

Remove lines 99-100 (MySQL case).

**Step 5: Remove MySQL username logic**

Update the `umami.database.username` helper (lines 106-117) to remove MySQL:

```go
{{/*
Return the name for the user to use
*/}}
{{- define "umami.database.username" -}}
  {{- if .Values.postgresql.enabled -}}
    {{- printf "%s" (include "postgresql.v1.username" .Subcharts.postgresql) -}}
  {{- else -}}
    {{- printf "%s" (tpl .Values.externalDatabase.auth.username $) -}}
  {{- end -}}
{{- end -}}
```

Remove lines 112-113 (MySQL case).

**Step 6: Remove MySQL password logic**

Update the `umami.database.password` helper (lines 119-130) to remove MySQL:

```go
{{/*
Get the password for the database
*/}}
{{- define "umami.database.password" -}}
  {{- if .Values.postgresql.enabled -}}
    {{- printf "%s" (tpl .Values.postgresql.auth.password $) -}}
  {{- else if .Values.externalDatabase.auth.password -}}
    {{- printf "%s" (tpl .Values.externalDatabase.auth.password $) -}}
  {{- end -}}
{{- end -}}
```

Remove lines 125-126 (MySQL case).

**Step 7: Verify helpers with template test**

```bash
helm template test-release . --debug 2>&1 | grep -A 5 "database-url"
```

Expected: Should show PostgreSQL connection string only

**Step 8: Commit helper updates**

```bash
git add templates/_helpers.tpl
git commit -m "refactor: remove MySQL support from template helpers"
```

---

## Task 4: Add Migration Warning to NOTES

**Files:**
- Modify: `templates/NOTES.txt`

**Step 1: Add v3 migration notice**

Add a warning section after the main notes (around line 20, before existing notes end):

```text

{{- if not .Values.postgresql.enabled }}
================================================================================
WARNING: Umami v3 requires PostgreSQL
================================================================================

Umami v3 only supports PostgreSQL databases. If you are using an external
database, ensure it is PostgreSQL.

If you are upgrading from Umami v2 with MySQL, you must migrate your data
to PostgreSQL before upgrading. See the migration guide:
https://docs.umami.is/docs/guides/migrate-mysql-postgresql

{{- end }}

{{- if .Values.mysql.enabled }}
================================================================================
ERROR: MySQL is no longer supported
================================================================================

Umami v3 removed MySQL support. The mysql.enabled setting will have no effect.

You must:
1. Migrate your MySQL data to PostgreSQL
2. Set postgresql.enabled=true
3. Set mysql.enabled=false

Migration guide: https://docs.umami.is/docs/guides/migrate-mysql-postgresql

{{- end }}
```

**Step 2: Test NOTES rendering with MySQL enabled**

```bash
helm template test-release . --set mysql.enabled=true --set postgresql.enabled=false --show-only templates/NOTES.txt
```

Expected: Should show ERROR message about MySQL not supported

**Step 3: Test NOTES rendering with external database**

```bash
helm template test-release . --set postgresql.enabled=false --set externalDatabase.hostname=db.example.com --show-only templates/NOTES.txt
```

Expected: Should show WARNING about ensuring PostgreSQL

**Step 4: Commit NOTES updates**

```bash
git add templates/NOTES.txt
git commit -m "feat: add v3 migration warnings to installation notes"
```

---

## Task 5: Update README Documentation

**Files:**
- Modify: `README.md.gotmpl`

**Step 1: Add breaking change notice at the top**

Add a prominent notice after the Introduction section (after line 16):

```markdown
## ⚠️ Important: Umami v3 Breaking Changes

**Umami v3 (chart version 8.0.0+) only supports PostgreSQL.** MySQL support has been completely removed.

If you are upgrading from chart version 7.x (Umami v2) with MySQL:

1. **You must migrate your data to PostgreSQL before upgrading**
2. Follow the [MySQL to PostgreSQL Migration Guide](https://docs.umami.is/docs/guides/migrate-mysql-postgresql)
3. Update your values to use PostgreSQL instead of MySQL

Database migrations from v2 to v3 schema run automatically on the first startup.

**What's New in Umami v3:**
- Redesigned user interface
- Universal filters with shareable URLs
- Segments and cohorts for user tracking
- Links and pixels for external tracking
- Dedicated admin dashboard

For more details, see the [Umami v3 announcement](https://umami.is/blog/umami-v3).
```

**Step 2: Update the Upgrading section**

Add new upgrade section at the beginning (after the values table but before "### To 7.0.0"):

```markdown
### To 8.0.0

**BREAKING CHANGE**: This major version upgrades Umami from v2.20.1 to v3.0.3 and removes MySQL support entirely.

**Prerequisites:**
- If using MySQL, you **must** migrate to PostgreSQL before upgrading
- Follow the [MySQL to PostgreSQL Migration Guide](https://docs.umami.is/docs/guides/migrate-mysql-postgresql)
- Ensure you are on the latest v2 version (chart 7.3.0) before migrating

**Changes:**
- Umami upgraded from v2.20.1 to v3.0.3
- MySQL subchart removed (no longer supported)
- `mysql.*` values are deprecated and have no effect
- PostgreSQL is now the only supported database backend
- Database migrations from v2 to v3 run automatically on first startup

**Migration steps:**

1. **Backup your database** (critical!)
   ```bash
   # PostgreSQL users
   pg_dump -h <host> -U <user> <database> > umami_v2_backup.sql
   ```

2. **If on MySQL, migrate to PostgreSQL first** (before upgrading the chart)
   - Follow: https://docs.umami.is/docs/guides/migrate-mysql-postgresql
   - Update your values.yaml to use PostgreSQL

3. **Upgrade the chart**
   ```bash
   helm repo update
   helm upgrade my-release christianhuth/umami --version 8.0.0
   ```

4. **Verify the upgrade**
   - Check pod logs for successful v2→v3 migration
   - Test login and data access
   - Verify all analytics data is present

**Rollback:**
If you need to rollback, restore your v2 database backup and downgrade:
```bash
helm rollback my-release
```
```

**Step 3: Update values table description for mysql**

The values table is auto-generated, but add a note above it (before line 47):

```markdown
> **Note:** As of chart version 8.0.0 (Umami v3), MySQL is no longer supported. The `mysql.*` values are kept for backwards compatibility but have no effect.
```

**Step 4: Regenerate README (if using helm-docs)**

```bash
# If helm-docs is available, regenerate
# helm-docs --dry-run
echo "Note: README.md is generated from README.md.gotmpl. After this plan is complete, regenerate with helm-docs or manually update README.md to match README.md.gotmpl changes."
```

**Step 5: Commit README template updates**

```bash
git add README.md.gotmpl
git commit -m "docs: add v3 upgrade guide and breaking change notices"
```

---

## Task 6: Update Values Schema

**Files:**
- Modify: `values.schema.json`

**Step 1: Add deprecation to mysql properties**

Find the `mysql` property in values.schema.json and add a deprecation notice. Search for the mysql section and update it:

```bash
# First, let's check the current schema structure
grep -n '"mysql"' values.schema.json
```

Expected: Find line number of mysql property definition

**Step 2: Mark mysql.enabled as deprecated**

Locate the mysql.enabled property in the schema and add deprecated flag. Add after the "description" field:

```json
"deprecated": true,
"description": "DEPRECATED: MySQL is no longer supported in Umami v3. This setting has no effect. Use postgresql.enabled instead."
```

**Step 3: Update externalDatabase.type enum**

Find the externalDatabase.type property and ensure it only allows postgresql:

```json
"type": {
  "default": "postgresql",
  "description": "Type of database (only postgresql supported in Umami v3)",
  "title": "type",
  "type": "string",
  "enum": ["postgresql"]
}
```

**Step 4: Validate schema**

```bash
# Validate JSON syntax
python3 -c "import json; json.load(open('values.schema.json'))" && echo "Schema is valid JSON"
```

Expected: "Schema is valid JSON"

**Step 5: Test schema with helm**

```bash
helm template test-release . --values values.yaml > /dev/null && echo "Schema validation passed"
```

Expected: "Schema validation passed"

**Step 6: Commit schema updates**

```bash
git add values.schema.json
git commit -m "chore: mark MySQL as deprecated in values schema, restrict database type to postgresql"
```

---

## Task 7: Update CLAUDE.md Documentation

**Files:**
- Modify: `CLAUDE.md`

**Step 1: Update overview section**

Update the overview section to reflect v3 (around line 7):

```markdown
## Overview

This is a Helm chart for deploying Umami (a privacy-focused web analytics platform) on Kubernetes. The chart packages Umami v3.x with PostgreSQL database dependency and provides extensive configuration options for Kubernetes deployment.

**Note:** Chart version 8.0.0+ uses Umami v3, which only supports PostgreSQL. MySQL support was removed in Umami v3.
```

**Step 2: Update database architecture section**

Replace the "Database Architecture" section (around line 32) with updated information:

```markdown
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
```

**Step 3: Add upgrade notes section**

Add a new section after "Important Notes":

```markdown
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
```

**Step 4: Update image tag in values structure**

Update the example in "Chart Values Structure" section (around line 178):

```markdown
- `image.*` - Container image configuration (registry: ghcr.io, repository: umami-software/umami, tag: v3.0.3)
```

**Step 5: Remove MySQL references**

Remove or update any remaining MySQL references in the document. Search and update:

```markdown
# Remove "bundled MySQL" from lists
# Change "PostgreSQL or MySQL" to "PostgreSQL"
# Update any examples using MySQL
```

**Step 6: Commit CLAUDE.md updates**

```bash
git add CLAUDE.md
git commit -m "docs: update CLAUDE.md for v3 with PostgreSQL-only support"
```

---

## Task 8: Final Testing and Validation

**Files:**
- None (testing only)

**Step 1: Run comprehensive helm lint**

```bash
helm lint . --strict
```

Expected: `1 chart(s) linted, 0 chart(s) failed`

**Step 2: Test template rendering with PostgreSQL**

```bash
helm template test-release . \
  --set postgresql.enabled=true \
  --set postgresql.auth.password=testpass \
  --set umami.appSecret.secret=testsecret
```

Expected: All templates render successfully, no MySQL references in output

**Step 3: Test template with external PostgreSQL**

```bash
helm template test-release . \
  --set postgresql.enabled=false \
  --set externalDatabase.hostname=postgres.example.com \
  --set externalDatabase.auth.password=testpass \
  --set externalDatabase.type=postgresql \
  --set umami.appSecret.secret=testsecret
```

Expected: Database URL uses external hostname

**Step 4: Verify MySQL deprecation warnings**

```bash
helm template test-release . \
  --set mysql.enabled=true \
  --set postgresql.enabled=false \
  --show-only templates/NOTES.txt | grep -i "ERROR"
```

Expected: Shows ERROR message about MySQL not supported

**Step 5: Check image tag in deployment**

```bash
helm template test-release . --show-only templates/deployment.yaml | grep "image:"
```

Expected: Should show `ghcr.io/umami-software/umami:v3.0.3`

**Step 6: Verify no MySQL helpers remain**

```bash
grep -n "mysql" templates/_helpers.tpl
```

Expected: No matches (or only in comments)

**Step 7: Run final helm dependency update**

```bash
helm dependency update
```

Expected: Only PostgreSQL dependency downloaded, no MySQL

**Step 8: Commit any final cleanup**

```bash
# If Chart.lock needs updating
git add Chart.lock
git commit -m "chore: update Chart.lock after removing MySQL dependency"
```

---

## Task 9: Create Release Tag and Summary

**Files:**
- None (git operations)

**Step 1: Verify all changes are committed**

```bash
git status
```

Expected: `nothing to commit, working tree clean`

**Step 2: Review commit history**

```bash
git log --oneline -10
```

Expected: Should see all commits from this plan

**Step 3: Create annotated tag for release**

```bash
git tag -a v8.0.0 -m "Release v8.0.0: Upgrade to Umami v3.0.3

BREAKING CHANGES:
- Upgraded Umami from v2.20.1 to v3.0.3
- Removed MySQL support (PostgreSQL only)
- Removed MySQL subchart dependency

New Features in Umami v3:
- Redesigned UI with improved navigation
- Universal filters with shareable URLs
- Segments and cohorts for user tracking
- Links and pixels for external tracking
- Dedicated admin dashboard

Migration Required:
- MySQL users must migrate to PostgreSQL before upgrading
- See: https://docs.umami.is/docs/guides/migrate-mysql-postgresql
- Database schema migrations run automatically on first v3 startup

Chart Changes:
- Updated default image tag to v3.0.3
- Deprecated mysql.* values
- Removed MySQL template logic
- Added migration warnings to NOTES.txt
- Updated documentation with upgrade guide
"
```

**Step 4: Generate release summary**

Create a summary document:

```bash
cat > docs/RELEASE_8.0.0.md << 'EOF'
# Release 8.0.0 - Umami v3.0.3

## Summary

This release upgrades the Umami Helm chart to Umami v3.0.3, which is a breaking change release that removes MySQL support and only supports PostgreSQL.

## Breaking Changes

- **MySQL support removed**: Umami v3 no longer supports MySQL databases
- **PostgreSQL only**: Only PostgreSQL is supported as the database backend
- **MySQL subchart removed**: The Bitnami MySQL dependency has been removed from the chart
- **Migration required**: Users on MySQL must migrate to PostgreSQL before upgrading

## Upgrade Path

### For PostgreSQL Users (Current State: chart v7.x with postgresql.enabled=true)

1. Backup your database
2. Upgrade the chart to v8.0.0
3. Database migrations run automatically
4. Verify data after upgrade

### For MySQL Users (Current State: chart v7.x with mysql.enabled=true)

1. Backup your MySQL database
2. Migrate data to PostgreSQL using: https://docs.umami.is/docs/guides/migrate-mysql-postgresql
3. Update values.yaml: set postgresql.enabled=true, mysql.enabled=false
4. Upgrade to chart v8.0.0
5. Verify data after upgrade

## New Features in Umami v3

- **Redesigned Interface**: Modern UI with improved navigation and website switching
- **Universal Filters**: Filters in query strings for shareable analytics URLs
- **Segments & Cohorts**: Save filter sets and track user groups over time
- **Links & Pixels**: New tracking elements for external link clicks and embedded tracking
- **Admin Dashboard**: Centralized management for users, websites, and teams

## Testing

Tested with:
- helm lint: ✓ Pass
- helm template with postgresql: ✓ Pass
- helm template with externalDatabase: ✓ Pass
- MySQL deprecation warnings: ✓ Working

## Documentation

- README.md updated with migration guide
- CHANGELOG.md includes full breaking changes list
- CLAUDE.md updated for v3 architecture
- NOTES.txt includes migration warnings

## References

- [Umami v3 Release](https://github.com/umami-software/umami/discussions/3685)
- [Migration Guide](https://docs.umami.is/docs/guides/migrate-mysql-postgresql)
- [Umami v3 Blog Post](https://umami.is/blog/umami-v3)
EOF
```

**Step 5: Review final checklist**

```bash
cat << 'EOF'
## Release Checklist

- [x] Chart.yaml version bumped to 8.0.0
- [x] Chart.yaml appVersion updated to v3.0.3
- [x] MySQL dependency removed from Chart.yaml
- [x] values.yaml default tag updated to v3.0.3
- [x] values.yaml MySQL section deprecated
- [x] _helpers.tpl MySQL logic removed
- [x] NOTES.txt migration warnings added
- [x] README.md.gotmpl upgrade guide added
- [x] values.schema.json MySQL marked deprecated
- [x] CLAUDE.md updated for v3
- [x] CHANGELOG.md includes v8.0.0 entry
- [x] All tests passing (lint, template)
- [x] Git tag created
- [x] Release notes documented

Ready for release!
EOF
```

**Step 6: Final commit**

```bash
git add docs/RELEASE_8.0.0.md
git commit -m "docs: add release notes for v8.0.0"
```

---

## Post-Implementation Verification

After completing all tasks, verify the upgrade with these manual tests:

### Test 1: Fresh Install with PostgreSQL

```bash
helm install test-v3 . \
  --set postgresql.auth.password=testpassword \
  --set umami.appSecret.secret=testsecret123 \
  --dry-run --debug
```

Expected: Clean deployment, no errors, PostgreSQL configured

### Test 2: Verify MySQL Error Message

```bash
helm install test-mysql . \
  --set mysql.enabled=true \
  --set postgresql.enabled=false \
  --dry-run | grep -A 5 "ERROR"
```

Expected: Clear error message about MySQL not supported

### Test 3: External PostgreSQL

```bash
helm install test-external . \
  --set postgresql.enabled=false \
  --set externalDatabase.hostname=mydb.postgres.database.azure.com \
  --set externalDatabase.auth.username=umami \
  --set externalDatabase.auth.password=securepass \
  --set externalDatabase.auth.database=umami_db \
  --set umami.appSecret.secret=testsecret123 \
  --dry-run --debug
```

Expected: Deployment uses external database URL correctly

### Test 4: Image Tag Verification

```bash
helm template test . | grep "image:" | head -1
```

Expected: `image: "ghcr.io/umami-software/umami:v3.0.3"`

---

## Rollback Plan

If issues are discovered post-deployment:

1. **Restore database backup**:
   ```bash
   psql -h <host> -U <user> <database> < umami_v2_backup.sql
   ```

2. **Rollback Helm release**:
   ```bash
   helm rollback <release-name> <previous-revision>
   ```

3. **Verify rollback**:
   ```bash
   kubectl get pods
   helm list
   ```

---

## Known Limitations

1. **MySQL migration**: The MySQL to PostgreSQL migration must be done manually before chart upgrade
2. **No automatic migration**: The chart does not include automated MySQL→PostgreSQL data migration
3. **Breaking change**: This is a major version bump and cannot be rolled back without database restore
4. **API changes**: Umami v3 has API breaking changes that may affect external integrations

---

## Success Criteria

- [ ] Chart version 8.0.0 with appVersion v3.0.3
- [ ] helm lint passes with 0 errors
- [ ] helm template renders successfully with PostgreSQL
- [ ] MySQL deprecation warnings display correctly
- [ ] No MySQL references in templates or helpers
- [ ] Documentation complete and accurate
- [ ] All commits follow conventional commit format
- [ ] Git tag v8.0.0 created with detailed message
- [ ] Release notes document created
