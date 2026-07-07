# Contributing

Contributions are welcome. The bar is a green CI and the 100% coverage gate.

## Ground rules

- **Pure Go, zero cgo.** The only allowed module dependency is
  `github.com/coder/websocket`.
- **100% coverage, including error branches.** New code needs tests that reach
  every branch; the CI gate fails below 100%.
- **`gofmt` + `go vet` clean.**
- **BSD-3-Clause**, English-only public content.

## Local workflow

```sh
git clone git@github.com:go-lsp-bridge/lspbridge.git
cd lspbridge
go vet ./...
COVERPKG=$(go list ./... | paste -sd, -)
go test -race -coverpkg="$COVERPKG" -coverprofile=cover.out ./...
go tool cover -func=cover.out | tail -1   # want 100.0%
```

The exec/WebSocket tests need to spawn `cmd/fake-lsp` and open a loopback
socket. If you are running cross-arch under `qemu-user`, set
`LSPBRIDGE_NO_EXEC=1` to skip them (the pure logic still runs).

## Docs

This site is MkDocs Material, versioned with [mike]. Preview locally:

```sh
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
mkdocs serve      # http://localhost:8000
```

[mike]: https://github.com/jimporter/mike
