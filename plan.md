# pok.sh — System Design & Build Plan

## Context

`pok.sh` is an HTTP-tunneling service written in Go. It runs as a donation-funded service on a single Hetzner CX23 (2 vCPU, 8 GB RAM). The codebase aims to stay simple, minimal, contributor-friendly, and squeeze maximum concurrent tunnels out of one box.

**MVP user experience:**
- `pok 3000` — opens a tunnel mapping a random subdomain (e.g. `mossy-pelican-7421.pok.sh`) to local port 3000.
- `pok 3000 -d myapp` — claims `myapp.pok.sh`, anonymous, first-come-first-served.

A future paid tier (`pok 3000 -x <token>` backed by Turso) is out of scope for v1.

19 architectural decisions were locked through Q&A before designing (see "Locked Decisions" below). Build along the M1–M12 milestones in `todo.md`, each one producing something visibly working.

---

## Locked Decisions

| Area | Decision |
|---|---|
| Tunnel scope | HTTP(S) only — no raw TCP/UDP |
| Edge | Cloudflare in front of `*.pok.sh`, Full (strict) mode |
| Origin TLS for `*.pok.sh` | Cloudflare Origin Certificate (15-year wildcard) |
| Client transport | Custom multiplexed TCP over TLS using `hashicorp/yamux` |
| Control plane DNS | `connect.pok.sh` gray-cloud A record → Hetzner |
| Custom subdomain access | Anonymous FCFS, reserved+brand blocklists at compile time |
| Random subdomain shape | `adjective-noun-NNNN` |
| Subdomain rules | 3–63 chars, `[a-z0-9-]`, must start/end with letter |
| Streaming | Full support: WebSocket, SSE, chunked, long-lived |
| Body cap | 10 MB |
| Disconnect behavior | Subdomain released immediately, client exits (no auto-reconnect) |
| Per-IP client quota | 2 concurrent tunnels, 10 new tunnels/hour |
| Per-tunnel rate limit | 20 req/sec, burst 50 |
| Distribution | `curl pok.sh/install \| sh` + GitHub Releases binaries |
| Build pipeline | GitHub Actions + GoReleaser |
| License | MIT |
| Observability | JSON stdout logs + stdlib `expvar` counters on `127.0.0.1:9090/debug/vars` (no Prometheus dep, no off-box scraper in v1) |
| Landing page (`pok.sh/`) | Separate Cloudflare Pages site — NOT served by pok-server |
| IP banlist | NONE in-server; delegated to Cloudflare WAF + UFW |

---

## System Architecture

```
                              ┌──────────────────────┐
visitor (browser, curl) ────► │  Cloudflare WAF/TLS  │  *.pok.sh, orange cloud
                              └──────────┬───────────┘
                                         │ HTTPS (Origin Cert + mTLS)
                                         ▼
                              ┌──────────────────────┐
                              │  Hetzner CX23        │
                              │  pok-server          │
                              │   :443  public HTTPS │
                              │   :7000 control TLS  │  ← connect.pok.sh, gray cloud
                              │   :80   ACME only    │
                              │   :9090 /debug/vars  │ (localhost)
                              └──────────┬───────────┘
                                         │ TLS + yamux (one TCP per client)
                                         ▼
                              ┌──────────────────────┐
                              │ pok client (laptop)  │
                              │   forwards to        │
                              │   127.0.0.1:<port>   │
                              └──────────────────────┘
```

- One TLS+yamux session per connected client.
- Stream 0 = control plane (single, long-lived).
- Each public HTTP request opens a new server-initiated yamux stream → client → local app.

---

## Go Module Layout

Module: `github.com/gxanshu/pok.sh`

```
pok.sh/
  cmd/
    pok/                  # client CLI; main.go only
    pok-server/           # server daemon; main.go only
  internal/
    protocol/             # wire format, frame codec, version byte
      frame.go
      control.go
      version.go
    server/
      server.go           # top-level Server + Run()
      registry.go         # in-memory subdomain -> *Tunnel (RWMutex)
      public.go           # :443 HTTPS listener + Host dispatch
      control.go          # :7000 control listener + lifecycle
      tunnel.go           # *Tunnel struct (Session, meta)
      quota.go            # per-IP concurrent + sliding hourly window
      ratelimit.go        # per-tunnel token bucket
      subdomain.go        # validation + random name generator
      names.go            # //go:embed text lists
      metrics.go          # stdlib expvar counters/gauges
      tls.go              # cert loading helpers
    client/
      client.go           # dial, run loop
      forward.go          # stream -> 127.0.0.1 roundtrip
    buildinfo/
      buildinfo.go        # version/commit/date via ldflags -X
  assets/
    adjectives.txt
    nouns.txt
    reserved.txt          # www, admin, api, mail, ...
    brands.txt            # paypal, google, apple, ...
  deploy/
    pok-server.service
    Dockerfile
    install.sh
    smoke.sh
  .github/workflows/
    ci.yml
    release.yml
  docs/
    PLAN.md
    PROTOCOL.md
    OPERATIONS.md
    CONTRIBUTING.md
    todo.md
  .goreleaser.yaml
  go.mod
  LICENSE                 # MIT
  README.md
```

