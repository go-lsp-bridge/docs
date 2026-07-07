# Server registry & configuration

A language key resolves to a launcher through the **`Servers`** registry — a
plain `map[string]LanguageServer`:

```go
type LanguageServer struct {
	Lang        string   // canonical key ("latex", "go", …)
	Binary      string   // default executable name
	Args        []string // extra CLI arguments
	EnvOverride string   // env var that, if set to an existing file, wins
}
```

`Servers` is initialised to `DefaultServers()`. You can mutate it, extend it, or
replace it wholesale before you start serving.

## The defaults

`DefaultServers()` returns a fresh map with these entries:

| Key | Binary | Args | Env override |
|-----|--------|------|--------------|
| `latex` | `texlab` | | `LSPBRIDGE_TEXLAB` |
| `go` | `gopls` | `serve` | `LSPBRIDGE_GOPLS` |
| `python` | `pyright-langserver` | `--stdio` | `LSPBRIDGE_PYRIGHT` |
| `typescript` | `typescript-language-server` | `--stdio` | `LSPBRIDGE_TS` |
| `javascript` | `typescript-language-server` | `--stdio` | `LSPBRIDGE_TS` |
| `rust` | `rust-analyzer` | | `LSPBRIDGE_RUSTANALYZER` |

The env-override keys use a neutral `LSPBRIDGE_*` prefix — nothing about the
catalogue is tied to a particular host application.

!!! note "The `fake` entry"
    `DefaultServers()` also includes a `fake` launcher wired to `LSPBRIDGE_FAKE`.
    It resolves only when that variable points at a built `cmd/fake-lsp` binary,
    and exists so the integration test can run the full round-trip without any
    real language server installed. It never resolves in production.

## Binary resolution

For a given entry, `resolveBinary` picks the executable in this order:

1. If `EnvOverride` is set **and** the variable names a path that exists, use it.
2. Otherwise `exec.LookPath(Binary)` on `$PATH`.
3. If neither resolves, the connection is rejected with `503` and the
   language is omitted from `AvailableLanguages()`.

So operators can repoint a server without rebuilding the image:

```sh
LSPBRIDGE_GOPLS=/opt/toolchains/go/bin/gopls \
LSPBRIDGE_TEXLAB=/usr/local/bin/texlab \
  ./your-server
```

## Adding a language

Extend the defaults in place:

```go
lspbridge.Servers["zig"] = lspbridge.LanguageServer{
	Lang: "zig", Binary: "zls", EnvOverride: "LSPBRIDGE_ZLS",
}
```

…or supply a curated registry (e.g. to expose only the servers you ship):

```go
lspbridge.Servers = map[string]lspbridge.LanguageServer{
	"go":   {Lang: "go", Binary: "gopls", Args: []string{"serve"}, EnvOverride: "LSPBRIDGE_GOPLS"},
	"rust": {Lang: "rust", Binary: "rust-analyzer", EnvOverride: "LSPBRIDGE_RUSTANALYZER"},
}
```

## Advertising availability

`AvailableLanguages()` returns the sorted set of keys whose binary actually
resolves on the host. Serve it to your client so the editor enables LSP only
where it will work, and falls back to a simpler autocomplete elsewhere.
