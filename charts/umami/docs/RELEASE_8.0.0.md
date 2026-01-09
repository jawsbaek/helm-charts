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

## Chart Changes

### Files Modified

- `Chart.yaml` - Version bumped to 8.0.0, appVersion v3.0.3, MySQL dependency removed
- `CHANGELOG.md` - Added v8.0.0 entry with breaking changes
- `Chart.lock` - Regenerated without MySQL dependency
- `values.yaml` - Deprecated MySQL configuration, updated image tag to v3.0.3
- `templates/_helpers.tpl` - Removed MySQL logic, added validation for PostgreSQL-only
- `templates/NOTES.txt` - Added migration warnings for MySQL users
- `README.md.gotmpl` - Added v3 upgrade guide and breaking change notices
- `values.schema.json` - Marked MySQL as deprecated, restricted database type to PostgreSQL
- `CLAUDE.md` - Updated architectural documentation for v3
- `docs/plans/2026-01-09-upgrade-umami-v3.md` - Implementation plan (reference)

### Validation Added

- Schema validation blocks `externalDatabase.type=mysql`
- Helper template validation fails with clear error for MySQL
- NOTES.txt displays ERROR message when `mysql.enabled=true`
- NOTES.txt displays WARNING when using external database

## Testing

Tested with:
- ✅ helm lint --strict: PASS (0 failures)
- ✅ helm template with postgresql: PASS (renders correctly)
- ✅ helm template with externalDatabase: PASS (uses external hostname)
- ✅ MySQL deprecation warnings: PASS (schema blocks, NOTES displays)
- ✅ Image tag verification: PASS (ghcr.io/umami-software/umami:v3.0.3)
- ✅ MySQL helper cleanup: PASS (only validation references)
- ✅ Dependency update: PASS (PostgreSQL 18.2.0 only)

## Documentation

- README.md.gotmpl updated with migration guide
- CHANGELOG.md includes full breaking changes list
- CLAUDE.md updated for v3 architecture
- NOTES.txt includes migration warnings
- values.schema.json documents deprecation

## References

- [Umami v3 Release](https://github.com/umami-software/umami/discussions/3685)
- [Migration Guide](https://docs.umami.is/docs/guides/migrate-mysql-postgresql)
- [Umami v3 Blog Post](https://umami.is/blog/umami-v3)
- [Implementation Plan](docs/plans/2026-01-09-upgrade-umami-v3.md)

## Rollback

If upgrade fails:
```bash
# Restore database backup
psql -h <host> -U <user> <database> < umami_v2_backup.sql

# Rollback helm release
helm rollback <release-name>
```

## Contributors

- Implemented by: Claude Code (Anthropic)
- Chart maintained by: christianhuth
- Based on: Umami v3.0.3 by umami-software

---

**Ready for release: ✅ YES**

All validation tests passed, documentation complete, breaking changes clearly communicated.
