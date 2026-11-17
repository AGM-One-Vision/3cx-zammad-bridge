---
owner: group:default/dto
last_reviewed: 2026-11-17
---

# Deploy & operate

## Environments

There is **one environment: production**. There is no dev or staging bridge.

| Branch | Environment | Where it runs | URL |
|---|---|---|---|
| `main` (an image is built on every push) | Production, after a manual roll-out | Container `zammad-3cx-zammad-bridge-1` in the Docker Compose project `/opt/zammad/docker-compose.yml` on the Zammad EC2 host `zammad-prod` (instance `i-04ab9be06910cba01`, eu-central-1, private subnet, reached only through SSM Session Manager) | Admin page: <https://helpdesk.agmegypt.com/bridge-admin/>. The bridge itself has no public URL; it polls 3CX and posts to Zammad inside the host. |
| any other branch | None | Can be built into an image by hand (`workflow_dispatch`), but nothing runs it | None |

- Config file on the host: `/opt/zammad/3cx-bridge-config.yaml`, mounted into the container at `/etc/3cx-zammad-bridge/config.yaml`.
- The host, its security group, the ALB rule and the ECR repository are defined in `AGM-One-Vision/agm-infrastructure` → `environments/prod/zammad.tf`.
- History: until the AWS migration on 2026-06-30 the bridge ran on the old on-premise server (now decommissioned) with the `docker-compose.yml` in this repo. That file is no longer used in production.

## CI/CD stages

| Workflow | Trigger | What it does |
|---|---|---|
| `.github/workflows/go.yml` ("Go") | push and pull request to `main` | `go build -v ./...`, `go test -v ./...`, `go test -race ./...`, then a `goimports -d -e .` check. |
| `.github/workflows/deploy-to-agm-ecr.yml` ("Build & Push to AGM ECR") | push to `main`, or manual `workflow_dispatch` on any branch | Assumes the AWS role `bridge-github-actions-deploy` through GitHub OIDC, builds the repo `Dockerfile`, and pushes to ECR repository `3cx-zammad-bridge` (eu-central-1) with tags `:<7-char commit SHA>` and `:latest` (plus an optional extra tag on manual runs). It does **not** deploy. |
| `.github/workflows/release.yaml` | a GitHub release is created | Upstream's binary release job (linux/amd64). AGM has never created a release in this repo, so it has never run here. |

Deployment to the host is manual on purpose. The image and the config must change together: a new image with an old config, or the reverse, can stop London Cab tickets or flood Zammad with tickets from other business lines.

Any push to `main` builds and pushes a new image (it does not deploy it), so treat every merge to `main` as a release candidate.

## How to deploy

