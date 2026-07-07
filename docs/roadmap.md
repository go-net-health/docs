# Roadmap

## Done

- **TCP probe** — connect check with a per-probe timeout.
- **HTTP probe** — configurable path, method and timeout; any-`2xx`-by-default or
  an explicit accepted status set; redirects deliberately **not** followed;
  latency reported even on failure; a fresh keep-alive-disabled transport per
  probe.
- **`Runner`** — one-shot initial delay, immediate first attempt, fixed period,
  success/failure thresholds rolling into `Healthy` / `Unhealthy` / `Unknown`, a
  buffered non-blocking `Result` stream, a goroutine-safe `Status()`, and an
  idempotent `Stop()`.
- **`IsTimeout`** — separates "never reached the target" from "the target
  refused".
- **Injectable clock** for deterministic tests.
- **100% test coverage** across the six 64-bit Go targets (amd64, arm64,
  riscv64, loong64, ppc64le, s390x), `-race`, `gofmt` + `go vet` clean.

## Reserved / out of scope

- **`TypeExec` probe.** Running a command *inside* the target (rather than a
  network check *against* it) needs an in-target agent channel this stateless
  network runner has no access to. `New` returns an explicit error for
  `TypeExec` today so a misconfiguration surfaces immediately, rather than
  silently doing nothing. It stays out of scope until such a channel exists.

- **Acting on the verdict.** Restarting, rerouting, alerting — deliberately *not*
  here. The library reports `Healthy` / `Unhealthy`; the decision of what to do
  is the caller's. This split is what keeps the runner small, dependency-free and
  reusable.

## Possible future work

- Additional probe kinds that remain *pure network checks* (e.g. gRPC health,
  TLS-handshake-only) — anything expressible as a stateless attempt with no
  in-target agent.
- Optional structured logging / metrics hooks, kept behind an interface so the
  core stays dependency-free.
