# AGENTS.md

Guidance to AI Agents working with code in this repository.

## What this is

Open Code Review (`ocr`, module `github.com/open-code-review/open-code-review`) is a Go CLI that reviews git
diffs (or whole files, via `ocr scan`) by driving an LLM through a tool-use loop and emitting line-precise
review comments. It's distributed via npm (wraps a downloaded/optional-dependency native binary), a GitHub
composite Action, a VS Code extension, and Claude Code/Codex/Cursor plugin skills — all in this one repo.

Two remotes exist: `origin` (`alibaba/open-code-review`, upstream project) and `upstream`
(`nigel-upstart/open-code-review`, this fork's primary remote).

## Commands

```bash
make build              # go build ./cmd/opencodereview -> ./dist/opencodereview
make test               # LC_ALL=C go test -v -race -count=1 $(go list ./... | grep -v /extensions/)
make coverage           # same test run + enforces 80% total coverage threshold
make check              # go mod tidy && go fmt && go vet — run before committing
make fmt / make vet
make run                # builds then runs `opencodereview --staged`
make build-all          # cross-compile all release platforms into ./dist
```

Single test / single package (not wired into any Makefile target, but this is how CI effectively runs things):

```bash
go test ./internal/tool/ -run TestReadLines_Disk_FullFile -v
LC_ALL=C go test -race -count=1 ./internal/agent/...
```

`LC_ALL=C` matters — locale-dependent string sorting in some tests will flake without it. There is no
golangci-lint config; `go vet` + `go fmt` + `govulncheck` (CI only, not vendored) is the entire lint story.

`go.mod` pins `go 1.25.5`, but `.github/workflows/ci.yml` and `release.yml` both build inside
`golang:1.26.5` — don't be surprised if CI behaves slightly differently from a local build matching
`go.mod` exactly.

Other subtrees have their own build/test tooling, deliberately excluded from `make test`
(`Makefile`'s `PACKAGES` greps out `/extensions/`):
- `extensions/vscode` — separate npm package, `webpack`/`jest`/`eslint`.
- `pages/` — the docs/landing site, separate npm package (`npm run dev` / `build` / `typecheck`).

Test conventions: stdlib `testing` only (no testify), table-driven tests with `t.Run` subtests, no
`testdata/` dirs (fixtures built with `t.TempDir()` + small helpers), naming like
`TestReadLines_Disk_FullFile`.

## Architecture

### CLI entry point

`cmd/opencodereview/main.go` hand-rolls subcommand dispatch (no cobra) over `os.Args[1:]`:
`review`/`r`, `scan`/`s`, `config`, `llm`, `rules`, `viewer`, `session`, `version`. Each subcommand lives in
its own `*_cmd.go` file. `flags.go` wraps stdlib `flag` to add short-flag aliases (`-c` → `--commit`).
`shared.go` builds the common `commonContext`/`llmRuntime` (resolved LLM endpoint, tool registry,
template/rules) used by both `review_cmd.go` and `scan_cmd.go`.

### Two review pipelines sharing one tool-use loop

- **`internal/agent`** — diff-based review (`ocr review`). `Agent.Run` loads diffs (`internal/diff`),
  applies deterministic file filtering (`internal/config/allowlist` + `internal/config/rules`), then
  `dispatchSubtasks` launches **one goroutine per changed file** (bounded by `--concurrency`, default 8).
  Each file gets its own conversation: optional Plan Phase (skipped below
  `Template.PlanModeLineThreshold`, ~50 changed lines) → main tool-use loop → a post-hoc Review Filter Task
  that lets the LLM strip its own provably-wrong comments.
- **`internal/scan`** — whole-file review (`ocr scan`), no git diff involved. `scan.Provider.Enumerate`
  walks the filesystem instead of diffing; each `ScanItem.AsDiff()` adapts into a synthetic `model.Diff` so
  the rest of the pipeline (line resolution, the shared loop) is agnostic to which mode produced it. Adds
  scan-only features: batching (`by-language`/`by-directory`/`none`, sequential batches / concurrent files
  within a batch), pre-run cost estimation with a token budget gate, and an optional final
  Project Summary pass.
- **`internal/llmloop.Runner`** is the actual tool-use loop used by *both* pipelines (`RunPerFile`): repeat
  LLM call → parse tool calls → execute → append results, until a `task_done` tool call, up to
  `Template.MaxToolRequestTimes` rounds. Handles context-window management via a 3-zone token budget (soft
  threshold triggers async compression, hard threshold forces sync compression or aborts the file), and
  routes `code_comment` calls through a `CommentWorkerPool` so comment post-processing (line resolution) is
  async and doesn't block the loop.

There is currently **no cross-file "bundling"** on `main` — each changed file is its own independent
subtask; cross-file context is limited to a `{{change_files}}` listing and an on-demand `file_read_diff`
tool. The README's description of grouping related files (e.g. `message_en.properties` +
`message_zh.properties`) into one review unit lives only on the unmerged branch `feat/grouping` — don't
assume it's implemented when reading README claims against `main`.

### LLM client layer (`internal/llm`)

Single `LLMClient` interface, three protocols (`anthropic`, `openai`, `openai-responses`), one concrete
client per protocol, selected by `NewLLMClient`. `providers.go` has a hardcoded registry of ~16 built-in
provider presets (name → base URL/auth header/protocol/models); users can also register arbitrary
`custom_providers.*` via `ocr config set`. Endpoint resolution (`resolver.go`) tries, in order: the OCR
config file, `OCR_LLM_*` env vars, Claude Code's own `ANTHROPIC_*` env vars, then shell rc scraping — so
this tool can piggyback on a Claude Code Anthropic setup with zero extra config.

### Tools exposed to the LLM (`internal/tool`)

`file_read`, `file_find`, `file_read_diff` (diff or full content depending on mode, via a frozen
`DiffMap`), `code_search` (via `git grep`), `code_comment` (submits findings), plus the sentinel
`task_done`. Tool *schemas* (JSON-schema params, plan/main phase flags) live in
`internal/config/toolsconfig/tools.json`, not in Go — the Go `Provider` types are execution only.
`internal/mcp` lets additional tools be registered from external MCP servers over stdio
(`ocr config set mcp_servers.<name>.*`); OCR is an MCP *client* only on `main` (ROADMAP.md's mention of MCP
server support is aspirational, not shipped).

### Config, rules, and templates (`internal/config/*`)

- **`allowlist`** — default extension/path filters (`supported_file_types.json`, `default_exclude_patterns.json`).
- **`rules`** — *review* rule matching (not linting): glob path → markdown rule doc, resolved with 4-layer
  priority (`--rule` flag > `<repo>/.opencodereview/rule.json` > `~/.opencodereview/rule.json` > embedded
  system default via `system_rules.json`). `merge_system_rule: true` appends to rather than overrides a
  matched system rule.
- **`template`** — prompt templates (`task_template.json` for review, `scan_template.json` for scan) with
  placeholder substitution (`{{diff}}`, `{{system_rule}}`, `{{change_files}}`, etc.) and budget knobs
  (`MaxTokens`, `MaxToolRequestTimes`, `PlanModeLineThreshold`).
- **`toolsconfig`** — the tool JSON schemas mentioned above.
- The actual user config file (`~/.opencodereview/config.json`) is read/written by
  `cmd/opencodereview/config_cmd.go`, not by anything under `internal/config/`.

### Diff/git plumbing

All git subprocess calls go through `internal/gitcmd.Runner`, a semaphore-bounded
(`--max-git-procs`, default 16) wrapper around `exec.CommandContext` — never a shell, by design (see
`ASSURANCE_CASE.md` T1). `internal/diff` builds diffs for three modes (workspace / commit / range) and
resolves an LLM's `existing_code` snippet back to line numbers, falling back to an LLM-driven
"relocation" call when text matching fails. `internal/pathutil` enforces that all file-tool reads stay
within the repo root (path-traversal guard).

### Other internal packages

`internal/session` persists a JSONL trace per run under `~/.opencodereview/sessions/` and backs
`--resume`/`ocr session list`. `internal/viewer` is a local-only HTTP server for browsing those session
files, with a Host-header allowlist to block DNS rebinding. `internal/telemetry` is an OpenTelemetry
wrapper, config'd via `ocr config set telemetry.*`. `internal/suggestdiff` is presentation-only (ANSI
before/after rendering), not part of the review pipeline.

