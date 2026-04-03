# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`bb` is a Go CLI for the Bitbucket API, built with [Cobra](https://github.com/spf13/cobra). The binary is named `bb`. Module path: `bitbucket.org/gildas_cherruel/bb`. Version is defined in `version.go`.

## Build & Development Commands

```bash
# Build for current platform
make build

# Run tests (all)
make test

# Run a specific test or suite
make test what='TestPullRequestSuite/TestCanUnmarshal'

# Lint (golangci-lint)
make lint

# Format code
make fmt

# Run go vet
make vet

# Install to $GOPATH/bin or /usr/local/bin
make install

# Update Go modules
make dep

# Build for all platforms (darwin/amd64, darwin/arm64, linux/amd64, linux/arm64, windows/amd64, windows/arm64)
make build  # calls __build_all__
```

Coverage output lands in `tmp/coverage/`. Binaries land in `bin/<GOOS>/<GOARCH>/bb`.

## Architecture

### Command Structure

Each Bitbucket resource lives in its own package under `cmd/`:

```
cmd/
  root.go          # RootCmd (cobra), global flags, viper config loading, profile init
  profile/         # Auth profiles — the central auth/HTTP layer
  common/          # Shared types and utilities used by all command packages
  pullrequest/     # bb pullrequest (aliases: pr, pull-request)
  repository/      # bb repo
  branch/          # bb branch
  workspace/       # bb workspace
  project/         # bb project
  pipeline/        # bb pipeline
  issue/           # bb issue
  commit/          # bb commit
  artifact/        # bb artifact
  component/       # bb component
  user/            # bb user
  ssh-key/         # bb ssh-key
  gpg-key/         # bb gpg-key
  cache/           # bb cache
  remote/          # git remote parsing helpers
```

Each resource package follows the pattern: one file per subcommand (`create.go`, `get.go`, `list.go`, `delete.go`, `update.go`) plus a main `<resource>.go` defining the struct, `Command`, and output methods.

### Profile (Auth) System

`cmd/profile/profile.go` — `Profile` struct holds all auth credentials (user/password, OAuth2 client ID/secret, access token). `profile.Current` is the global current profile set during `cobra.OnInitialize`.

`cmd/profile/profile_client.go` — HTTP transport layer. All API calls go through `Profile.Get/Post/Put/Patch/Delete`. `GetAll[T]()` is the generic paginated fetcher used throughout the codebase. The Bitbucket API root defaults to `https://api.bitbucket.org/2.0`.

Config file: `$XDG_CONFIG_HOME/bitbucket/config-cli.yml` (YAML), or `~/.bitbucket-cli` (legacy). Env var `BB_CONFIG` overrides. Credentials also stored in the OS keyring via `go-keyring`.

### Output System

`cmd/common/tableable.go` — `Tableable` and `Tableables` interfaces that every resource struct implements (`GetHeaders`, `GetRow`/`GetRowAt`). `Profile.Print()` dispatches to `PrintTable/PrintJSON/PrintYAML/PrintCSV/PrintTSV` based on `--output` flag or profile's `outputFormat`.

`cmd/common/column.go` — Generic `Columns[T]` type used within each resource package to define sortable/filterable column metadata, powering `--columns` and `--sort-by` flags.

### Dry-Run Pattern

`cmd/common/whatif.go` — `common.WhatIf(ctx, cmd, format, ...)` gates any mutating operation. Returns `false` (skip) when `--dry-run` is set. All `create`, `delete`, `update`, `merge`, `approve` commands call this before executing.

### Error Handling

`cmd/common/error_processing.go` — `ErrorProcessing` enum (`stop-on-error`, `warn-on-error`, `ignore-errors`). Commands that accept multiple arguments loop over them and respect this setting from the profile or CLI flag.

### Testing Conventions

Tests use `github.com/stretchr/testify/suite`. Each package has a `*_test.go` in the `package foo_test` external test package. JSON fixtures live in `testdata/` at the repo root and are loaded via `os.ReadFile("../../testdata/<file>.json")`. Test suites follow the pattern in `cmd/pullrequest/pullrequest_test.go`.

## Key Dependencies

| Package | Purpose |
|---|---|
| `github.com/spf13/cobra` | CLI framework |
| `github.com/spf13/viper` | Config file loading |
| `github.com/gildas/go-request` | HTTP client wrapping API calls |
| `github.com/gildas/go-logger` | Structured logging (context-carried) |
| `github.com/gildas/go-errors` | Typed errors (`ArgumentMissing`, `ArgumentInvalid`, etc.) |
| `github.com/gildas/go-flags` | `EnumFlag`, `EnumSliceFlag` for validated flag values with tab completion |
| `github.com/go-git/go-git/v5` | Reading local git config (workspace/repo auto-detection) |
| `github.com/zalando/go-keyring` | OS keyring for credential storage |
| `github.com/gildas/go-core` | Utility helpers: `Map`, `Filter`, `Sort`, `GetEnvAsString`, etc. |
| `gopkg.in/yaml.v3` | YAML marshaling (config file read/write) |

## Linting

golangci-lint with `.golangci.yaml`. Configuration: standard linters enabled, `testdata/` and `tmp/` excluded, `nakedret` disabled in test files. Issues are scoped to `new-from-rev: HEAD~1` (only new issues since last commit).

```bash
golangci-lint run ./...
```
