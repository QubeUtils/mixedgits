# Multi-Repo Monitor — CLI + Optional Web Dashboard

A production-ready, open-source tool (CLI-first, optional web dashboard) to monitor multiple Git repositories and their CI pipelines at once. Designed to scan 10–50 repos concurrently and report:

* local repo status (branch, uncommitted changes, unpushed commits, ahead/behind)
* remote sync state
* latest CI/CD workflow/pipeline results
* exportable JSON for dashboarding or automation

---

# Features

* Fast, concurrent scanning of many repos
* Native Git inspection (no shelling out required)
* Pluggable CI integrations (cloud & self-hosted)
* Clean CLI UX with table output and export modes (json, markdown)
* Optional web dashboard with live updates (SSE / WebSocket)
* Cross-platform single-binary distribution

---

# Tech Stack

* Go — language and runtime
* GitHub — repo hosting + Actions API
* GitLab — repo hosting + CI API
* Jenkins — CI server support
* CircleCI — CI provider
* Docker — local test environments / packaging
* Cobra — CLI framework
* go-git — native Git interactions
* Gin — dashboard API (optional)
* Fiber — alternative web framework option
* React — dashboard UI
* Tailwind — UI styling
* Viper — config parsing
* tablewriter — table output
* resty — CI API requests
* SQLite — optional local cache
* GitHub Actions — example CI integration
* GitLab CI — example CI integration

> The project is designed to be modular so each integration is a separate package and can be extended.

---

# Quickstart — from setting up the language to running the MVP

> These steps assume a typical Unix-like environment. Replace package-manager commands for your OS.

## 1. Prerequisites (local dev)

1. Install the language and toolchain (use the official installer for your OS, verify `go version`).
2. Install `git` and ensure it’s in your `PATH`.
3. Have API tokens ready for any CI providers you want to query (cloud CI tokens stored in environment variables).

## 2. Clone & bootstrap

```bash
git clone https://github.com/sasmithaK/mixedgit.git
cd mixedgit
# initialize modules
go mod download
```

## 3. Build the CLI (development build)

```bash
go build -o bin/mrm ./cmd/monitor
# run
./bin/mrm --help
```

## 4. Install for local use (optional)

```bash
go install ./cmd/monitor
# or build release binaries with cross-compilation:
# GOOS=linux GOARCH=amd64 go build -o releases/mrm-linux-amd64 ./cmd/monitor
```

---

# Project Structure

```text id="project-structure"
mixedgit/
├── cmd/                 # CLI entrypoints
│   └── monitor/
│       └── main.go      # CLI bootstrap with Cobra
├── internal/
│   ├── config/          # Config parsing and validation (viper)
│   │   └── config.go
│   ├── git/             # Native Git operations (go-git)
│   │   └── git.go
│   ├── ci/              # CI/CD API clients (GitHub, GitLab, Jenkins)
│   │   └── github.go
│   ├── cache/           # Optional local cache (SQLite)
│   │   └── cache.go
│   ├── output/          # CLI and JSON output rendering
│   │   └── table.go
│   └── server/          # Optional web dashboard backend
│       └── server.go
├── web/                 # Frontend for optional dashboard
│   ├── src/
│   ├── package.json
│   └── ...
├── configs/
│   └── repos.yaml       # Example repo configuration
├── go.mod
├── go.sum
├── README.md
└── LICENSE
```

---




---

# Configuration

Create a configuration file (YAML/JSON/TOML supported). Example `repos.yaml`:

```yaml
repos:
  - name: service-a
    path: /home/dev/projects/service-a
    ci:
      type: github
      repo: org/service-a
  - name: service-b
    path: /home/dev/projects/service-b
    ci:
      type: gitlab
      repo: org/service-b
scan:
  concurrency: 12
  fetch_remotes: true
output:
  format: table   # table | json | markdown
```

Set required tokens as environment variables (example):

```bash
export GITHUB_TOKEN="ghp_..."
export GITLAB_TOKEN="glpat-..."
```

---

# Usage Examples

Basic scan of configured repos:

```bash
mrm scan --config repos.yaml
```

Scan a parent directory auto-discovering repos:

```bash
mrm scan --path ~/projects
```

Export JSON:

```bash
mrm scan --config repos.yaml --output json > status.json
```

Run scheduled scan in the background (cron example):

```cron
*/10 * * * * cd /path/to/mrm && ./mrm scan --config repos.yaml --output json > /var/tmp/mrm-status.json
```

---

# Implementation Guide (Developer Steps)

This section summarizes the implementation strategy and the key modules to implement.

## Core modules

