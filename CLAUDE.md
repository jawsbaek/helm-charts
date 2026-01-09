# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Kubernetes Helm Charts repository containing 42 production-ready charts for deploying various applications on Kubernetes. Charts are published to `https://jawsbaek.github.io/helm-charts` via GitHub Pages.

**Current state:** Working on Umami v3 upgrade (feature/umami-v3-upgrade branch) - PostgreSQL-only breaking change.

## Common Commands

### Chart Development Workflow

```bash
# Make changes to a chart
cd charts/<chart-name>

# Validate changes
helm lint .
helm template my-release . --debug

# Update dependencies if Chart.yaml changed
helm dependency update

# Run pre-commit hooks (generates docs, schemas, validates)
pre-commit run --all-files

# Test chart installation in Kind cluster
ct install --charts charts/<chart-name> --config .github/config/chart-testing.yaml
```

### Testing and Validation

```bash
# Run all linting checks
ct lint --config .github/config/chart-testing.yaml

# Validate against Kubernetes schemas
helm template charts/<chart-name> . | kubeconform -strict

# Run unit tests (if tests exist)
helm unittest charts/<chart-name>
```

### Documentation

**CRITICAL:** Never edit `README.md` directly in charts - edit `README.md.gotmpl` instead.

```bash
# Regenerate chart documentation (runs automatically in pre-commit)
docker run --rm -v $(pwd):/helm-docs jnorwood/helm-docs:v1.14.2

# Regenerate values schema
docker run --rm -v $(pwd):/schema dadav/helm-schema:latest
```

### Git Workflow

```bash
# Create feature branch
git checkout -b feature/my-change

# Commit changes (triggers pre-commit hooks automatically)
git add charts/<chart-name>
git commit -m "feat(chart-name): description"

# Push and create PR
git push -u origin feature/my-change
gh pr create --title "feat(chart-name): description" --base main
```

### GitHub CLI Operations

```bash
# View PR status and checks
gh pr view <number> --json statusCheckRollup

# Check code scanning alerts
gh api repos/jawsbaek/helm-charts/code-scanning/alerts

# View workflow runs
gh run list --limit 10

# Merge PR after checks pass
gh pr merge <number> --squash --auto
```

## Architecture

### Chart Structure

Each chart follows this standard structure:

```
charts/<chart-name>/
├── Chart.yaml              # Metadata, version, dependencies
├── Chart.lock              # Locked dependency versions
├── values.yaml             # Default configuration
├── values.schema.json      # JSON schema validation (auto-generated)
├── README.md               # Documentation (auto-generated from .gotmpl)
├── README.md.gotmpl        # Documentation template (EDIT THIS, not README.md)
├── CHANGELOG.md            # Version history
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl        # Template helper functions
│   └── NOTES.txt           # Post-install instructions
└── tests/                  # Helm unittest tests (optional)
```

### CI/CD Pipeline

The main pipeline (`chart-pipeline.yml`) orchestrates 6 sequential stages:

```
1. Preparation (chart-preparation.yml)
   └─ Runs pre-commit hooks, auto-commits generated files

2. Linting (chart-linting.yml) - 3 parallel checks
   ├─ ArtifactHub validation
   ├─ Chart-testing lint
   └─ Helm lint

3. Kubeconform (chart-kubeconform.yml)
   └─ Validates rendered templates against Kubernetes API schemas

4. Unit Testing (chart-unittesting.yml)
   └─ Runs helm-unittest tests (if present)

5. Integration Testing (chart-testing.yml)
   ├─ Creates Kind cluster
   └─ Actually installs charts and validates

6. Chart Releasing (chart-releasing.yml) - main branch only
   └─ Packages and publishes to GitHub Pages
```

**Trigger:** Pipeline runs on any push to `charts/**` paths.

### Template Helper Pattern

Charts use `_helpers.tpl` for DRY helper functions:

```yaml
{{- define "chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
```

Common helper patterns:
- `*.fullname` - Generates resource names
- `*.labels` - Standard Kubernetes labels
- `*.selectorLabels` - Pod selector labels
- `*.database.*` - Database connection abstraction (see Umami chart)

### Configuration Layers

```
values.yaml (defaults)
    ↓
User values (--set flags or -f override.yaml)
    ↓
values.schema.json (validation)
    ↓
Template rendering (helm template / helm install)
    ↓
Kubernetes manifests
```