**Layout rationale:** precedent from `caddyserver/caddy` (split `cmd/`), `frpc` (one repo, two binaries), `chi` (flat `internal/`). `internal/` keeps the API surface unstable on purpose. No `pkg/` — nothing here is a stable library yet. `protocol` is the only package both client and server import; keep it free of `yamux`/`net/http` deps.

---

## Wire Protocol

### Layering

```
TCP → TLS → yamux session
  stream 0   control plane (client-initiated, long-lived)
  stream N   data plane (server-initiated, one per public request)
```

### Control plane

First byte client writes on stream 0 = **version byte (0x01)**. Server replies with version it accepted or closes the stream.

Subsequent messages framed as `[4-byte BE uint32 length][JSON bytes]`. **JSON, not CBOR** — control traffic is tiny, debuggable with `tcpdump` + `openssl s_client`, no extra toolchain.

```go
ControlRequest{
    Type: "open_tunnel",
    Subdomain: "myapp" | "",     // empty = random
    ClientVersion: "v1.2.3",
}

ControlResponse{
    Type: "tunnel_ready" | "error",
    Subdomain: "mossy-pelican-7421",
    PublicURL: "https://mossy-pelican-7421.pok.sh",
    ErrorCode: "subdomain_taken" | "invalid_subdomain" | "quota_exceeded" | "reserved" | "brand_blocked" | "",
    ErrorMessage: "...",
}
```

### Data plane

**Modified HTTP/1.1 over the stream — no custom envelope.** The server forwards the parsed `*http.Request` via `r.Write(stream)`; the client reads it with `http.ReadRequest(bufio.NewReader(stream))`, does an `http.Client.Do` to `127.0.0.1:<port>`, writes the response back with `resp.Write(stream)`.

Stdlib handles chunked / trailers / connection-close correctly. Zero hand-rolled HTTP parsing in the hot path.

Headers added/rewritten by server before forwarding:
- `X-Pok-Request-ID: <uuid>` (log correlation)
- `X-Forwarded-For: <visitor-ip>`
- `X-Forwarded-Proto: https`
- `X-Forwarded-Host: <subdomain>.pok.sh`
- `Host:` rewritten to `<subdomain>.pok.sh` (preserved — apps that vhost on Host header need this)

If `CF-Ray` is present on the inbound request, log it alongside `X-Pok-Request-ID`.

Body cap enforced with `http.MaxBytesReader(_, body, 10<<20)` on both sides. Overflow → 413 to visitor.

### Keepalive

Use yamux built-in: `KeepAliveInterval = 30s`, `ConnectionWriteTimeout = 10s`. **No application-level Ping/Pong** — yamux pings cover liveness.

### Upgrades (WebSocket / SSE)

- **WebSocket:** when client sees `resp.StatusCode == 101`, write 101 to stream, hijack the local conn from the roundtrip, spawn two `io.CopyBuffer` goroutines (stream↔local). Server side does the same: hijack the visitor conn via `http.Hijacker`, dual copy between visitor TCP and yamux stream. After the upgrade the stream is a raw byte pipe; server never decodes WS frames.
- **SSE:** no upgrade needed — `resp.Write(stream)` already streams `text/event-stream` bodies through.

**WebSocket Origin / CSWSH:** pok does **not** enforce same-origin. It is a transparent tunnel, not a security proxy. Document this in PROTOCOL.md.

---

## Connection Lifecycle

`pok 3000 -d myapp` end-to-end:

