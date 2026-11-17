---
owner: group:default/dto
last_reviewed: 2026-11-17
---

# Run locally

The bridge has no local mode. At start-up it logs in to a real 3CX server and keeps retrying for up to two minutes; if that fails, it exits. So day-to-day local work is **build and unit tests**. Running the full daemon needs a reachable 3CX and a Zammad you are allowed to write to.

!!! danger "Never run a local bridge against production"
    Do not start a local bridge with the production `config.yaml`. A second bridge polling the same 3CX sends every call event to Zammad twice and, with auto-create on, opens duplicate tickets and users for real customers.

## Prerequisites

- Go **1.25** or newer (`go.mod` says `go 1.25`; it also declares `goimports` as a Go tool).
- Git.
- Optional: Docker, if you want to build the container image the same way CI does.
- Only to run the daemon end to end: a test 3CX (v20+ needs an API client ID and secret with the extensions you want to watch added to its permissions), and a test Zammad with an API token and the CTI integration turned on. TODO(owner): is there a test 3CX and a test Zammad that engineers may use? None is recorded.

## Steps

1. Clone the repo and enter it:
   ```bash
   git clone https://github.com/AGM-One-Vision/3cx-zammad-bridge.git
   cd 3cx-zammad-bridge
   ```
2. Build:
   ```bash
   go build ./...
   ```
   `make build` also works on Linux/macOS, but it runs a bare `goimports -w .` first, so install that binary once: `go install golang.org/x/tools/cmd/goimports@latest` (and have `$(go env GOPATH)/bin` on `PATH`). It writes `./dist/zammadbridge`.
3. Run the unit tests (the same checks CI runs; CI installs `goimports` and runs it directly):
   ```bash
   go test ./...
   go test -race ./...    # needs cgo (a C compiler); CI runs it on ubuntu-latest
   go tool goimports -d -e .   # formatting check; prints nothing when clean
   ```
   Expected result: `ok  github.com/qmexnetworks/3cx-zammad-bridge`.
4. Only if you have a test 3CX and a test Zammad: copy `config.yaml.dist` to `config.yaml` in the repo root (it is git-ignored), fill in the test values, and run:
   ```bash
   go run ./cmd -f plain -v
   ```
   The admin page starts only when `Admin.enabled` is `true` and both `Admin.user` and `Admin.pass` are set. It then listens on `Admin.listen` (default `:8090`), for example <http://localhost:8090/healthz>.

## Test data

- The unit tests need no external systems. They use fake 3CX and Zammad servers built with Go's `httptest` (`zammad_test.go`, `admin_test.go`) and pure-function tests for the filters and the de-duplication window (`autocreate_test.go`).
- There is no seed data, fixture database or shared test 3CX account in the repo.

## Common problems

| Symptom | Cause and fix |
|---|---|
| `Unable to load configuration` and the process exits | No readable `config.yaml` was found. The bridge looks in `config.yaml` (working directory), then `/etc/3cx-zammad-bridge/config.yaml`, then `//3cx-zammad-bridge/config.yaml`, or only in the file given with `-c`. The upstream README also lists `/opt/3cx-zammad-bridge/config.yaml`, but the code does not look there. |
| `Unable to create Zammad bridge` after about two minutes | The 3CX login failed (wrong host, client ID/secret, or 3CX not reachable). Check `3CX.host` and the credentials. |
| `go test -race` fails with `-race requires cgo` | No C compiler on your machine. Rely on CI, which runs the race test. |
| `goimports -d` prints a diff for every file on Windows | Git converted line endings to CRLF on checkout (`core.autocrlf=true`). The files in the repo use LF. Clone with `git -c core.autocrlf=false clone …`, or ignore the local check and rely on CI. |
| Nothing is ticketed, and the log says `extension_filter_mode is unset` | Intended: an unset mode fails closed. Set `Zammad.extension_filter_mode` to `include` (with a list) or `all`. |
| `Admin UI enabled but user/pass empty — refusing to start` | Set both `Admin.user` and `Admin.pass`. |
| The repo's `docker-compose.yml` fails on your machine | It is the old single-host setup: it mounts the config from `//e/3cx-zammad-bridge/config.yaml` (a Windows drive under Docker Desktop) and joins an external `zammad_default` network. Change the volume path and remove the network block for local use. Production does not use this file (see [Deploy & operate](deploy-operate.md)). |
