# shared-workflows

Shared GitHub Actions workflows for [mcarthey](https://github.com/mcarthey) projects.

## Reusable Workflows

### `dotnet-ci-reusable.yml`

A reusable CI workflow for .NET MAUI + API projects. Provides three jobs:

1. **Build & Test** — Restore, build, test with coverage, optional smoke test
2. **Build MAUI** — Compile check for MAUI projects (conditional, macOS runner)
3. **PR Summary** — Publish test results as a PR comment (conditional, PRs only)

#### Usage

In your repository, create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: ["main", "feature/**", "fix/**"]
  pull_request:
    branches: ["main"]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: mcarthey/shared-workflows/.github/workflows/dotnet-ci-reusable.yml@main
    permissions:
      contents: read
      pull-requests: write
      checks: write
    with:
      solution-filter: "MyProject.CI.slnf"
      maui-project: "MyProject.App/MyProject.App.csproj"
      maui-ui-project: "MyProject.UI/MyProject.UI.csproj"
      smoke-test-url: "http://localhost:5099/health"
      smoke-test-cwd: "MyProject.Api"
      smoke-test-env: "Jwt__Secret=CI-Key-32-Chars-Minimum-For-HMAC!"
    secrets: inherit
```

#### Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `solution-filter` | **Yes** | — | Solution filter (`.slnf`) for build & test |
| `dotnet-version` | No | `9.0.x` | .NET SDK version |
| `test-settings` | No | `./coverage.runsettings` | Path to coverage runsettings file |
| `maui-project` | No | `""` | MAUI App `.csproj` path (skip MAUI job if empty) |
| `maui-ui-project` | No | `""` | MAUI UI `.csproj` path (skip if empty) |
| `maui-runner` | No | `macos-15` | Runner for MAUI build |
| `maui-target-framework` | No | `net9.0-android` | Target framework for MAUI compile check |
| `smoke-test-url` | No | `""` | Health check URL (skip smoke test if empty) |
| `smoke-test-cwd` | No | `""` | Working directory for smoke test |
| `smoke-test-env` | No | `""` | Space-separated `KEY=VALUE` env vars for smoke test |
| `codecov-enabled` | No | `false` | Upload coverage to Codecov |
| `test-timeout-minutes` | No | `15` | Timeout for build & test job |
| `maui-timeout-minutes` | No | `30` | Timeout for MAUI build job |

#### Caller Permissions

The caller must grant permissions that the reusable workflow's jobs need. Repos with `default_workflow_permissions=read` (recommended) will get `startup_failure` without these:

```yaml
permissions:
  contents: read        # for checkout
  pull-requests: write  # for PR test summary
  checks: write         # for PR test summary
```

#### Prerequisites

Each consuming repository needs:

1. **Solution filter** (`.slnf`) — excludes MAUI projects for the Linux build job
2. **Coverage runsettings** (`coverage.runsettings`) — excludes generated code from coverage

## Design Principles

Based on [Anatomy of a CI Workflow](https://learnedgeek.com/Blog/Post/anatomy-of-a-ci-workflow):

- **Solution filters** over fragile project enumeration
- **Per-job permissions** (least privilege)
- **NuGet caching** with composite key
- **Concurrency** with cancel-in-progress (configured in caller)
- **Conditional MAUI builds** (main + PRs only, to control macOS runner costs)
- **Artifact retention** at 14 days