1. Client parses args. Local port = 3000. Desired sub = `myapp`.
2. `tls.Dial("tcp", "connect.pok.sh:7000", &tls.Config{ServerName:"connect.pok.sh"})`. Default system root CAs.
3. `yamux.Client(conn, yamux.DefaultConfig())`.
4. Open stream 0 → send version byte → read accepted version → send `ControlRequest`.
5. Server:
   - Accept TLS conn, wrap in `yamux.Server`.
   - Accept stream 0, version exchange.
   - Read `ControlRequest`. Validate subdomain (length, charset, start/end letter, reserved, brand).
   - Check per-IP quota (concurrent < 2, hourly < 10).
   - Insert into registry (RWMutex-guarded `map[string]*Tunnel`); fail with `subdomain_taken` if collision.
   - Write `ControlResponse{Type:"tunnel_ready"}`, log `tunnel_opened`.
6. Client prints `https://myapp.pok.sh`.
7. Client `session.Accept()` loop → per stream → `forward.HandleStream`.
8. Server begins routing `Host: myapp.pok.sh` → `tunnel.Session.OpenStream()` → forward request.

### Disconnect

Per-tunnel goroutine waits on `<-tunnel.Session.CloseChan()`. On fire:
1. `registry.Remove(sub)`
2. `quota.Release(remoteIP)`
3. `metrics.activeTunnels.Add(-1)`
4. Log `tunnel_closed` with duration.

In-flight visitor requests get EOF on their stream → server returns 502.

### Registry choice: `sync.RWMutex + map`, NOT `sync.Map`

Hot path is read-heavy (every public request does a lookup), writes happen on connect/disconnect. `sync.Map` is tuned for write-once read-many-disjoint, the wrong fit here. RWMutex with mostly-RLock is faster and simpler.

---

## HTTP Routing

Use `net/http` with one root handler that dispatches by `r.Host`:

```go
srv := &http.Server{
    Addr:    ":443",
    Handler: http.HandlerFunc(publicHandler),
    TLSConfig: originCertTLSConfig,
    ReadHeaderTimeout: 10 * time.Second,
    // NO ReadTimeout/WriteTimeout — would kill long-lived streams
    IdleTimeout: 120 * time.Second,
}
```

`publicHandler` extracts the subdomain with `strings.Cut`, looks up the tunnel, opens a stream, forwards. WebSocket = `http.Hijacker`.

**Why not custom HTTP parsing for v1:** stdlib is C10K-capable, handles every edge case correctly, gives us `Hijacker` for free. Header parsing isn't the bottleneck — the round-trip to the client is. v2 escape hatch: swap to `fasthttp` on the public listener only if profiling demands it.

---

## Hot Path Optimization (CX23 target: ~5K idle tunnels, ~500 concurrent active)