1. **Merge to `main`.** Wait for both "Go" and "Build & Push to AGM ECR" to pass. Note the 7-character SHA of the merge commit; that is the image tag.
2. **Check config impact.** Read the PR for new or changed config keys (see [Reference → Configuration](reference.md#configuration)). If keys must change, prepare the new config file first.
3. **Open a shell on the host:**
   ```bash
   aws ssm start-session --region eu-central-1 --target i-04ab9be06910cba01
   cd /opt/zammad
   ```
4. **Back up the config and the compose file:**
   ```bash
   TS=$(date +%Y%m%d%H%M%S)
   sudo cp 3cx-bridge-config.yaml 3cx-bridge-config.yaml.bak.$TS
   sudo cp docker-compose.yml docker-compose.yml.bak.$TS
   ```
   Other people edit these files in place too. Always diff against a fresh backup before you change them.
5. **Set the new image tag.** In `docker-compose.yml`, service `3cx-zammad-bridge`, change the `image:` tag to the new SHA. Production pins a SHA, so `docker compose pull` alone does nothing until you change this line. Do not switch to `:latest`.
6. **Apply config changes, if any,** to `3cx-bridge-config.yaml`.
7. **Roll the container:**
   ```bash
   sudo docker compose pull 3cx-zammad-bridge
   sudo docker compose up -d 3cx-zammad-bridge
   ```
8. **Verify:**
   ```bash
   sudo docker logs --since 5m zammad-3cx-zammad-bridge-1
   curl -s localhost:8090/healthz        # expect: ok
   ```
   - The log shows `Starting 3CX-Zammad bridge` and `Admin UI listening ... base_path=/bridge-admin/`.
   - The log does **not** show `extension_filter_mode is unset`.
   - <https://helpdesk.agmegypt.com/bridge-admin/> asks for a password and then shows the settings.
   - A test call answered on an allow-listed extension creates one ticket in the configured group; a call on another extension creates none.
9. **Record the tag in Terraform.** Open a PR in `agm-infrastructure` that sets `variable "bridge_image_tag"` in `environments/prod/zammad.tf` to the same SHA. The instance ignores `user_data` changes, so a Terraform apply does **not** change the running host; this step only keeps the record true.

## Rollback

- **Bad image:** set the `image:` tag in `/opt/zammad/docker-compose.yml` back to the previous SHA (it is in the `.bak` file from step 4), then `sudo docker compose up -d 3cx-zammad-bridge`.
- **Bad config:** copy the `.bak` config back over `3cx-bridge-config.yaml`, then `sudo docker compose restart 3cx-zammad-bridge`.
- **Stop ticket creation fast (kill switch):** set `Zammad.auto_create_ticket: false` in `/opt/zammad/3cx-bridge-config.yaml` and run `sudo docker compose restart 3cx-zammad-bridge`. CTI call events keep flowing; only ticket creation stops. Switching off both "Inbound" and "Outbound" on the admin page does the same, but **only if** the config mount is writable (see [Reference → Known gaps](reference.md#known-gaps), item 3); otherwise the save fails with `Could not write config` and nothing changes, so use the config file.
- ECR keeps older SHA tags, so a previous image can be pulled again. The lifecycle policy (`agm-infrastructure` → `environments/prod/ecr-lifecycle.tf`) keeps the last 30 images whose tags do not start with `live-`, `dev-` or `prod-`. The bridge's SHA tags have no such prefix, so the running image is not protected from expiry once 30 newer images exist.

## When an alarm fires

There is **no alarm for the bridge itself**. The Zammad host alarms (memory, disk, instance status, ALB 5xx and unhealthy hosts) are described in the Zammad runbook in `AGM-One-Vision/agm-infra-notes` → `runbooks/zammad.md`. The admin page has an ALB target group, `zammad-bridge-admin-tg`, with health check `GET /bridge-admin/healthz`. TODO(owner): does any alarm watch `zammad-bridge-admin-tg`, and should there be an alarm when the bridge stops creating tickets?

Usual signals are "no phone tickets", "duplicate tickets" or "too many tickets". First look:

```bash
aws ssm start-session --region eu-central-1 --target i-04ab9be06910cba01
sudo docker ps --filter name=3cx-zammad-bridge
sudo docker logs --since 30m zammad-3cx-zammad-bridge-1
```

| What you see | What it means | What to do |
|---|---|---|
| `auto_create_ticket is on but extension_filter_mode is unset` at start-up | The filter mode is missing, so the bridge creates no tickets (fail closed). | Set `extension_filter_mode` in the config (normally `include`) and restart. |
| `unable to authenticate`, or the container keeps restarting | The 3CX login failed for two minutes and the process exited; Docker restarts it. | Check that 3CX is up and the API client ID and secret are still valid. |
| `Error processing calls` (repeated) | Fetching active calls from 3CX failed with an error other than 401/403. | Check that 3CX is reachable from the host. |
| `unexpected response from Zammad` or `unable to make request`, logged with the message `new-call`, `answer`, `hangup-from-routing`, `hangup-from-talking` or `hangup-from-transferring` | Posting to the Zammad CTI webhook failed. Events, and tickets for calls that ended, are lost for that time. | Check Zammad health (runbook above), and that the CTI integration is still on in Zammad. |
| `Failed to create Zammad ticket` | The REST call to create a ticket failed. | Check the API token is valid and the group in `ticket_group` exists. |
| `Dedup lookup failed; creating a new ticket` | The ticket search failed. Duplicates are expected until it is fixed. | Check Zammad's Elasticsearch. |
| Duplicate tickets and no lookup errors in the log | Often expected: the window counts from the **creation time of the newest new/open ticket**, not from the previous call, so a call after the window opens a new ticket even if earlier calls were appended (window 10: calls at 0, 8 and 12 minutes give two tickets). Also expected when the earlier ticket is no longer new/open, is in another group, or the caller matched a different Zammad user. Otherwise the search returns nothing without an error, usually because Elasticsearch is down (Zammad falls back to SQL search, which never matches this query). The "No recent open phone ticket matched" line is debug-level and hidden in production. | Rule out the expected cases first, then check Zammad's Elasticsearch. |
| Tickets from extensions that should not create them | The extension filter is wrong. | Fix the list on the admin page or in the config. |
| Admin page returns 404 or 502 | The container is down, `Admin.enabled` is false, `Admin.base_path` is not `/bridge-admin`, or the ALB target is unhealthy. | Check `docker ps`, the start-up log line `Admin UI listening`, and the target group health. |
| Admin page save says `Could not write config` | The bridge could not write the config file, so the change was **not** applied. | See the read-only mount question in [Reference → Known gaps](reference.md#known-gaps). Edit the file on the host and restart instead. |

## Upgrading

This fork was taken at upstream release **v0.8.0** (commit `580c20c`). Upstream has since released **v0.9.0** (2026-10-15, 15 commits). AGM has 15 commits on `main` that upstream does not have. AGM has never merged upstream back.

A trial merge (`git merge-tree --write-tree --name-only main upstream/main`, run 2026-11-17) reports conflicts in:
`.github/workflows/go.yml`, `Dockerfile`, `bridge.go`, `bridge_3cx_post20.go`, `cmd/main.go`, `config.go`, `config.yaml.dist`.

Steps:

1. Add the upstream remote and fetch it:
   ```bash
   git remote add upstream https://github.com/qmexnetworks/3cx-zammad-bridge.git
   git fetch upstream --tags
   ```
2. Create a branch from `main` and **merge** the upstream tag. Do not rebase `main` onto upstream: `main` is shared and its commits are image tags, so rewriting it breaks the link between images and commits.
   ```bash
   git switch -c chore/upstream-v0.9.0 main
   git merge v0.9.0
   ```
3. Resolve each conflict so that every row in [Reference → AGM changes](reference.md#agm-changes) still holds. Known traps in v0.9.0:
   - **Dockerfile.** Upstream's new Dockerfile is a distroless image that copies a binary built beforehand by GoReleaser (`COPY $TARGETPLATFORM/zammadbridge`). AGM's ECR workflow runs a plain `docker build .` and needs AGM's multi-stage Dockerfile. Keep AGM's Dockerfile, or change the workflow to build with GoReleaser first.
   - **File rename.** Upstream renamed `brigde_3cx.go` to `bridge_3cx.go`. Git detects the rename and merges AGM's changes into `bridge_3cx.go` (the trial merge did this without a conflict). Check that `brigde_3cx.go` is gone afterwards and the AGM `FetchExtensions` interface method is still there.
   - **3CX polling.** Upstream removed the unused 3CX websocket listener and changed `bridge_3cx_post20.go`. AGM moved call polling to XAPI `/xapi/v1/ActiveCalls` and added the extension directory calls in the same file. Keep the XAPI calls and the race-safe access token.
   - **Outbound calls.** Upstream added `Zammad.log_outbound_calls` (default `true`), which keeps outbound calls out of Zammad's CTI entirely. AGM's `auto_create_directions` controls only ticket creation. They are different settings; keep both and document the difference.
   - **Go version.** Upstream moved to Go 1.27.2. Update `go.yml` and the `golang:` image in the Dockerfile to match `go.mod`.
   - **Config file.** Upstream's `LoadConfigFromYaml` now sets defaults before parsing. Re-check AGM's config defaults and the admin save path.
   - **Start-up and shutdown.** Upstream `daa576e` (clean shutdown on SIGTERM) changes `cmd/main.go`. Keep AGM's `go zammadbridge.StartAdminServer(...)` call.
   - **Hangup timing.** Upstream `5c0d278` retries fetching calls before ending them in Zammad, which changes when `hangup` fires. AGM ticket creation runs on `hangup`, so re-test auto-create and de-duplication.
   - **CI.** Upstream's `go.yml` adds golangci-lint and has no `-race` step. Keep AGM's race step.
   - **Admin page saves.** A save rewrites the config from the `Config` struct, so any key the struct does not know is dropped and all comments are lost. Make sure every new upstream key is in the struct before anyone saves from the admin page.
4. Run `go build ./...` and `go test ./...` locally, and let CI run `go test -race ./...`. AGM's own behaviour is covered by `admin_test.go`, `autocreate_test.go` and `zammad_test.go`; upstream v0.9.0 adds `bridge_test.go`, `bridge_3cx_post20_test.go` and `config_test.go`, which may need updating for AGM's XAPI changes.
5. Open a PR to `main`. After merge, CI pushes a new image. Deploy it with the steps in [How to deploy](#how-to-deploy), and add any new upstream keys to the production config first.