### Dependency Management

Chart dependencies are declared in `Chart.yaml`:

```yaml
dependencies:
  - name: postgresql
    version: "18.2.0"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

Dependencies are:
- Managed via `helm dependency update` (creates Chart.lock)
- Conditionally enabled via `*.enabled` values
- Configured via scoped values (e.g., `postgresql.auth.password`)

## Security

### GitHub Actions Hardening

All workflows follow zizmor security recommendations:

**Permissions:** Explicitly set minimal permissions (contents: read by default)
```yaml
permissions:
  contents: read  # Most jobs
  # OR
  contents: write  # Only for jobs that push commits/releases
```

**Template Injection Prevention:** Use environment variables instead of direct expansion
```yaml
# BAD - vulnerable to code injection
run: echo "${{ steps.output.value }}"

# GOOD - safe via environment variable
env:
  VALUE: ${{ steps.output.value }}
run: echo "$VALUE"
```

**Credential Protection:** Disable credential persistence
```yaml
- uses: actions/checkout@v4
  with:
    persist-credentials: false
```

**pull_request_target Safety:** labeler.yml implements:
- Branch filter (only main branch)
- Repository check (github.repository == 'jawsbaek/helm-charts')
- Base branch checkout only (no PR code execution)

### Ignored Security Warnings

`.github/config/zizmor.yaml` ignores specific warnings with justification:

```yaml
artipacked:
  ignore:
    - claude-code-review.yml  # Requires credentials for GitHub API
    - claude.yml              # Requires credentials for GitHub API

dangerous-triggers:
  ignore:
    - labeler.yml            # Implements all safety measures
```

## Important Files

### Configuration Files

| File | Purpose | When to Edit |
|------|---------|--------------|
| `.pre-commit-config.yaml` | Pre-commit hook definitions | Adding new validation steps |
| `.github/config/chart-testing.yaml` | Chart-testing tool config | Changing test behavior |
| `.github/config/zizmor.yaml` | Security scanner config | Adding workflow exceptions |
| `.github/config/labeler.yaml` | Auto-labeler rules | Adding new chart labels |
| `renovate.json` | Dependency update config | Changing update strategy |

### Generated Files (DO NOT EDIT DIRECTLY)

- `charts/*/README.md` - Edit `README.md.gotmpl` instead
- `charts/*/values.schema.json` - Auto-generated from values.yaml
- `Chart.lock` - Generated by `helm dependency update`

### Template Files (EDIT THESE)

- `charts/*/README.md.gotmpl` - Source for README.md
- `charts/*/templates/*.yaml` - Kubernetes manifest templates
- `charts/*/templates/_helpers.tpl` - Template helper functions

## Chart-Specific Guidance

### For Charts with Database Dependencies

When working on charts that use PostgreSQL/MySQL subcharts:

1. **Connection URL Construction:** Use helper functions in `_helpers.tpl` to abstract database connection details (see Umami chart for reference)

2. **Secret Management:** Store database URLs in Secrets, not ConfigMaps
   ```yaml
   {{- define "chart.database.url" -}}
   postgresql://{{ .Values.postgresql.auth.username }}:{{ .Values.postgresql.auth.password }}@...
   {{- end }}
   ```

3. **External Database Support:** Always provide option for external database
   ```yaml
   postgresql:
     enabled: true  # Bundled subchart
   externalDatabase:
     enabled: false  # User-managed database
   ```

### For Charts with Breaking Changes

When introducing breaking changes (e.g., Umami v3 PostgreSQL-only):

1. **Bump major version** in Chart.yaml (7.x → 8.x)
2. **Update CHANGELOG.md** with migration notes
3. **Document in README.md.gotmpl** under "Upgrading" section
4. **Consider migration Job** in templates/job.yaml for automated migration
5. **Update values.schema.json** to remove deprecated fields

## Claude Code Integration

### Interactive Agent (claude.yml)

Trigger by mentioning `@claude` in:
- Issue comments
- PR comments
- PR reviews
- Issue descriptions

**Capabilities:**
- Runs with repository context
- Can read CI results (`actions: read` permission)
- Custom prompts via workflow configuration

### Automated Code Review (claude-code-review.yml)

Runs automatically on every PR (opened, synchronize, ready_for_review, reopened)

**Uses:** `code-review` plugin from marketplace

### Permissions

Configured in `.claude/settings.local.json`:
- Git operations (add, commit, push, checkout, switch)
- GitHub CLI (pr create, pr merge, api calls)
- Web search and fetching
- Superpowers: brainstorming skill

## Development Workflow

### Standard Feature Development

1. **Create branch from main**
   ```bash
   git checkout main
   git pull
   git checkout -b feature/chart-name-feature
   ```

2. **Make changes**
   - Edit chart templates/values
   - Edit `README.md.gotmpl` (not README.md)
   - Bump version in `Chart.yaml`
   - Update `CHANGELOG.md`

3. **Test locally**
   ```bash
   helm lint charts/chart-name
   helm template test charts/chart-name --debug
   ```

4. **Commit (triggers pre-commit hooks)**
   ```bash
   git add charts/chart-name
   git commit -m "feat(chart-name): add feature X"
   ```

5. **Push and create PR**
   ```bash
   git push -u origin feature/chart-name-feature
   gh pr create --title "feat(chart-name): add feature X" --base main
   ```

6. **Wait for CI checks** - Pipeline runs all 6 stages automatically

7. **Merge after approval** - Auto-merge or manual merge

### Dependency Updates (via Renovate)

Renovate bot automatically:
- Updates subchart versions weekly
- Updates container image tags
- Auto-merges minor/patch versions
- Runs post-upgrade tasks:
  - `renovate-update-chart-dependency.sh` - Bumps chart version
  - `renovate-update-chart-values.sh` - Updates image tags
  - `.github/generate-chart-changelogs.sh` - Updates CHANGELOG.md

**Manual dependency update:**
```bash
cd charts/<chart-name>
# Edit Chart.yaml dependencies
helm dependency update
git add Chart.yaml Chart.lock charts/
git commit -m "chore(chart-name): update dependencies"
```

## Troubleshooting

### Pre-commit hooks failing

```bash
# Skip hooks temporarily (NOT recommended)
git commit --no-verify