### Runtime tuning
- `GOMAXPROCS=2` (explicit in systemd unit — matches the CX23's 2 vCPU).
- `GOGC=200` — tunnel state is long-lived, fewer GCs helps.
- `GOMEMLIMIT=6GiB` — leaves 2 GB for OS + Cloudflare keepalive overhead.

### Buffer pools
- One `sync.Pool` of 32 KB `[]byte` for body copies in both `public.go` and `forward.go`.
- Use `io.CopyBuffer(dst, src, buf)` with pooled buffers, not `io.Copy`.

### Listeners
- Single `net/http` listener per port. **No SO_REUSEPORT in v1** — Go's netpoller spreads accept across the 2 Ps. Especially the right call at 2 cores; it only buys throughput on 4+ cores under contention.
- `LimitNOFILE=1048576` in systemd unit.

### Allocations
- `strings.Cut` not `strings.Split` for Host parsing.
- Pre-computed `[]byte(".pok.sh")` constant for trimming.
- Avoid `fmt.Sprintf` in dispatch path.

### Realistic ceiling math
~70 KB per idle tunnel (TLS conn + yamux session + struct + Accept goroutine). 5000 idle ≈ 350 MB. Active request streams ~50 KB each. 500 active ≈ 25 MB. **CPU is the binding constraint at 2 vCPU, not memory.** Honest expectations: ~5K idle tunnels with light traffic, ~300–500 concurrent active tunnels under sustained load. Document this in OPERATIONS.md and re-benchmark before announcing publicly.

### TLS
- `tls.Config{SessionTicketsDisabled: false, MinVersion: tls.VersionTLS12}` — session tickets cut handshake CPU when Cloudflare reconnects.

---

## TLS Strategy (Two Listeners, Two Certs)

| Port | Cert | Purpose |
|---|---|---|
| `:443` | Cloudflare Origin Certificate (wildcard, 15 yr) | Public HTTPS from Cloudflare only |
| `:7000` | Let's Encrypt cert for `connect.pok.sh` (via `autocert`) | Direct client control plane |
| `:80` | (ACME HTTP-01 handler only) | Cert renewal |

**Why not reuse the Origin Cert for `:7000`:** Origin Certs are signed by a Cloudflare-private CA. Clients connect directly, not through Cloudflare, so they'd need `InsecureSkipVerify` (loses MITM protection) or a pinned bundled cert (every rotation breaks installed clients). Let's Encrypt is in every OS root store, zero client config, auto-rotates.

**Recommendation: enable Cloudflare Authenticated Origin Pulls.** `tls.Config{ClientCAs: cloudflareCABundle, ClientAuth: tls.RequireAndVerifyClientCert}` on `:443`. Blocks anyone hitting the Hetzner IP directly with a guessed `Host` header. Free, recommended, no client-side changes.

Cert files on disk:
- `/etc/pok/origin.crt`, `/etc/pok/origin.key` — manual one-time from Cloudflare dashboard
- `/etc/pok/cloudflare-ca.pem` — for Authenticated Origin Pulls
- `/etc/pok/autocert-cache/` — managed by `golang.org/x/crypto/acme/autocert`

Client TLS config has zero customization in production: `tls.Config{ServerName: "connect.pok.sh"}`. Behind a `POK_DEV=1` env var, allow `InsecureSkipVerify=true` for local self-signed testing.

---

## Quotas & Rate Limiting

In `internal/server/quota.go`:

```go
type IPQuota struct {
    mu      sync.Mutex
    current map[string]int        // ip -> active count, cap 2
}

type HourlyCounter struct {
    mu     sync.Mutex
    events map[string][]time.Time  // ip -> recent open timestamps, cap 10/hour
}
```

`HourlyCounter.Record(ip)` appends `time.Now()`, prunes entries > 1h old in the same call, rejects if post-append length > 10. Slice + linear scan is fine — max 10 entries per IP.

Background goroutine every 5 min walks the maps and drops idle entries to keep memory bounded.

In `internal/server/ratelimit.go`:

```go
type Bucket struct {
    mu         sync.Mutex
    tokens     float64
    lastRefill time.Time
}
// per-Tunnel: burst=50, refill=20/sec
```

Public handler calls `tunnel.Bucket.Allow()` before opening a stream. On exhaustion: 429 with `Retry-After: 1`.

Metric: `expvar.NewMap("rejections")`, incremented with the reason as key (`ip_concurrent`, `ip_hourly`, `tunnel_rate`, `subdomain_taken`, `reserved`, `brand_blocked`, `invalid`).

---

## Observability

**Logs:** `log/slog` with `slog.NewJSONHandler(os.Stdout, nil)`. Events: `tunnel_opened`, `tunnel_closed`, `request_forwarded`, `quota_rejected`. Never log request bodies; log path but not query string.

**Counters:** stdlib `expvar` only. Bind to a separate `http.Server` listening on `127.0.0.1:9090`:

```go
mux := http.NewServeMux()
mux.Handle("/debug/vars", expvar.Handler())
metricsSrv := &http.Server{Addr: "127.0.0.1:9090", Handler: mux}
go metricsSrv.ListenAndServe()
```

Suggested vars (declare in `metrics.go` with `expvar.NewInt`/`expvar.NewMap`):

- `active_tunnels` (Int, current count)
- `tunnels_total` (Int, lifetime count)
- `http_requests_total` (Map keyed by status: `2xx`, `3xx`, `4xx`, `5xx`)
- `mux_streams_open` (Int)
- `bytes_in` / `bytes_out` (Int)
- `rejections` (Map keyed by reason)

Inspect on the box with `curl -s 127.0.0.1:9090/debug/vars | jq`. No off-box scraper, no Prometheus, no Grafana in v1. Sign up for Grafana Cloud free tier later if/when graphs become useful.

---

## Testing Strategy

- **Unit:** table-driven, `go test -race ./...`. Cover `protocol/frame`, `server/subdomain`, `server/quota`, `server/registry`.
- **Integration:** `server/integration_test.go` spins up server + client in-process on `:0` ports, hits a fake local HTTP server through the tunnel, asserts response. Subtests for WebSocket echo, 10 MB cap, rate-limit kick-in, quota rejection.
- **No live end-to-end against prod from CI** (costs money, flaky). Instead: `deploy/smoke.sh` run manually after each deploy + UptimeRobot pinging a permanent `health.pok.sh` tunnel running on the box itself (exempt loopback from the per-IP quota).

---

## Dependencies (deliberately few)

- `github.com/hashicorp/yamux` — connection multiplexing
- `golang.org/x/crypto/acme/autocert` — Let's Encrypt for `connect.pok.sh`
- Stdlib for everything else: `flag` for CLI, `log/slog` for JSON logs, `expvar` for counters, `net/http` for HTTP, `crypto/tls` for TLS. No chi, no cobra, no zap, no Prometheus client.

---

## Operations Runbook

### Cloudflare DNS / config
- `pok.sh` → Cloudflare Pages (landing page)
- `*.pok.sh` A → Hetzner IP, **orange cloud**
- `connect.pok.sh` A → Hetzner IP, **gray cloud (DNS only)**
- CAA record allowing `letsencrypt.org` (needed for autocert on `connect.pok.sh`)
- SSL/TLS mode: **Full (strict)**
- Authenticated Origin Pulls: **ON** for `*.pok.sh`
- WAF rate-limit rule on `*.pok.sh`: 100 req/min per IP (outer guard before our 20/sec inner limit)
- Cache: bypass for `*.pok.sh`

### Hetzner CX23 provisioning
1. Ubuntu 24.04, create `pok` user, `/etc/pok/`, `/var/log/pok/`, `/opt/pok/`
2. UFW: allow 22, 80, 443, 7000; deny rest
3. Place `origin.crt`, `origin.key`, `cloudflare-ca.pem` in `/etc/pok/`
4. `deploy/pok-server.service` with `LimitNOFILE=1048576`, `GOMAXPROCS=2`, `GOMEMLIMIT=6GiB`, `GOGC=200`, `AmbientCapabilities=CAP_NET_BIND_SERVICE`
5. `logrotate` daily, keep 7 days, compress

### Deploy
```
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o pok-server ./cmd/pok-server
scp pok-server hetzner:/opt/pok/pok-server.new
ssh hetzner 'mv /opt/pok/pok-server.new /opt/pok/pok-server && systemctl restart pok-server'
./deploy/smoke.sh
```

### Incident playbook
1. `systemctl status pok-server`
2. `curl -s 127.0.0.1:9090/debug/vars | jq '{active_tunnels, rejections, mux_streams_open}'`
3. Tail JSON logs, `jq` top offender IPs by `tunnel_opened` in last 10 min
4. Per-IP abuse → **Cloudflare WAF block** (never in-server)
5. Broad pattern → tighten Cloudflare rate-limit rule

`systemctl restart` will drop all tunnels (clients exit per locked decision). Deploy during low-traffic windows.

---

## Verification (how to know v1 is done)

End-to-end:
- `pok 3000` on your laptop, on a fresh laptop install via `curl https://pok.sh/install | sh`, opens `https://<random>.pok.sh` reachable from a phone on cellular.
- `pok 3000 -d myapp` succeeds; second laptop trying same `-d myapp` gets `subdomain_taken`.
- `websocat wss://myapp.pok.sh/echo` round-trips through a local WS server.
- `curl -X POST --data-binary @20MB.bin https://myapp.pok.sh/` returns 413.
- `ab -n 500 -c 50 https://myapp.pok.sh/` returns mix of 200 and 429 once burst is exhausted.
- 3rd concurrent `pok` from the same IP rejected with `quota_exceeded`.
- `curl https://connect.pok.sh:7000` → TLS handshake succeeds (cert valid in system store).
- `curl 127.0.0.1:9090/debug/vars` on the box returns JSON with `active_tunnels`, `tunnels_total`, `rejections{...}`, etc.
- `pok --version` prints commit SHA + date.
- All tests pass with `go test -race ./...` in CI.

Acceptance numbers to verify in load test before announcing publicly (calibrated for 2 vCPU / 8 GB):
- 1000 idle tunnels held simultaneously with < 1 GB RSS.
- 100 concurrent active tunnels each doing 5 req/sec sustained for 5 minutes, p99 < 200ms added latency vs direct, no goroutine leaks.

---

## Out of Scope for v1 (do not build)

- Turso / paid subdomains / `-x token` flag
- Admin dashboard
- Runtime banlist module (`banlist.go`, `banned-*.txt`, `POK_BANLIST_DIR`)
- Auto-reconnect on the client
- TCP/UDP tunneling
- HTTP/3 / QUIC
- Per-tunnel custom rate-limit flag (`--rate`)
- OAuth/identity
- Hot-reload of brand/reserved lists (rebuild + redeploy is fine)