* `cmd/monitor/` — CLI entrypoint using the CLI framework; parse flags and call the engine.
* `internal/config/` — load and validate config using the config lib.
* `internal/git/` — native Git inspection (open repo, get current branch, worktree status, fetch, compute ahead/behind).
* `internal/ci/` — pluggable CI client interfaces (github, gitlab, jenkins, circleci).
* `internal/cache/` — optional local cache (sqlite) for storing last-known statuses and rate-limit control.
* `internal/output/` — table / json / markdown renderers.
* `internal/server/` — optional HTTP API for the dashboard (endpoints: `/status`, `/repo/:name`).
* `web/` — frontend app (React + Tailwind) that consumes the API and shows live updates.

## Concurrency model

* Use a worker pool (goroutines + channels) sized by `scan.concurrency`.
* Each worker scans one repo: open, fetch (if enabled), compute local diffs, query CI client async.
* Aggregate results via channel and render once all finished.

## Git details to implement

* Open repo with native library.
* Use Worktree.Status() to find uncommitted changes.
* To compute unpushed commits: compare local HEAD to `origin/<branch>` using commit graph traversal (rev-list equivalent).
* Avoid forcing network if user disabled fetch; otherwise `Fetch()` then compare refs.

## CI client considerations

* Respect provider rate limits; implement exponential backoff and caching.
* Query only the most recent workflow/pipeline run and extract: status, conclusion, duration, url to logs.
* Authenticate via token header (bearer or token style depending on provider).

## Error handling & UX

* Make outputs resilient: mark timeouts and failures clearly (color / emoji / codes).
* Always return machine-parseable output in `--output json` mode including partial results and error fields.

## Tests

* Unit test: Git module using fixture repos (create temp repos with commits & branches).
* Integration test: Mock CI responses; use recorded HTTP responses or test against a sandbox org.
* End-to-end: smoke test that scans a small set of real repos without tokens (read-only).

---

# Dashboard (optional)

* Backend exposes `/api/v1/status` which returns aggregated JSON.
* Use server-sent events (SSE) for low-overhead live updates or WebSockets for interactive features.
* Frontend shows:

  * grid of repos
  * filters (failing CI, unpushed, uncommitted)
  * detail pane for last commits and link to CI logs

Packaging hint: ship the backend as a single binary that can run in a small container for team deployment.

---

# CI for this project (example)

Create a pipeline to run on every PR:

* `go fmt` and `go vet`
* `golangci-lint` for linting
* `go test ./...` for unit tests
* Build test binary for core OS targets
* Optionally run frontend tests (if dashboard included)

Add badge(s) to README showing build status and test coverage.

---

# Release & Distribution

* Use cross-compile env vars for building OS/arch binaries.
* Provide GitHub Releases with tarballs for major versions.
* Consider using a release tool to sign binaries and create checksums.
* Provide Homebrew / scoop manifests for easy local install (later stage).

---

# Security & Secrets

* Never store provider tokens in repo. Use environment variables or secrets in CI.
* Follow least privilege for tokens (read-only scopes where possible).
* Sanitize logs to avoid printing tokens.

---

# Contribution Guidelines

* Fork → Feature branch → Open PR with clear description & tests.
* Use commit message convention: `feat:`, `fix:`, `chore:`, `docs:`.
* Run `make test` and `make lint` before creating a PR.
* Add an issue template and PR template in `.github/`.

---

# Roadmap & Extensions

* Add VS Code extension showing multi-repo status in the editor.
* Add batch Git operations (safe pull/push with confirmation).
* Add dependency graph awareness (which service depends on which).
* Add Slack/Discord alerts for failing CI or long-unpushed commits.
* Add self-hosted agent to run containerized integration tests.

---

# Troubleshooting

* If a repo fails to open: verify `path` in config points to a valid Git checkout.
* If CI results are missing: verify token env vars and provider repo identifier.
* If API rate limits: reduce concurrency or add caching in config.

---

# Example Commands Summary

```bash
# build
go build -o bin/mrm ./cmd/monitor

# scan configured repos
./bin/mrm scan --config repos.yaml

# export json
./bin/mrm scan --config repos.yaml --output json > status.json

# dev: run tests
go test ./...
```

---

# Documentation & Resources

* Official language tour and docs (install & env setup)
* CLI framework docs for building robust CLIs
* Native Git library docs and examples
* CI provider API docs for fetching workflow runs
* Web framework docs for optional dashboard server
* Guides on cross-compiling Go binaries and packaging

(Use the official docs for each provider and library for the most current and authoritative instructions.)

---

# License

Suggested: Apache-2.0 or MIT (pick one and add `LICENSE` file).

---

# Final Notes

This README contains the full project blueprint from local setup to release and maintenance. Use the modular structure described above to implement incrementally: prioritize a solid CLI MVP with robust Git handling, then layer CI integrations and finally an optional dashboard. Good tests, clear error handling, and careful token management will make the project industry ready and attractive as an open-source offering.

If you want, I can now generate:

* the initial repository skeleton (`go.mod`, `cmd/monitor/main.go`, minimal git module), and
* a working MVP `internal/git` module that scans a list of local repos concurrently and prints a table.

