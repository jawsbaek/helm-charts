# umami

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

## 7.3.0

### Added

- support for Gateway API routes
