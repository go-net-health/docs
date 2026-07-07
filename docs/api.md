# Usage & API

The public API lives at the module root (`github.com/go-net-health/health`). It
is small, value-typed, and free of global state.

!!! success "Status: implemented"
    The probe runner is built and importable as
    `github.com/go-net-health/health`.

## Install

```sh
go get github.com/go-net-health/health
```

## A one-off attempt

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/go-net-health/health"
)

func main() {
	probe, err := health.New(&health.Spec{
		Type:     health.TypeHTTP,
		HTTPPort: 8080,
		HTTPPath: "/healthz",
		Timeout:  time.Second,
	})
	if err != nil {
		panic(err)
	}
	pass, latency, err := probe.Attempt(context.Background(), "10.0.0.7")
	fmt.Println(pass, latency, err)
}
```

## Driving it on a schedule

```go
spec := &health.Spec{
	Type:             health.TypeTCP,
	TCPPort:          22,
	Timeout:          time.Second,
	Period:           2 * time.Second,
	InitialDelay:     time.Second,
	FailureThreshold: 3,
	SuccessThreshold: 1,
}
probe, _ := health.New(spec)
r := health.NewRunnerFromSpec(probe, "10.0.0.7", spec)

ctx, cancel := context.WithCancel(context.Background())
defer cancel()
go r.Run(ctx)

for res := range r.Results() {
	fmt.Printf("%s  pass=%v  latency=%s\n", res.Status, res.Pass, res.Latency)
	if res.Status == health.StatusHealthy {
		r.Stop()
	}
}
```

## Shape

### `Spec`

The dependency-free configuration for a probe and its runner. Zero values fill in
sensible defaults, so the smallest useful specs are
`&Spec{Type: TypeHTTP, HTTPPort: 8080}` and `&Spec{Type: TypeTCP, TCPPort: 22}`.

```go
type Spec struct {
	Type             Type
	HTTPPath         string        // default "/"
	HTTPPort         int           // required for HTTP
	HTTPMethod       string        // default "GET"
	HTTPStatusOK     []int         // empty = any 2xx
	TCPPort          int           // required for TCP
	InitialDelay     time.Duration // wait before the first probe
	Period           time.Duration // between probes; default 1s
	Timeout          time.Duration // per-probe deadline; default 1s
	FailureThreshold int           // consecutive failures = unhealthy; default 3
	SuccessThreshold int           // consecutive successes = healthy; default 1
}
```

### `Type` and `Status`

```go
type Type int
const (
	TypeNone Type = iota // New returns (nil, nil)
	TypeHTTP
	TypeTCP
	TypeExec             // reserved, not yet supported
)

type Status int
const (
	StatusUnknown Status = iota
	StatusHealthy
	StatusUnhealthy
)
```

Both implement `fmt.Stringer` (`"http"`, `"healthy"`, …).

### `New` and `Probe`

```go
// New translates a Spec into a runnable Probe. It returns (nil, nil) for
// TypeNone, and an error for TypeExec or an invalid Spec (missing port).
func New(s *Spec) (Probe, error)

type Probe interface {
	Attempt(ctx context.Context, addr string) (pass bool, latency time.Duration, err error)
	Kind() Type
}
```

### `Result`

```go
type Result struct {
	When    time.Time     // attempt time (injectable clock)
	Status  Status        // rolling status after this attempt
	Pass    bool          // this attempt's raw verdict
	Latency time.Duration // wall-clock duration, even on failure
	Err     error         // nil on pass
}
```

### `Runner`

```go
type RunnerOptions struct {
	Period           time.Duration
	InitialDelay     time.Duration
	FailureThreshold int
	SuccessThreshold int
	Now              func() time.Time // optional; defaults to time.Now
}

func NewRunner(probe Probe, addr string, opts RunnerOptions) *Runner
func NewRunnerFromSpec(probe Probe, addr string, s *Spec) *Runner

func (r *Runner) Run(ctx context.Context) // blocks until ctx done / Stop()
func (r *Runner) Results() <-chan Result  // closed when Run returns
func (r *Runner) Status() Status          // rolling verdict; goroutine-safe
func (r *Runner) Stop()                   // idempotent
```

### `IsTimeout`

```go
// IsTimeout reports whether err is a probe timeout ("never reached the target")
// rather than the target actively refusing.
func IsTimeout(err error) bool
```

## Defaults at a glance

| field | zero-value default |
| --- | --- |
| `HTTPPath` | `/` |
| `HTTPMethod` | `GET` |
| `HTTPStatusOK` | any `2xx` |
| `Timeout` | `1s` |
| `Period` | `1s` |
| `FailureThreshold` | `3` |
| `SuccessThreshold` | `1` |
| `RunnerOptions.Now` | `time.Now` |
