# Concurrency & limits

Every WebSocket connection spawns its **own** language-server subprocess, killed
when the socket closes. Left unbounded, a malicious or buggy client could
fork-bomb the host by reconnecting in a loop. The bridge defends with two
independent ceilings.

## Two layers

| Layer | Ceiling | Purpose |
|-------|---------|---------|
| Global | 16 concurrent spawns | The host as a whole can't be saturated. |
| Per-subject | 4 concurrent spawns | One editor can run ~one server per open language, but a single identity can't starve everyone else. |

Both are checked **before** the WebSocket is accepted. If either is exceeded the
request is rejected with HTTP `429 Too Many Requests`, so the client sees a clean
failure instead of a half-open socket.

## Keying the per-user cap

The per-subject counter buckets by whatever you stash with `WithSubject`:

```go
r = r.WithContext(lspbridge.WithSubject(r.Context(), userID))
lspbridge.HandleWS(w, r, lang, logger, accept)
```

If you don't tag the request, the subject falls back to `"anon"` and all
untagged connections share one bucket.

!!! tip "Bucket by identity, not by token"
    Keying on the authenticated **subject** rather than the raw bearer token or
    cookie matters: during token rotation (or on multiple devices) the same user
    can legitimately hold two valid tokens at once. Keying on the token would
    silently double their effective cap; keying on the subject does not.

## Lifecycle

Once past the caps and a successful upgrade, the bridge:

1. Resolves and starts the server subprocess (`exec.CommandContext` bound to the
   request context).
2. Runs two goroutines — one draining server stdout to the WS, one draining WS
   frames to server stdin — until either side closes or errors.
3. On teardown, cancels the context, kills and reaps the subprocess, and releases
   both concurrency slots.

Server `stderr` is forwarded to the process's own `stderr` so language-server
diagnostics land in your host logs.
