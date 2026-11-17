---
owner: group:default/dto
last_reviewed: 2026-11-17
---

# Reference

## Configuration

All settings are in one YAML file. The bridge reads the first file it can parse from: `config.yaml` in the working directory, `/etc/3cx-zammad-bridge/config.yaml`, `//3cx-zammad-bridge/config.yaml`. With `-c <path>` it reads only that file. (The upstream README also lists `/opt/3cx-zammad-bridge/config.yaml`; the code does not read it.)

**Where it lives in production:** `/opt/zammad/3cx-bridge-config.yaml` on the Zammad EC2 host, mounted into the container at `/etc/3cx-zammad-bridge/config.yaml`. Ask DTO or DevOps for access to the host file. The template for new installs is `config.yaml.dist` in this repo.

Values are never written here. The card's **Config** tab lists no settings for this component.

| Setting | Required | Added by AGM | What it does |
|---|---|---|---|
| `Bridge.poll_interval` | yes | no | Seconds between 3CX polls (decimal). |
| `3CX.host` | yes | no | Base URL of the 3CX server. |
| `3CX.client_id`, `3CX.client_secret` | yes on 3CX v20+ | no | API client from 3CX Admin → Integrations → API. **Secret.** |
| `3CX.user`, `3CX.pass`, `3CX.group` | only before v20 | no | Legacy login and the group to monitor. `pass` is a **secret**. |
| `3CX.extension_digits` | yes | no | Length of internal extension numbers. Numbers longer than this are treated as external (AGM change, see below). |
| `3CX.trunk_digits` | no (not read) | no | Upstream setting for trunk number length. The AGM code no longer reads it (since `051fa55`); keep it in the file for upstream compatibility. |
| `3CX.queue_extension` | yes | no | The queue the bridge also watches. Unanswered queue calls are logged against this number. |
| `3CX.country_prefix` | no | no | Country code (for example in `+CC…`, `00CC…` or `CC…` form) replaced by a leading `0` before numbers are sent to Zammad. Numbers with another country code keep their `+`. |
| `Admin.enabled` | no (default `false`) | yes | Starts the admin page. |
| `Admin.listen` | no (default `:8090`) | yes | Bind address of the admin page. |
| `Admin.user`, `Admin.pass` | yes if enabled | yes | HTTP Basic Auth for the admin page. The page refuses to start if either is empty. `pass` is a **secret**. |
| `Admin.base_path` | no (default empty = `/`) | yes | Path prefix for the admin page. Production uses `/bridge-admin` so the ALB can forward without rewriting. |
| `Zammad.endpoint` | yes | no | Zammad CTI webhook URL. The URL contains the CTI secret, so treat the whole value as a **secret**. |
| `Zammad.log_missed_queue_calls` | no | no | When `true`, an unanswered queue call is sent as a missed call, with `queue_extension` as the answering number. |
| `Zammad.api_url` | yes for auto-create | yes | Base URL of the Zammad REST API (production: the Compose service name, inside the host). |
| `Zammad.api_token` | yes for auto-create | yes | Zammad API token. **Secret.** |
| `Zammad.auto_create_ticket` | no (default `false`) | yes | Master switch for creating tickets on hangup. Editable on the admin page. |
| `Zammad.ticket_group` | no (default `Users`) | yes | Zammad group for new phone tickets, and the group searched by the de-duplication window. |
| `Zammad.auto_create_directions` | no (empty = `all`) | yes | `all`, `inbound`, `outbound` or `none`. Unknown values create nothing. Editable on the admin page. |
| `Zammad.extension_filter_mode` | **yes** for any ticket to be created | yes | `include` (only listed extensions), `exclude` (all but listed), `all` (every extension). **Empty or missing = create nothing** (fail closed, with a warning at start-up). Editable on the admin page. |
| `Zammad.extension_filter` | with `include`/`exclude` | yes | List of 3CX extension numbers (the agent side of the call). To ticket unanswered queue calls, include the `queue_extension` number. Editable on the admin page. |
| `Zammad.auto_create_dedup_window_minutes` | no (default `0` = off) | yes | When the caller's newest new/open ticket in `ticket_group` was **created** no more than this many minutes ago, the call is appended to it instead of opening a new ticket. The window counts from that ticket's `created_at`, not from the previous call, and appends do not extend it. Needs Zammad's Elasticsearch. Editable on the admin page. |
| `Zammad.call_id_field` | no (default empty = off) | yes | Name of a Zammad ticket attribute (create it first in Zammad Admin → Objects) that receives the 3CX call ID. |