# Fix specific hook
pre-commit run <hook-id> --all-files

# Update hook versions
pre-commit autoupdate
```

### Chart release not publishing

Check:
1. Is this the main branch? (Releases only happen on main)
2. Did Chart.yaml version bump? (Required for release)
3. Check workflow logs: `gh run view <run-id> --log`

### Kind cluster tests failing locally

```bash
# Ensure Kind is installed
kind --version

# Check cluster exists
kind get clusters

# View CT config
cat .github/config/chart-testing.yaml
```

### Database connection issues in charts

Check helper function output:
```bash
helm template test charts/chart-name --debug 2>&1 | grep -A5 "DATABASE_URL"
```

Verify secret generation:
```bash
helm template test charts/chart-name -s templates/db-secret.yaml
```

## Key Insights from Existing Charts

### Umami Chart (charts/umami/)

**Special features:**
- Has dedicated `CLAUDE.md` with chart-specific guidance
- PostgreSQL-only as of v3 (MySQL removed)
- Database URL construction via helper functions
- Migration Job for v1→v2 upgrades
- Gateway API HTTPRoute support (BETA)

**Reference for:**
- Complex database helper patterns
- Breaking change management (v2→v3)
- External vs bundled database logic

### ProxySQL Chart (charts/proxysql/)

**Reference for:**
- MySQL high availability patterns
- StatefulSet deployments
- Custom ConfigMap templating

### Priority Classes Chart (charts/priority-classes/)

**Reference for:**
- Charts without deployments (just resources)
- Namespace-scoped vs cluster-scoped resources

## Repository Conventions

### Commit Message Format

Follow Conventional Commits:
- `feat(chart-name): description` - New features
- `fix(chart-name): description` - Bug fixes
- `chore(chart-name): description` - Maintenance
- `docs(chart-name): description` - Documentation only
- Use `!` for breaking changes: `feat(chart-name)!: description`

### Chart Versioning

Follow Semantic Versioning:
- **MAJOR:** Breaking changes (e.g., removing MySQL support)
- **MINOR:** New features, backwards compatible
- **PATCH:** Bug fixes, no functionality changes

### Branch Naming

- `feature/chart-name-description` - New features
- `fix/chart-name-description` - Bug fixes
- `chore/description` - Repository maintenance

### PR Guidelines

See `.github/PULL_REQUEST_TEMPLATE.md` for checklist including:
- Chart version bumped
- CHANGELOG.md updated
- Documentation regenerated
- Tests pass locally
