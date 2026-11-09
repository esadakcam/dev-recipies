# Go recipe

Read project instructions and `go.mod`; use features supported by the declared Go version.

## Structure and APIs

- Organize by capability; separate executable wiring from reusable behavior. Use `cmd/<command>/` and `internal/` where useful.
- Share common behavior; isolate backend details. Document exported APIs, ownership, blocking, and errors; preserve initialisms (`HTTP`, `ID`, `URL`). [Go review guidance](https://go.dev/wiki/CodeReviewComments).

## Interfaces and structs

- Prefer concrete structs and explicit ownership; define small consumer interfaces only for demonstrated needs. Avoid speculative abstractions.
- Inject dependencies; use constructors for required initialization/validation, returning concrete types and construction errors.
- Favor useful zero values; keep invariant-bearing fields private.
- Use pointer receivers for mutation/non-copyable state/large values; value receivers for small value types. Stay consistent.
- Compose through fields; embed only for intentional method promotion.
- A typed nil inside an interface is non-nil; return explicit `nil` for absence.

[Interfaces/receivers](https://go.dev/wiki/CodeReviewComments), [zero values/composition](https://go.dev/doc/effective_go), [nil interfaces](https://go.dev/doc/faq#nil_error).

## Dependencies

Prefer the standard library when sufficient. Commit `go.mod` and generated `go.sum`; review dependency changes, avoid unrelated upgrades, and pin development tools. [Modules](https://go.dev/doc/modules/managing-dependencies).

## Configuration and validation

Use typed configuration and explicit tags. Centralize documented defaults; validate fields/ranges before use and test invalid inputs. Keep examples current and choose decoding strictness without silently breaking compatibility.

## Errors and logging

Add useful context with `%w` when the cause belongs to the API contract. Inspect with `errors.Is`/`errors.As`, not strings. Use `(bool, error)` when a negative result differs from evaluation failure. [Errors](https://go.dev/blog/go1.13-errors).

Normally log errors once at the handling boundary, using consistent severity and structured attributes. Exclude credentials, tokens, and sensitive payloads. Follow the project's logger; default to `log/slog` for new applications. [Structured logging](https://pkg.go.dev/log/slog).

## Context and cleanup

- Pass context first through blocking calls. Set operation deadlines, call cancellation functions, and make waits/retries/communication cancellable; preserve cancellation causes.
- Never pass nil contexts; normally pass them explicitly rather than storing them in structs. Use context values for request-scoped metadata, not configuration or dependency injection.
- Clean up successful acquisitions and partial construction failures. Scope `defer` to resource lifetimes, including loop iterations. Post-cancellation cleanup needs a live bounded context and must finish before exit.

Create the service's parent context at entry and pass it to workers. Import `context`, `os`, `os/signal`, and `syscall`; `stop` releases signal resources:

```go
ctx, stop := signal.NotifyContext(
	context.Background(),
	os.Interrupt,
	syscall.SIGTERM,
)
defer stop()
```

[Context](https://pkg.go.dev/context), [signals](https://pkg.go.dev/os/signal#NotifyContext), [defer](https://go.dev/doc/effective_go#defer).

## Concurrency

> Do not communicate by sharing memory; instead, share memory by communicating.

- Prefer synchronous APIs. Give goroutines an owner, stop condition, and completion path; cancel and join them before releasing resources.
- Sending a pointer, slice, or map through a channel shares the referenced data. Choose explicitly: the sender stops reading and writing that data after sending; each goroutine gets an independent copy, including mutable nested data; all goroutines treat the shared data as read-only; or all reads and writes use the same mutex.
- Bound workers with a pool or `errgroup.SetLimit`; configure limits before starting work.
- Join tasks with `sync.WaitGroup`: `Add` before launch, defer `Done`. On Go 1.25+, `WaitGroup.Go` simplifies this; callbacks must not panic. For errors and sibling cancellation, use `errgroup.WithContext`, propagate its context, and return `Wait()`'s error.
- Protect shared mutable state consistently with mutexes; use atomics for independent values. Never copy used synchronization primitives.
- Let the sender/coordinator close channels after all sends finish. Make blocking sends/receives cancellable; buffering does not ensure shutdown.
- Test errors, cancellation, early consumer exit, and completion with race detection; sleeps do not synchronize.

[Effective Go](https://go.dev/doc/effective_go#sharing), [synchronous APIs](https://go.dev/wiki/CodeReviewComments#synchronous-functions), [goroutine lifetimes](https://go.dev/wiki/CodeReviewComments#goroutine-lifetimes), [sync](https://pkg.go.dev/sync), [errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup), [pipelines](https://go.dev/blog/pipelines).

## HTTP, when used

- Reuse clients/transports, use `NewRequestWithContext`, bound request lifetimes, and close response bodies on every path. Long calls may use request-specific deadlines.
- Use an explicit server/mux, injected handlers, and typed payloads. Validate inputs, propagate `r.Context()`, and set headers before writing consistent status/error responses.
- Configure header/idle/body/write limits; use `MaxBytesReader` and bound buffered response reads. Treat `ErrServerClosed` as normal, propagate listener failures, and await bounded `Shutdown`. Manage hijacked connections separately. [net/http](https://pkg.go.dev/net/http), [LimitReader](https://pkg.go.dev/io#LimitReader).

## Testing

- Colocate `_test.go` files; use standard `testing`, named subtests/tables, and test-local doubles. Use same-package tests for private access.
- Assert observable success, invalid input, errors, defaults, and boundaries; cover cancellation, concurrency, and recovery where relevant. Compare error semantics unless exact diagnostics are contractual.
- Use `t.Helper`, `TempDir`, `Cleanup`, and `Setenv`. Isolate parallel tests; environment changes forbid parallel tests/ancestors. Synchronize readiness/completion with bounded waits, clean up workers/resources, report intentional skips, and fail required setup errors.
- Use `httptest` servers for clients and requests/recorders for handlers; populate direct-call path values. Check requests, responses, and failures.
- Race-test realistic concurrency; only executed paths are checked. Fuzz pure parsers with deterministic invariants, seeds, bounded runs, and saved regressions. Benchmark concrete performance questions, report relevant allocations, and use `b.Loop()` when supported, excluding setup. [Testing](https://pkg.go.dev/testing), [error assertions](https://go.dev/wiki/TestComments#test-error-semantics), [race detection](https://go.dev/doc/articles/race_detector), [fuzzing](https://go.dev/doc/security/fuzz/).

## Building the application

Provide these Make targets. Set `APP`/`MAIN`; preserve recipe tabs. Builds default to the host, accept platform overrides, and write `bin/<os>-<arch>/`:

```makefile
APP ?= app
MAIN ?= ./cmd/app
GOOS ?= $(shell go env GOOS)
GOARCH ?= $(shell go env GOARCH)
EXT := $(if $(filter windows,$(GOOS)),.exe,)
OUT := bin/$(GOOS)-$(GOARCH)/$(APP)$(EXT)

.DEFAULT_GOAL := build
.PHONY: fmt vet run build

fmt:
	go fmt ./...

vet:
	go vet ./...

run:
	go run "$(MAIN)"

build:
	mkdir -p "$(dir $(OUT))"
	GOOS="$(GOOS)" GOARCH="$(GOARCH)" go build -o "$(OUT)" "$(MAIN)"
```

```sh
make fmt vet
make run
make build                              # Host platform
make build GOOS=linux GOARCH=amd64
make build GOOS=darwin GOARCH=arm64      # macOS on Apple Silicon
make build GOOS=linux GOARCH=arm GOARM=7 # 32-bit ARMv7
```

`arm` is 32-bit; `arm64` is distinct. Check supported pairs with `go tool dist list`. GNU Make forwards overrides such as `GOARM`; cgo cross-builds need a target C toolchain. [Go targets](https://go.dev/doc/install/source#environment), [Make overrides](https://www.gnu.org/software/make/manual/html_node/Overriding.html).

Use a Go Alpine builder and Distroless Debian nonroot runtime. This BuildKit template disables cgo; cgo applications need a compatible builder/runtime pairing:

```dockerfile
# syntax=docker/dockerfile:1
ARG GO_VERSION
FROM --platform=$BUILDPLATFORM golang:${GO_VERSION}-alpine AS builder
ARG TARGETOS
ARG TARGETARCH
ARG TARGETVARIANT
ARG MAIN=./cmd/app
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS="${TARGETOS}" GOARCH="${TARGETARCH}" \
    GOARM="${TARGETVARIANT#v}" go build -o /out/app "${MAIN}"

FROM gcr.io/distroless/static-debian13:nonroot
COPY --from=builder /out/app /app
USER nonroot:nonroot
ENTRYPOINT ["/app"]
```

Replace placeholders with a supported Go image version compatible with `go.mod` and the actual main package:

```sh
docker build --platform=linux/amd64 \
  --build-arg GO_VERSION='<go-version>' \
  --build-arg MAIN='./<main-package>' -t app:local .
```

Docker supplies platform arguments; select `linux/arm64` or `linux/arm/v7` as needed. Build Darwin binaries with Make. Exclude `bin/`, VCS data, and credentials via `.dockerignore`. Keep only runtime artifacts; retain exec-form entrypoint and verify nonroot startup/shutdown. Distroless has no shell. [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/), [cross-compilation](https://docs.docker.com/build/building/multi-platform/#cross-compiling-a-go-application), [Go image](https://hub.docker.com/_/golang), [Distroless](https://github.com/GoogleContainerTools/distroless).

## Verification

Run from the module root with dependencies and required fixtures available. Race tests need a supported platform, cgo, and a C compiler. `-run` still executes `TestMain`.

| Purpose          | Command                                                                                                             |
| ---------------- | ------------------------------------------------------------------------------------------------------------------- |
| Format / inspect | `go fmt ./...`; require empty `gofmt -s -l .` output (differences alone do not fail).                               |
| Vet / build      | `go vet ./...`; `go build ./...`                                                                                    |
| Dependencies     | `go mod download`; `go mod verify`; `go mod tidy` for deliberate edits.                                             |
| Tests            | `go test ./...`; add `-count=1` for fresh execution.                                                                |
| Race + coverage  | `go test -race -coverprofile=coverage.out ./...`                                                                    |
| Reports          | `go tool cover -func=coverage.out`; `go tool cover -html=coverage.out -o coverage.html` after generating a profile. |
| Fuzz             | `go test './<package>' -run='^$' -fuzz='^Fuzz<Parser>$' -fuzztime=30s`                                              |
| Benchmark        | `go test './<package>' -run='^$' -bench='^Benchmark<Operation>$' -benchmem -count=5`                                |
| Security         | `govulncheck ./...` with a compatible installed tool and vulnerability-data access.                                 |

Replace `<...>` with existing packages/targets. Downloads need network access or cached dependencies. Follow project-specific integration prerequisites. [Go commands](https://pkg.go.dev/cmd/go).