Command-line flags: `-c/--config <path>`, `-f/--log-format json|plain` (default `json`; the Docker image runs with `-f plain`), `-v/--verbose`, `--trace`.

!!! warning "Admin page saves rewrite the whole file"
    A save writes the config back from the bridge's internal settings structure. All comments are lost, and any key the bridge does not know is dropped. Keep a backup of the file before you use the admin page after an upgrade.

## API

The bridge serves no business API. The only HTTP server is the operator admin page, so there is no OpenAPI file.

**Admin page** (only when `Admin.enabled` is `true`; `<base>` is `Admin.base_path`):

| Method and path | Auth | What it does |
|---|---|---|
| `GET <base>/` | Basic Auth | HTML page showing the current auto-create settings and the 3CX extension list. |
| `POST <base>/save` | Basic Auth, plus a same-site check (rejects `Sec-Fetch-Site: cross-site/same-site` and a foreign `Origin`) | Form fields `inbound`, `outbound` (checkboxes), `extension_filter_mode`, `extension_filter` (repeated or one per line), `dedup_window_minutes`. Writes the config file, then applies the settings to the running bridge. |
| `GET <base>/healthz` and `GET /healthz` | none | Returns `ok`. Used by the ALB target group health check. |

**Calls the bridge makes** (it is a client of these APIs):

| To | Calls | Auth |
|---|---|---|
| 3CX v20+ | `POST /connect/token`; `GET /xapi/v1/ActiveCalls`; `GET /xapi/v1/Users`, `/xapi/v1/Queues`, `/xapi/v1/RingGroups` (admin page) | Client ID and secret, then a bearer token |
| 3CX before v20 | `/api/login`, `/api/activeCalls`, `/api/GroupList`, `/api/GroupList/set`, `/api/edit/update` | User and password (cookie session) |
| Zammad CTI webhook | `POST <Zammad.endpoint>` with JSON fields `event` (`newCall`, `answer`, `hangup`), `from`, `to`, `direction` (`in`/`out`), `call_id`, `callid`, `cause`, `answeringNumber`, `user` | Secret in the URL |
| Zammad REST API | `GET /api/v1/users/search`, `POST /api/v1/users`, `GET /api/v1/tickets/search`, `POST /api/v1/tickets`, `POST /api/v1/ticket_articles` | `Authorization: Token token=<api_token>` |

## Data

