# Testing & coverage

The module is tested to **100% statement coverage, including every error
branch**, and the coverage gate is enforced in CI.

```sh
COVERPKG=$(go list ./... | paste -sd, -)
go test -race -coverpkg="$COVERPKG" -coverprofile=cover.out ./...
go tool cover -func=cover.out | tail -1   # 100.0%
```

## The hermetic round-trip

`cmd/fake-lsp` is a deterministic LSP-like subprocess: it reads `Content-Length`
-framed JSON-RPC on stdin and writes canned responses on stdout
(`initialize`, `completion`, `hover`, `definition`, and a `didOpen` →
`publishDiagnostics` notification). The integration test builds it, points
`LSPBRIDGE_FAKE` at it, and drives a real WebSocket through `HandleWS` the way a
CodeMirror SPA would — exercising the whole **WS → subprocess → WS** path without
depending on texlab/gopls/etc. being installed.

## Reaching the error branches

The otherwise-unreachable failure paths are covered with small seams:

- **Framing pumps** are extracted as standalone functions over an `io.Reader` /
  `io.Writer` and a narrow WS interface, so an in-memory fake can force framing
  errors, short bodies, and write failures on demand.
- **Pipe/spawn failures** (`StdinPipe`, `StdoutPipe`, `Start`) are injected via
  method-value seams to confirm the bridge emits the right JSON-RPC error frame.
- **Concurrency caps** are unit-tested directly (global saturation, per-user
  ceiling, and the release path that deletes the counter at zero).

## Six arches, gated exec

CI builds and tests on all six 64-bit Go targets: amd64 and arm64 run natively
(the full round-trip runs there), while riscv64, loong64, ppc64le and s390x run
under `qemu-user`.

A host-built `fake-lsp` can't be exec'd for a foreign arch under `qemu-user`, and
there's no real loopback net, so the qemu lanes set **`LSPBRIDGE_NO_EXEC=1`**,
which skips the exec/WebSocket tests. The pure framing, registry and concurrency
logic still runs on every arch — so the cross-arch lanes validate the portable
core, and the native ubuntu/macos lane enforces the 100% gate over the whole
suite.
