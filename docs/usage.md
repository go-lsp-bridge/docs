# Wiring the bridge

`HandleWS` is the single entry point. Mount it on any `net/http` route that can
give you a **language key** and the request:

```go
func HandleWS(w http.ResponseWriter, r *http.Request, langKey string,
	logger *slog.Logger, acceptOpts *websocket.AcceptOptions)
```

- **`langKey`** — the key into the [server registry](configuration.md)
  (`"latex"`, `"go"`, …). Extract it however your router exposes it; the bridge
  returns `404` on an unknown key.
- **`logger`** — a `*slog.Logger`; the bridge logs spawn/exit and framing errors.
- **`acceptOpts`** — the same `*websocket.AcceptOptions` you use for your other
  WebSocket endpoints, so the Origin allowlist stays uniform. Pass `nil` for a
  permissive dev-mode default. Compression is always forced off (large
  `initialize` responses).

## A complete handler

```go
package main

import (
	"log/slog"
	"net/http"

	"github.com/coder/websocket"
	"github.com/go-lsp-bridge/lspbridge"
)

func main() {
	logger := slog.Default()
	accept := &websocket.AcceptOptions{
		OriginPatterns: []string{"editor.example.com"},
	}

	mux := http.NewServeMux()
	mux.HandleFunc("GET /lsp/{lang}", func(w http.ResponseWriter, r *http.Request) {
		// Tag the request so the per-user cap keys by identity, not token.
		r = r.WithContext(lspbridge.WithSubject(r.Context(), subjectOf(r)))
		lspbridge.HandleWS(w, r, r.PathValue("lang"), logger, accept)
	})

	// Let the client light up LSP only where a server is installed.
	mux.HandleFunc("GET /lsp-langs", func(w http.ResponseWriter, r *http.Request) {
		writeJSON(w, lspbridge.AvailableLanguages()) // ["go", "latex", ...]
	})

	http.ListenAndServe(":8080", mux)
}
```

`subjectOf(r)` is your own function returning the authenticated identity (user
id, session subject, …). If you don't tag the request, the per-user cap buckets
everything under `"anon"`.

## The client side

Any CodeMirror LSP client that speaks JSON-RPC over a WebSocket works unchanged,
because the WS payload is the bare JSON object. Point it at
`wss://your-host/lsp/go` (or whichever language) and it will send `initialize`,
`textDocument/didOpen`, `textDocument/completion`, and so on — the bridge relays
each to the language server and streams the responses (and server-initiated
notifications like `publishDiagnostics`) back.

## Error surfaces

The bridge fails **visibly** rather than silently dropping traffic:

| Condition | Response |
|-----------|----------|
| Global or per-user concurrency cap hit | HTTP `429 Too Many Requests` |
| Unknown `langKey` | HTTP `404 Not Found` |
| Server binary not on `$PATH` | HTTP `503 Service Unavailable` |
| stdin/stdout pipe or spawn failure | a JSON-RPC error frame, then WS close |

`EncodeError(id, code, msg)` is exported if you want to synthesise the same
JSON-RPC error shape yourself before handing a connection to the bridge.