- **No database, bucket or queue of its own.** Ongoing calls, the current auto-create settings and a 5-minute cache of the 3CX extension list are kept in memory and lost on restart.
- **Config file** (`/opt/zammad/3cx-bridge-config.yaml`): the only state on disk. The admin page writes to it.
- **Records it creates in Zammad** (when auto-create is on):
  - Customers: first name = the phone number, last name `3CX`, phone = the number. Created only when no user has that phone number.
  - Tickets: title `Phone Call from <number> (Inbound|Outbound|Missed)` (for outbound calls `<number>` and the article's `Caller:` line are the agent extension, not the customer), in `ticket_group`, with one `phone` article (caller, agent, call type, direction, `3CX Call ID`). If `call_id_field` is set, the 3CX call ID is also written to that ticket attribute. If the customer cannot be created, the ticket goes to Zammad user ID 1.
  - Articles: a repeat call within the de-duplication window (counted from the existing ticket's creation time) adds a `phone` article to that ticket instead.
- **Personal data:** caller phone numbers and agent names pass through the bridge and are stored in Zammad. The bridge logs phone numbers at info level (for example `Created Zammad user`), so container logs contain personal data.
- **Container images:** ECR repository `3cx-zammad-bridge` (eu-central-1), tags `:<commit SHA>` and `:latest`.

## AGM changes

**Base:** GitHub fork of [qmexnetworks/3cx-zammad-bridge](https://github.com/qmexnetworks/3cx-zammad-bridge). The fork point is upstream tag **v0.8.0** (commit `580c20c`). On 2026-11-17, AGM `main` (`c9fd977`) is 15 commits ahead of and 15 behind upstream `main` (v0.9.0). Upstream has never received a pull request from anyone, so every AGM change below is **not sent** upstream.

**Branches:** `main` holds every AGM change. `3.5` is an old upstream branch (Python, 2021, upstream author only) with no AGM commits. `feature/configurable-auto-create` (PR #1, closed unmerged) is fully replaced by PR #4: everything it has, `main` has in a newer form. The PR branches for #2–#5 and a CI debug branch (`debug-and-retrigger-oidc`, workflow-file changes only) have been deleted.

**What is in production:** the last recorded production image is `c9fd977` (current `main`), per the DevHub service-catalog audit of 2026-11-09. `zammad.tf` still pins `bridge_image_tag = "3e97fa9"` (2026-08-24), and the instance ignores `user_data` changes, so Terraform does not show what runs. Rows marked "yes" below are in both `3e97fa9` and `c9fd977`. See [Known gaps](#known-gaps) for the check that settles this.

| Change | Why (business reason) | Branch / PR / commit | Files touched | In production? | Upstream status |
|---|---|---|---|---|---|
| Container packaging: multi-stage Go 1.25 Dockerfile; config mounted at `/etc/3cx-zammad-bridge/config.yaml`; plain-text logs | Run the bridge as a container next to Zammad | `main`: `22e6d08`, `17fd2c9`, `defd142`, `23df3ea` | `Dockerfile` | yes: the ECR workflow builds this Dockerfile | not sent |
| Old single-host `docker-compose.yml` (Windows drive path, external `zammad_default` network, admin port on 127.0.0.1) | First deployment on the on-premise server | `main`: `17fd2c9`, `6d4055e`, `9fded5c`, `23df3ea`; port added in #4 | `docker-compose.yml` | no: production uses `/opt/zammad/docker-compose.yml` on the EC2 host | not sent |
| Poll calls from XAPI `/xapi/v1/ActiveCalls` instead of `/callcontrol`, and parse the `ext Name (phone)` party format | `/callcontrol` only shows calls for the logged-in extension; ActiveCalls shows every call on the system | `main`: `d5a821f` | `bridge_3cx_post20.go` | yes | not sent |
| Inbound/outbound detection by "longer than `extension_digits`" instead of an exact `trunk_digits` match | External numbers vary in length (10–14 digits) | `main`: `051fa55` | `bridge.go` | yes | not sent |
| Use the caller number only, not number + name, for the phone number | Correct phone numbers in Zammad | `main`: `3c77166` | `bridge.go` | yes | not sent |
| Create a Zammad phone ticket when a call ends (answered or missed) | Every LC call becomes a trackable ticket | `main`: `22e6d08` | `config.go`, `zammad.go` | yes | not sent |
| Create the Zammad customer by phone number when none is found | Tickets need a customer; new callers had none | `main`: `d0366f2` | `zammad.go` | yes | not sent |
| Direction and extension filters (`auto_create_directions`, `extension_filter_mode`, `extension_filter`) | The LC team asked (April 2026) to stop tickets for outbound calls and for inbound calls answered by a non-LC queue | PR #4, `3e97fa9` (replaces PR #1) | `autocreate.go`, `autocreate_test.go`, `config.go`, `config.yaml.dist`, `zammad.go`, `README.md` | yes | not sent |
| Fail closed when `extension_filter_mode` is unset, with a start-up warning | When the Automotive brands joined the same 3CX, their calls filed about 2,100 tickets a day into the London Cab phone group | PR #4, `3e97fa9` | `autocreate.go`, `bridge.go`, `admin.go`, `config.go`, `config.yaml.dist`, `README.md` | yes | not sent |
| Admin page with Basic Auth: change filters live, no restart; extension picker from the 3CX directory (users, queues, ring groups) | Let operators maintain the LC extension list without an engineer | PR #4, `3e97fa9` | `admin.go`, `admin_test.go`, `bridge.go`, `brigde_3cx.go`, `bridge_3cx_post20.go`, `bridge_3cx_pre20.go`, `cmd/main.go`, `config.go`, `docker-compose.yml` | yes: ALB rule `/bridge-admin/*` → port 8090 | not sent |
| `Admin.base_path` so the admin page works behind the AWS ALB | The ALB cannot strip a path prefix; `/bridge-admin/` returned 404 after the AWS migration | PR #4, `3e97fa9` | `admin.go`, `config.go`, `config.yaml.dist`, `README.md` | yes | not sent |
| Same-site (CSRF) check on `POST /save` | Basic Auth alone lets any web page the operator visits change the filters | PR #4, `3e97fa9` | `admin.go`, `admin_test.go` | yes | not sent |
| Repeat-call de-duplication window (`auto_create_dedup_window_minutes`) | The LC team reported duplicate tickets when a customer re-dials within minutes | PR #4, `3e97fa9` (design and plan in `docs/superpowers/`) | `zammad.go`, `zammad_test.go`, `autocreate.go`, `admin.go`, `config.go`, `config.yaml.dist`, `README.md`, `docs/superpowers/**` | yes | not sent |
| Key the ticket customer on the external number; URL-escape the phone search | Outbound tickets were owned by a fake customer named after the agent extension; callers with a leading `+` were never found | PR #4, `3e97fa9` | `zammad.go`, `zammad_test.go` | yes | not sent |
| Race-safe 3CX access token and extension cache; `go test -race` step in CI | The admin page runs in its own goroutine next to the polling loop | PR #4, `3e97fa9` | `bridge_3cx_post20.go`, `bridge.go`, `.github/workflows/go.yml` | yes | not sent |
| 3CX call ID on every call article, plus optional `call_id_field` ticket attribute | Join phone tickets to the 3CX call log (GLPI #14781) | PR #5, `c9fd977` | `zammad.go`, `zammad_test.go`, `config.go`, `config.yaml.dist`, `README.md` | yes, per the 2026-11-09 audit (image `c9fd977`); Terraform still pins `3e97fa9`, which does not have it | not sent |
| Build and push the image to AGM ECR through GitHub OIDC | Build images in CI; deploy stays manual | PR #3, `078b3f2` | `.github/workflows/deploy-to-agm-ecr.yml` | yes: it built `3e97fa9` and `c9fd977` | not sent (AGM-only) |
| DevHub catalog entry and TechDocs | Register the bridge in DevHub | PR #2, `e1a99c1` / `6b3f729`; updated by branch `docs/devhub` | `catalog-info.yaml`, `mkdocs.yml`, `docs/**` | not applicable (docs) | not sent (AGM-only) |

## Known gaps

Things the repo cannot prove. Each needs an answer from the owner.

1. TODO(owner): Which image runs today? Run `sudo docker inspect --format '{{.Config.Image}}' zammad-3cx-zammad-bridge-1` on the host. The 2026-11-09 audit says `c9fd977`; `zammad.tf` pins `3e97fa9`.
2. TODO(owner): Terraform has drifted from the host: the live config and compose file are edited in place (for example the extension list and `call_id_field` were changed on the host in October 2026). Which is the source of truth, and who updates the other?
3. TODO(owner): Do admin page saves work in production? The Terraform compose template mounts the config read-only (`:ro`), and the bridge writes the file **before** applying a change. On a read-only mount every save would fail with `Could not write config` and change nothing. Is the live mount read-only?
4. TODO(owner): `docs/superpowers/` holds working design notes that are kept out of the DevHub site. Should they stay in the repo?
5. TODO(owner): Which 3CX admin account owns the API client the bridge uses, and who may rotate its secret?
6. TODO(owner): Settings outside the repo that the bridge depends on: the CTI (generic) integration settings in Zammad's admin UI, the Zammad user behind the API token and its permissions, the ticket group in `ticket_group`, and the ticket attribute in `call_id_field`. Who owns each?
7. TODO(owner): Is there any alarm on the bridge (container down, `zammad-bridge-admin-tg` unhealthy, no tickets created for N hours)?
8. TODO(owner): Is there a test 3CX and a test Zammad that engineers may use? There is no dev environment.
9. TODO(owner): Other docs are out of date and should be fixed by their owners: `agm-infrastructure` → `docs/projects/zammad.md` describes a 3CX webhook, a `3CX_WEBHOOK_SECRET` and a `3cx-bridge` service, none of which exist; the `zammad.tf` header calls this a "personal-org repo"; the DevHub system card `devhub-catalog` → `systems/zammad.yaml` links `zammad.agmegypt.com`, which does not point at the Zammad load balancer (`helpdesk.agmegypt.com` does).
10. TODO(owner): Fix these inaccuracies in this repo in a separate code PR? These docs work around them: `config.yaml.dist` says an admin save "restarts the process" (it does not; it applies live); the README lists a config path the code does not read; the "Rollout hint" step in `.github/workflows/deploy-to-agm-ecr.yml` is stale (it says compose runs `:latest`, omits `--region`, and says `auto_create_ticket` is false), and it contradicts [How to deploy](deploy-operate.md#how-to-deploy); `.github/workflows/go.yml` sets up Go 1.23 while `go.mod` needs Go 1.25 (CI passes today).
