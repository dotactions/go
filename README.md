# dotactions/go

Composite actions and a reusable CI workflow for Go modules: setup, lint, test with coverage.

## Versioning

Pin the **full commit SHA** of a release tag, with the version in a comment, or a floating minor tag:

```yaml
uses: dotactions/go/setup@<40-char-sha> # v0.1.0
uses: dotactions/go/.github/workflows/ci.yml@v0.1
```

Resolve a SHA with `git ls-remote https://github.com/dotactions/go 'refs/tags/v0.1.0^{}'`. Only SHAs of **release tags** pin the whole tree; a SHA from `main` still contains `@main` self-references.

| Ref | Moves on | Use |
| :-- | :-- | :-- |
| `@<sha>` | never | **Recommended**: immutable, full reproducibility |
| `@v0.1` | patch releases of `0.1` | **Recommended**: bug fixes, no breaking changes |
| `@v0.1.0` | never (tag can be moved) | Readable, but tags are mutable |
| `@v0` | every `0.x` release | Not recommended: during `0.x`, minor releases may break you |

Release notes: [GitHub Releases](https://github.com/dotactions/go/releases).

## Permissions

Examples assume a **public** repository. In a **private** repository, `actions/checkout` needs `contents: read`. Reusable workflows can only narrow the caller's token permissions, never widen them: grant the permissions listed under `ci.yml` in the calling job.

## Setup Go (`setup`)

Installs Go and caches modules and the build cache. The version is read from `<working-directory>/go.mod` unless `go-version` is set. Caching is skipped for modules without a `go.sum`, or with `cache: false`.

> **Prerequisite**: Run `actions/checkout` before calling this action.

| Input               | Description                                                  | Default |
| :------------------ | :----------------------------------------------------------- | :------ |
| `go-version`        | Go version; takes precedence over `go-version-file`          | `""`    |
| `go-version-file`   | Version file relative to `working-directory`                 | `go.mod` |
| `working-directory` | Directory containing `go.mod`                                | `"."`   |
| `cache`             | Cache modules and build output: `true` or `false`            | `"true"` |

| Output       | Description           |
| :----------- | :-------------------- |
| `go-version` | Installed Go version  |

## Lint Go (`lint`)

Runs [golangci-lint](https://golangci-lint.run) through `golangci/golangci-lint-action`, which caches its own binary and analysis. Formatting is checked only when the config enables formatters (see [Recommended config](#recommended-golangciyml)). golangci-lint finds `.golangci.y(a)ml` in `working-directory` or any parent directory, so a repo-root config works for a module in a subdirectory; set `config` for one elsewhere (e.g. `.github/linters/.golangci.yml`).

> **Prerequisite**: `actions/checkout` and `setup`.

| Input               | Description                                                            | Default         |
| :------------------ | :--------------------------------------------------------------------- | :-------------- |
| `working-directory` | Directory containing `go.mod`                                          | `"."`           |
| `version`           | golangci-lint version                                                  | pinned `v2.15.0` |
| `config`            | golangci-lint config path relative to the repo root (empty searches `working-directory` and its parents) | `""` |
| `args`              | Additional golangci-lint arguments (e.g. `--timeout=5m`)               | `""`            |

### Recommended `.golangci.yml`

Put it at the repo root. `gofmt` with `simplify` replaces a separate `gofmt -s` check, and the same `exclusions` cover linters and formatters. Generated files (`// Code generated ... DO NOT EDIT.`) and `vendor/` are skipped by default.

```yaml
version: "2"
formatters:
    enable:
        - gofmt
    settings:
        gofmt:
            simplify: true
    # exclusions:
    #     paths:
    #         - third_party/
```

## Test Go (`test`)

Runs `go test -coverprofile -covermode=atomic` through [gotestsum](https://github.com/gotestyourself/gotestsum) (via `go run`, nothing to install), writes `unit-tests.xml` and `coverage.out` into `working-directory`, and rewrites the profile's module import paths to **repo-relative** paths so PR diff-based coverage tools match files when `go.mod` is not at the repo root. Environment variables set on the calling step (e.g. a test database DSN) reach the tests.

> **Prerequisite**: `actions/checkout` and `setup`. One module per `working-directory`.

Notes:
- `coverage.out` and `unit-tests.xml` are written into `working-directory`; add them to `.gitignore`.
- `race: true` needs cgo; set `race: false` where `CGO_ENABLED=0` (e.g. Alpine containers).
- gotestsum is built with `go run`, so it needs module proxy access and fails under `GOFLAGS=-mod=vendor`.
- Tested on Linux and macOS runners.

| Input               | Description                                                   | Default         |
| :------------------ | :------------------------------------------------------------ | :-------------- |
| `working-directory` | Directory containing `go.mod`                                 | `"."`           |
| `packages`          | Packages to test, split on whitespace                         | `"./..."`       |
| `args`              | Additional `go test` flags (e.g. `-tags=integration -count=1`) | `""`           |
| `race`              | Race detector: `true` or `false`                              | `"true"`        |
| `gotestsum-version` | gotestsum version                                             | pinned `v1.13.0` |

| Output          | Description                                                     |
| :-------------- | :-------------------------------------------------------------- |
| `coverage`      | Total statement coverage, e.g. `87.5`; empty without a profile  |
| `coverage-file` | Coverage profile path relative to the repo root                 |
| `junit-file`    | JUnit XML path relative to the repo root                        |

### Usage

```yaml
steps:
    - uses: actions/checkout@<sha> # v7.0.1
    - uses: dotactions/go/setup@<sha> # v0.1.0
      with:
          working-directory: code/backend
    - uses: dotactions/go/lint@<sha> # v0.1.0
      with:
          working-directory: code/backend
    - id: test
      uses: dotactions/go/test@<sha> # v0.1.0
      with:
          working-directory: code/backend
      env:
          TEST_DB_DSN: host=localhost user=app dbname=test sslmode=disable
```

## Go CI (`ci.yml`)

Reusable workflow: parallel **Lint** and **Test** jobs built from the actions above. Test uploads `go-test-results` and `go-coverage` artifacts. Unless disabled, a separate **Coverage report** job (the only job with write permissions, so the code under test never runs next to a write token) posts a coverage PR comment and `go` status via [getsentry/coverage-action](https://github.com/getsentry/coverage-action). The report is skipped on PRs from forks, whose token is read-only. To call it more than once per workflow (e.g. a matrix over modules), give each call a distinct `name-suffix`: it is appended to the artifact names and the coverage flag, comment and status keys.

Use the actions directly instead when a job needs service containers or extra steps before the tests (a reusable workflow can't take those from the caller).

### Permissions Required

```yaml
permissions:
    contents: read
    actions: read # coverage-action reads base-branch artifacts
    pull-requests: write # coverage PR comment
    statuses: write # coverage status
```

GitHub checks these when the workflow starts, so they are required even with `coverage-report: false`.

### Inputs

| Name                    | Description                                                 | Default |
| :---------------------- | :---------------------------------------------------------- | :------ |
| `working-directory`     | Directory containing `go.mod`                               | `"."`   |
| `go-version`            | Go version (empty reads `go.mod`)                           | `""`    |
| `golangci-lint-version` | golangci-lint version (empty uses `lint`'s pinned version)  | `""`    |
| `lint-config`           | golangci-lint config path relative to the repo root (empty searches `working-directory` and its parents) | `""` |
| `lint-args`             | Additional golangci-lint arguments                          | `""`    |
| `test-args`             | Additional `go test` flags                                  | `""`    |
| `race`                  | Run tests with the race detector                            | `true`  |
| `name-suffix`           | Suffix for artifact names and coverage keys                 | `""`    |
| `coverage-report`       | Post coverage PR comment and status                         | `true`  |

| Output     | Description                          |
| :--------- | :----------------------------------- |
| `coverage` | Total statement coverage percentage  |

### Usage

```yaml
# .github/workflows/ci.yml
name: CI

on:
    push:
        branches: [main]
    pull_request:

concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

permissions:
    contents: read

jobs:
    go:
        uses: dotactions/go/.github/workflows/ci.yml@v0.1
        permissions:
            contents: read
            actions: read
            pull-requests: write
            statuses: write
```

## Go Vulnerability Audit (`govulncheck.yml`)

Reusable workflow: runs [govulncheck](https://go.dev/doc/security/vuln/) on a Go module and reports the findings through [`dotactions/sarif-reporter`](https://github.com/dotactions/sarif-reporter). The Go version is read from `<working-directory>/go.mod`.

In SARIF mode govulncheck exits `0` even when it finds vulnerabilities, so the job fails only through `fail-on-findings`.

### Permissions Required

```yaml
permissions:
    contents: read
    pull-requests: write # sticky PR comment
    issues: write # tracking issue; required even if manage-issue is false
```

### Inputs

| Name                | Description                                    | Required | Default |
| :------------------ | :--------------------------------------------- | :------- | :------ |
| `working-directory` | Directory containing `go.mod`                  | No       | `"."`   |
| `fail-on-findings`  | Fail the workflow if findings are detected     | No       | `false` |
| `upload-artifact`   | Upload the SARIF report as `govulncheck-sarif` | No       | `true`  |
| `manage-issue`      | Create/update/close tracked GitHub issue       | No       | `false` |
| `post-comment`      | Create/update the sticky PR comment            | No       | `true`  |

Secret `token`: GitHub token with pull-requests and issues write permissions (optional, defaults to `github.token`).

### Usage

```yaml
# .github/workflows/govulncheck.yml
name: Go Vulnerabilities

on:
    push:
        branches: [main]
        paths: &paths
            - "**.go"
            - "go.mod"
            - "go.sum"
    pull_request:
        branches: [main]
        paths: *paths
    schedule:
        - cron: "0 4 * * 4" # weekly Thursday at 04:00 UTC
    workflow_dispatch:

concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

jobs:
    govulncheck:
        uses: dotactions/go/.github/workflows/govulncheck.yml@v0.1
        permissions:
            contents: read
            pull-requests: write
            issues: write
        with:
            working-directory: code/backend
            manage-issue: ${{ github.event_name == 'schedule' || github.event_name == 'workflow_dispatch' }}
```

A scheduled run matters here: a new advisory can affect code that hasn't changed.

## Releasing (maintainers)

Go to **Actions → Release → Run workflow** on `main` and enter a version (`0.1.0`, or `0.2.0-rc.1` for a pre-release). `internal-release.yml` (via [dotactions/tooling](https://github.com/dotactions/tooling)) rewrites the `@main` self-references in `ci.yml` to the release tag, tags, creates the release and moves the floating tags. Don't draft releases through the UI: it does neither.

While on `0.x`, bump the **minor** version for breaking changes and the **patch** version for everything else.

The workflow needs the `RELEASE_APP_PRIVATE_KEY` secret, the `RELEASE_APP_CLIENT_ID` org variable, and the release app installed on this repo with **Contents** and **Workflows** read & write.