### npm distribution

Root `package.json` declares `bin/ocr.js` as the `ocr` shim and 6 per-platform `optionalDependencies`
(`@alibaba-group/ocr-<os>-<arch>`) under `npm/<platform>/`. `scripts/install.js` (postinstall) prefers the
optional-dependency binary; if npm didn't resolve one, it downloads from GitHub Releases and verifies a
SHA-256 checksum before making it executable. `bin/ocr.js` resolves the binary via `scripts/platform.js`
and `spawnSync`s it, propagating exit code, plus a background update-check.

### Other entry points

- `action.yml` — composite GitHub Action; installs `ocr` via npm, runs
  `ocr review --from <merge-base> --to <head-sha> --format json`, then a Node script
  (`scripts/github-actions/post-review-comments.js`) posts inline PR comments / a sticky summary.
- `examples/{github_actions,gitlab_ci,gitflic_ci}` — equivalent CI recipes for other platforms.
- `plugins/open-code-review/` + `skills/open-code-review/` — near-duplicate `SKILL.md` files (kept manually
  in sync, not symlinked, since plugin installs may only materialize the plugin subtree) instructing an
  agent host to run `ocr review --audience agent --background "..."`, never `--audience human`, and to
  triage resulting comments by severity before auto-fixing.
- `extensions/vscode` and `pages/` — see Commands section above; both are independent npm apps.

## Release process

Tag `vX.Y.Z` → `release.yml` cross-compiles all platform binaries, generates release notes from
Conventional Commit messages since the previous tag, signs/attests binaries via Sigstore (keyless OIDC),
then publishes each `npm/<platform>` package and the root package. Commit messages
(`<type>(<scope>): <summary>`) are therefore load-bearing for release notes, not just style — keep to
Conventional Commits.
