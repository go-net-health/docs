# go-net-health documentation

**A pure-Go active health-probe runner** — stateless network checks, HTTP or TCP,
folded into a rolling `healthy` / `unhealthy` / `unknown` verdict, built with
**zero cgo**.

`go-net-health/health` points a `Probe` at a single target address, runs it on a
schedule with a `Runner`, and derives a rolling `Status` from consecutive
passes/failures. The module path is `github.com/go-net-health/health`.

The module is **standalone and reusable**: any Go program — a supervisor, a
load-balancer, a respawn loop — can import it and act on the verdict itself. It
has no opinion on what `Healthy` vs `Unhealthy` should *do*.

!!! success "Status: probe runner complete"
    `TypeHTTP` (configurable path/method/timeout, any-`2xx`-or-explicit status
    set, **no redirect following**) and `TypeTCP` (connect check) probes; a
    periodic `Runner` with a one-shot initial delay, success/failure thresholds
    and a buffered `Result` stream; and `IsTimeout` to separate "unreachable"
    from "refused". `TypeExec` is reserved. 100% coverage, `gofmt` + `go vet`
    clean, CI green across the six 64-bit Go targets.

## Quick taste

```go
spec := &health.Spec{Type: health.TypeHTTP, HTTPPort: 8080, HTTPPath: "/healthz"}
probe, _ := health.New(spec)

// one-off attempt
pass, latency, err := probe.Attempt(context.Background(), "10.0.0.7")

// or drive it on a schedule
r := health.NewRunnerFromSpec(probe, "10.0.0.7", spec)
go r.Run(ctx)
for res := range r.Results() {
    fmt.Println(res.Status, res.Pass, res.Latency)
}
```

## What it is — and isn't

Each probe attempt is **stateless**: one network call, a pass/fail plus latency,
no memory. The **`Runner`** is the only stateful piece — it stacks consecutive
attempts into a rolling `Status` per explicit thresholds. Beyond that one network
call the library has **no side effects**: it does not restart anything, route
traffic, or page anyone.

> **It probes and reports; the caller decides.**

## Repositories

| Repo | What it is |
| --- | --- |
| [`health`](https://github.com/go-net-health/health) | the probe runner — `Probe` (HTTP/TCP), `Runner`, `Status`, `Result`, `IsTimeout` |
| [`docs`](https://github.com/go-net-health/docs) | this documentation site (MkDocs Material, versioned with mike) |
| [`go-net-health.github.io`](https://github.com/go-net-health/go-net-health.github.io) | the organization landing page (Hugo) |
| [`brand`](https://github.com/go-net-health/brand) | logo and brand assets |

## Principles

- **Pure Go, `CGO_ENABLED=0`** — depends only on the standard library; trivial
  cross-compilation, a single static binary, no C toolchain.
- **Stateless probes, rolling verdict** — an attempt is a fact; the `Runner`
  turns a window of facts into `Healthy` / `Unhealthy`.
- **It probes; you decide** — no respawn, no routing, no side effects beyond the
  one network call.
- **100% test coverage** is the target, enforced as a CI gate.

## Where to go next

- [How it works](how-it-works.md) — probes vs the runner, the rolling status
  state machine, and health-check-correct HTTP semantics.
- [Usage & API](api.md) — `Spec`, `New`, `Probe`, `Runner`, `Result`,
  `IsTimeout`.
- [Roadmap](roadmap.md) — what is done and what is out of scope by design.

Source lives at
[github.com/go-net-health/health](https://github.com/go-net-health/health).
