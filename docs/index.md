# go-lsp-bridge documentation

**A pure-Go (no cgo) bridge that relays JSON-RPC LSP traffic between a
WebSocket and a language-server subprocess's stdio.** The module path is
`github.com/go-lsp-bridge/lspbridge`.

It owns the `Content-Length` framing on the stdio side so each WebSocket text
frame carries one **bare JSON object** — exactly what browser LSP clients such
as [`codemirror-languageserver`](https://github.com/FurqanSoftware/codemirror-languageserver)
and `@open-rpc/codemirror-lsp` expect. A web editor gets real completions,
hovers, diagnostics and go-to-definition from an actual language server
(texlab, gopls, pyright, typescript-language-server, rust-analyzer, …) behind a
single HTTP handler.

The only dependency is [`github.com/coder/websocket`](https://github.com/coder/websocket).

!!! success "Status: complete and released"
    `HandleWS` relays JSON-RPC in both directions with correct `Content-Length`
    framing; `DefaultServers()` ships launchers for latex / go / python /
    typescript / javascript / rust with neutral `LSPBRIDGE_*` env overrides;
    `WithSubject` drives the per-user concurrency cap; `AvailableLanguages()`
    reports resolvable servers; `EncodeError` surfaces setup failures. Validated
    by a hermetic WS → subprocess → WS round-trip against the bundled
    `cmd/fake-lsp` stub at **100% coverage including error branches**, `gofmt` +
    `go vet` clean, CI green across the six 64-bit Go targets.

## The wire shape

```
server → client : one JSON-RPC object per WS text frame
client → server : one JSON-RPC object per WS text frame
```

The bridge is the only party that touches the stdio framing. On the way in it
prepends `Content-Length: <n>\r\n\r\n` to each WS payload before writing it to
the server's stdin; on the way out it parses that header off the server's stdout
and forwards the bare body as a single WS text frame. The editor and the
language server never see each other's transport.

## A bridge, not a client

go-lsp-bridge transports JSON-RPC **untouched** — it never parses or rewrites
the LSP messages themselves, only the transport framing. That keeps it protocol
-version agnostic: as the LSP spec and individual servers evolve, the bridge
needs no changes.

## Quick taste

```go
http.HandleFunc("/lsp/", func(w http.ResponseWriter, r *http.Request) {
	lang := r.PathValue("lang")
	r = r.WithContext(lspbridge.WithSubject(r.Context(), subjectOf(r)))
	lspbridge.HandleWS(w, r, lang, slog.Default(), acceptOpts)
})
```

See [Wiring the bridge](usage.md) for the full handler, and
[Server registry & configuration](configuration.md) for choosing and overriding
language servers.

## Install

```sh
go get github.com/go-lsp-bridge/lspbridge
```

## License

BSD-3-Clause. Copyright the go-lsp-bridge/lspbridge authors.
