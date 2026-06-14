# CLAUDE.md

Guidance for Claude Code (claude.ai/code) when working in this repository.

## Project Overview

Ollama is a Go application for running large language models locally. It ships a single
`ollama` binary that serves an HTTP API, provides a CLI / interactive TUI, and manages model
downloads, conversion, and inference. Inference runs through native C/C++/Metal backends
(llama.cpp + GGML, and MLX on Apple Silicon) that are vendored and built via CMake, then
driven from Go over CGO and a `llama-server` subprocess.

This is a fork of `ollama/ollama`. The main divergence from upstream is the **`anthropic/`
package** (plus `middleware/anthropic.go`), which adds an Anthropic Messages API
(`POST /v1/messages`) compatibility layer on top of Ollama's chat engine. See
[Fork-Specific Notes](#fork-specific-notes).

- Module: `github.com/ollama/ollama` (see `go.mod`)
- Go version: **1.26.0** (CI uses `go-version-file: go.mod` — match that)
- Entry point: `main.go` → `cmd.NewCLI()` (Cobra)

## Building

Prerequisites: Go, CMake **3.24+**, a C/C++ compiler (Clang on macOS, VS 2022 tools on
Windows, GCC/Clang on Linux), and ideally Ninja in `PATH`.

```shell
# Pure-Go iteration against an already-built native payload (fastest loop):
go run . serve

# Full native build from a fresh checkout or after changing native code.
# Builds the Go binary at repo root + installs the runtime payload to build/lib/ollama.
# macOS arm64 → Metal/MLX inference; everything else → CPU-only by default.
cmake -B build .
cmake --build build --parallel 8
./ollama serve
```

GPU / accelerated backends are opt-in via CMake flags:

```shell
# Select llama.cpp GPU backends explicitly (cuda_v12, cuda_v13, rocm_v7_1, rocm_v7_2,
# vulkan, cuda_jetpack5, cuda_jetpack6):
cmake -B build . -DOLLAMA_LLAMA_BACKENDS="cuda_v13;vulkan"

# MLX engine backends on non-macOS (e.g. CUDA 13+/cuDNN 9+):
cmake -B build . -DOLLAMA_MLX_BACKENDS=cuda_v13
```

> CGO data structures can drift and cause crashes. If you hit unexplained native crashes
> after pulling, run `go clean -cache` then rebuild.

See `docs/development.md` for per-platform details (Xcode Metal toolchain, Vulkan SDK,
local MLX source overrides via `OLLAMA_MLX_SOURCE` / `OLLAMA_MLX_C_SOURCE`, Docker builds).

Pinned native versions live in root files: `LLAMA_CPP_VERSION`, `MLX_VERSION`, `MLX_C_VERSION`.

## Running

```shell
./ollama serve              # start the API server (default :11434)
./ollama run gemma3         # pull (if needed) + chat with a model
ollama                      # interactive launcher / TUI
ollama launch claude        # launch a coding integration (claude, codex, opencode, ...)
```

## Testing

```shell
go test ./...                              # all unit tests
go test -count=1 -benchtime=1x ./...       # as run in CI (no test cache, benchmarks once)
go test ./anthropic/ ./middleware/         # the fork's Anthropic layer
go test -count=1 -tags updater_live ./app/...   # app tests with the live-updater build tag
```

Integration tests are guarded by a build tag and are **not** run by default:

```shell
go test -tags integration ./integration/...
```

Some tests depend on generated wrappers; CI runs `go generate ./...` before `go test`.
Test conventions (from `CONTRIBUTING.md`): include tests, and test *behavior, not
implementation*.

## Linting / Formatting

```shell
golangci-lint run            # CI uses golangci/golangci-lint-action@v9
go mod tidy                  # CI fails if `go mod tidy --diff` is non-empty — keep go.mod/go.sum tidy
```

Config is `.golangci.yaml` (v2 schema). Formatters enabled: **gofmt** and **gofumpt**
(gofumpt is stricter than gofmt — run it). Notable: `errcheck` is disabled, `staticcheck`
runs `all` minus a curated suppression list, and `gofmt`/`goimports`/`intrange` findings are
informational rather than errors.

## Architecture / Directory Layout

| Path | Purpose |
|------|---------|
| `main.go` | Tiny entry point; delegates to `cmd.NewCLI()`. |
| `cmd/` | Cobra CLI, interactive REPL (`interactive.go`), TUI (`tui/`), launcher (`launch/`), `serve`/`start` wiring. |
| `cmd/launch/` | Coding-tool integrations (`claude`, `codex`, `opencode`, `copilot`, `droid`, `openclaw`, `cline`, etc.). |
| `api/` | Go API client + shared request/response types (`api.ChatRequest`, `api.Message`, `api.Tool`, ...). |
| `server/` | HTTP server, route registration (`routes.go`), model scheduling, registry/download, cloud proxy. |
| `middleware/` | gin middleware translating external API shapes to Ollama's: `openai.go`, **`anthropic.go`**. |
| `openai/` | OpenAI-compatible API types/conversion (`/v1/chat/completions`, etc.). |
| `anthropic/` | **Fork addition** — Anthropic Messages API types, conversion, streaming, tracing. |
| `llm/` | Native runtime/subprocess management glue. |
| `llama/` | Vendored llama.cpp bindings + `llama/server/` CMake build. |
| `ml/`, `model/`, `kvcache/` | ML backend abstractions, model graph definitions, KV cache. |
| `convert/` | Convert HF/safetensors/GGUF source models into Ollama's format. |
| `runner/`, `cmd/runner/` | The inference runner process. |
| `harmony/`, `thinking/`, `template/`, `tools/`, `parser/` | Prompt templating, reasoning/"thinking" parsing, tool-call parsing, Modelfile parsing. |
| `discover/` | GPU/CPU/accelerator detection (CUDA, ROCm, Vulkan, Metal). |
| `fs/`, `manifest/`, `types/`, `format/`, `envconfig/` | Filesystem/GGUF helpers, manifests, shared types, formatting, env config. |
| `auth/`, `internal/`, `middleware/` | Auth, internal-only packages (e.g. `internal/cloud`), request middleware. |
| `app/`, `x/` | Desktop app shell and experimental/extra packages. |
| `cmake/`, `CMakeLists.txt`, `CMakePresets.json` | Native build orchestration. |
| `scripts/` | Platform build scripts (`build_linux.sh`, `build_darwin.sh`, `build_windows.ps1`, Docker, install). |
| `docs/` | User + developer docs (`development.md` is the build bible). |
| `integration/` | Integration tests behind `//go:build integration`. |

Native runtime payloads are discovered at runtime in `lib/ollama` (relative to the binary)
or `build/lib/ollama` / `dist/<platform>/lib/ollama` for dev builds — see "Library detection"
in `docs/development.md`.

## Fork-Specific Notes

The `anthropic/` package and `middleware/anthropic.go` implement Anthropic Messages API
compatibility — they are the primary divergence from upstream `ollama/ollama`. When touching
this layer:

- **Route:** `POST /v1/messages` is registered in `server/routes.go` and runs the
  `middleware.AnthropicMessagesMiddleware()` before the shared `s.ChatHandler`. It converts
  Anthropic `MessagesRequest` ⇄ Ollama `api.ChatRequest`/`api.ChatResponse`, including
  streaming (`anthropic.StreamConverter`) and error mapping (`anthropic.NewError`).
- **Types** (`anthropic/anthropic.go`): `MessagesRequest`, `MessageParam`, `Tool`,
  `ThinkingConfig`, `ToolChoice`, `MessagesResponse`, `Usage`, etc., mirror Anthropic's wire
  format (string-or-block `system`, content blocks like `text`/`thinking`/`tool_use`/
  `tool_result`/`image`).
- **Tracing** (`anthropic/trace.go`): compact, truncated trace helpers used for `logutil.Trace`
  logging on both the Anthropic and Ollama (`api.*`) sides. Truncation limits are constants
  (`TraceMaxStringRunes`, `TraceMaxSliceItems`, `TraceMaxMapEntries`, `TraceMaxDepth`).
- Keep this layer backwards-compatible with the Anthropic API surface, and prefer reusing
  the shared `api.*` types over inventing parallel ones. Add tests in
  `anthropic/anthropic_test.go` / `middleware/anthropic_test.go`.

## Key Conventions

- **Commit messages** (`CONTRIBUTING.md`): `<package>: <short description>`, where
  `<package>` is the most-affected Go package (or directory/file name if no Go code changed).
  The description starts lowercase and continues "This changes Ollama to...". Example:
  `llm/backend/mlx: support the llama architecture`. Avoid `feat:`/`fix:`/`chore:` prefixes.
- **Non-trivial changes:** open an issue to discuss before a large PR. Changes that break
  API backwards compatibility (including the OpenAI-compatible API) are generally not accepted.
- **Dependencies:** add sparingly; justify any new dependency.
- **CGO/native drift:** be aware Go ↔ native struct mismatches can cause silent crashes;
  rebuild native code (`go clean -cache`) when changing CGO boundaries.
- Run `gofumpt`, `golangci-lint run`, and `go mod tidy` before committing.

## Do Not

- Commit or push unless explicitly asked.
- Edit vendored native code under `llama/` casually — it is pinned to `LLAMA_CPP_VERSION` /
  `MLX_VERSION` and synced from upstream.
- Break the OpenAI-compatible (`openai/`) or Anthropic-compatible (`anthropic/`) API shapes.
