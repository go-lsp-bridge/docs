# Roadmap

The core bridge is **complete and released**. The items below are optional
enhancements; none are required for the transport bridge, which stays
dependency-light (one module: `github.com/coder/websocket`).

## Shipped

- [x] `HandleWS` — bidirectional JSON-RPC relay with correct `Content-Length`
      framing in both directions.
- [x] `DefaultServers()` registry with neutral `LSPBRIDGE_*` env overrides;
      caller-supplied / extensible `Servers` map.
- [x] Global + per-subject concurrency caps (`WithSubject`).
- [x] `AvailableLanguages()` progressive-enablement probe.
- [x] Visible failure surfaces (`429` / `404` / `503` / JSON-RPC error frames);
      `EncodeError` helper.
- [x] Hermetic `cmd/fake-lsp` round-trip test at 100% coverage; CI on six arches.

## Planned

- [ ] **Warm-subprocess pooling** — optionally keep a small pool of started
      servers per language to amortise cold-start latency (gopls/rust-analyzer
      init can be slow), behind an opt-in so the default stays one-process-per
      -connection.
- [ ] **Metrics hooks** — counters for spawns, cap rejections and frame
      throughput, exposed through a pluggable interface (no hard dependency on a
      metrics library).
- [ ] **Configurable caps** — expose the global / per-user ceilings as options
      rather than compile-time constants.
