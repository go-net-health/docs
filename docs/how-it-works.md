# How it works

Two pieces: a **`Probe`** (one stateless attempt) and a **`Runner`** (the
schedule + the rolling verdict).

## Probes

A `Probe` runs a single attempt of a configured check against a target address
and returns `(pass, latency, err)`. It is side-effect-free apart from the one
network call.

```go
type Probe interface {
	Attempt(ctx context.Context, addr string) (pass bool, latency time.Duration, err error)
	Kind() Type
}
```

`New(*Spec)` builds one:

- **`TypeTCP`** — dials `addr:TCPPort` with the per-probe timeout. A successful
  connect is a pass; the connection is closed immediately.
- **`TypeHTTP`** — issues `HTTPMethod` `http://addr:HTTPPort/HTTPPath` and checks
  the status code.
- **`TypeNone`** — `New` returns `(nil, nil)` so "no probe configured" needs no
  type assertion.
- **`TypeExec`** — reserved; `New` returns an error (see [Roadmap](roadmap.md)).

### Health-check-correct HTTP semantics

The HTTP probe behaves the way a real liveness/readiness check must:

- **Any `2xx` passes by default.** Set `HTTPStatusOK` to an explicit list to
  accept a different set (e.g. `[]int{503}` to treat a draining service as
  healthy); anything outside the set fails with a descriptive error.
- **Redirects are not followed.** A `3xx` fails the default `2xx` check — the
  probe verifies *this* endpoint is alive, not wherever it would forward to.
- **Latency is always reported**, even on failure: a timeout surfaces with
  `Latency ≈ Timeout`.
- **A fresh, keep-alive-disabled transport per probe**, so connection pools do
  not bleed between targets watched concurrently and frequent probes do not pin
  idle file descriptors open.

## The Runner and the rolling status

A `Runner` drives one probe against one target:

```go
r := health.NewRunnerFromSpec(probe, "10.0.0.7", spec)
go r.Run(ctx)          // blocks until ctx is cancelled or r.Stop()
for res := range r.Results() {
	// res.Status, res.Pass, res.Latency, res.Err, res.When
}
```

- **Initial delay** is honoured exactly once, before the first attempt.
- The first attempt then fires **immediately**, so the first `Result` lands
  without waiting a full period; subsequent attempts fire every `Period`.
- Each attempt rolls into the `Status`:

    | transition | condition |
    | --- | --- |
    | → `Healthy` | `SuccessThreshold` consecutive passes |
    | → `Unhealthy` | `FailureThreshold` consecutive failures |
    | `Unknown` | before any threshold is met |

  A single pass resets the failure streak and vice-versa.

- Every attempt emits a `Result` on a **buffered** channel (32 slots). Sends are
  **non-blocking**: a wedged consumer causes drops rather than freezing the
  probe — a deliberate safety valve. `Results()` is closed when `Run` returns.
- `Status()` returns the current rolling verdict and is safe to call from any
  goroutine.
- `Stop()` is idempotent and drains the goroutine without losing the final
  `Result`.

## Timeout vs refused

Both are failures, but they often warrant different handling — a grace period for
a target that is merely slow to come up, none for one that is actively refusing:

```go
if health.IsTimeout(res.Err) {
	// never reached the target (deadline / net timeout)
} else {
	// reached it; it said no (e.g. connection refused, bad status)
}
```

## Testability

The `Runner` takes an injectable clock via `RunnerOptions.Now`, so tests can
assert on `Result.When` with no wall-clock dependency. The whole package is
exercised at 100% coverage against loopback `httptest` / `net.Listen` servers and
a deterministic fake probe, under the race detector, on all six 64-bit Go arches.
