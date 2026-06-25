# astrojones/.github — org-wide reusable workflows

This **is** `astrojones/.github`: the org's single source of truth for reusable
GitHub Actions workflows. Apps call these with `uses:`; they don't duplicate them.
It also hosts the ephemeral Hetzner CI-runner system documented further below.

## Reusable workflow catalog

All reusable workflows are called as:

```yaml
uses: astrojones/.github/.github/workflows/<file>@main
```

A reusable (`uses:`) job cannot contain `steps:`. Pass secrets with
`secrets: inherit` unless a workflow declares an explicit `secrets:` block (then
pass those by name). `GITHUB_TOKEN` is always provided automatically — the
"(auto)" entries below need no `secrets:` at all.

| Workflow | Kind | Required inputs | Secrets |
|---|---|---|---|
| `ci.yml` | reusable | `language` | (none) |
| `app-ci.yml` | reusable | `images` | `GITHUB_TOKEN` (auto) |
| `release.yml` | reusable | (none) | `GITHUB_TOKEN` (auto) |
| `nuk-deploy.yml` | reusable | (none) | `secrets: inherit` — opt. `APP_ENV`, org env |
| `nuk-apply.yml` | reusable | (none) | `GITHUB_TOKEN` (auto) |
| `nuk-preview.yml` | reusable | `mode` | `GITHUB_TOKEN` (auto) |
| `e2e-compose-playwright.yml` | reusable | `runner_label`, `backend_image`, `frontend_image` | `GITHUB_TOKEN` (auto) |
| `migration-shadow-check.yml` | reusable | `prod_postgres_container`, `backend_image`, `backend_tag`, `critical_tables` | `GITHUB_TOKEN` (auto) |
| `runner-up.yml` | reusable | (none — all defaulted) | `HCLOUD_TOKEN`, `RUNNER_PAT` |
| `runner-down.yml` | reusable | `server` | `HCLOUD_TOKEN` |
| `runner-sweep.yml` | **internal** (scheduled, not `workflow_call`) | (n/a) | `HCLOUD_TOKEN` (org/repo secret) |

Per-workflow contracts (inputs show `name` *(default)*; **bold** = required):

### `ci.yml` — org-wide minimum CI gate
Lint + format + test where they exist; a trivial green for content-only repos.
- Inputs: **`language`** (`python` \| `node` \| `static`), `python-version` *(3.12)*, `node-version` *(20)*.
- Secrets: none. Permissions: `contents: read`.

### `app-ci.yml` — build + push images to GHCR
Builds a JSON matrix of images, pushes `ghcr.io/<owner>/<repo>/<name>:<sha>` (+`:latest`).
- Inputs: **`images`** (JSON array of `{name, context, dockerfile?, build_args?, prebuild?}`), `push` *(true)*, `tag_latest` *(true)*, `extra_tags` *('')*.
- Secrets: `GITHUB_TOKEN` (auto). Needs `packages: write` (granted internally).

### `release.yml` — no-PR conventional-commit release
Derives the next version, stamps version files, commits to main `[skip ci]`, tags + releases.
- Inputs: `version-files` *('')*, `runs-on` *(`["self-hosted","nuklaut"]`)* — **public repos MUST override to `["ubuntu-latest"]`**.
- Outputs: `released` (true/false), `tag`.
- Secrets: `GITHUB_TOKEN` (auto). Permissions: `contents: write`.

### `nuk-deploy.yml` — build + deploy a single-image app
Builds the image, writes per-app + org-shared env files, runs `nuk apply` on the controller.
- Inputs: `dockerfile` *(Dockerfile)*, `build_context` *(.)*, `manifest` *(.nuklaut/deployment.yml)*.
- Secrets: `secrets: inherit`. Optional repo secret **`APP_ENV`** (multiline `key=value`); reads org-wide `ORG_SENTRY_DSN`, `ORG_OPENAI_API_KEY` into `_shared.env`; `GITHUB_TOKEN` (auto).
- Runs on `[self-hosted, nuklaut]`.

### `nuk-apply.yml` — deploy-only (images already in GHCR)
For multi-image stacks built by their own CI. Pulls images into the shared daemon, `nuk apply`. Does NOT build.
- Inputs: `manifest` *(.nuklaut/deployment.yml)*, `compose_file` *(docker-compose.yml)*.
- Secrets: `GITHUB_TOKEN` (auto, via `secrets: inherit`). Runs on `[self-hosted, nuklaut]`.

### `nuk-preview.yml` — per-PR ephemeral preview stack
Routes a preview at `pr-<n>.<base_host>` via Traefik. Three modes via `mode`.
- Inputs: **`mode`** (`deploy` \| `teardown` \| `cleanup`). For `deploy`/`teardown` also supply `pr_number` + `app` (and `base_host` for `deploy`) — declared optional but validated at runtime. Plus `images` *({})*, `compose_file` *(docker-compose.pr.yml)*, `manifest_template` *('')*, `backend_service` *(backend)*, `backend_port` *(8000)*, `frontend_service` *(frontend)*, `frontend_port` *(80)*, `max_previews` *(2)*, `comment` *(true)*.
- Secrets: `GITHUB_TOKEN` (auto). Runs on `[self-hosted, nuklaut]`.

### `e2e-compose-playwright.yml` — sharded Playwright E2E on an ephemeral box
Stands up a compose stack on the fresh box from `runner-up.yml` and runs a sharded Playwright suite.
- Inputs: **`runner_label`** (from `runner-up` output), **`backend_image`**, **`frontend_image`** (GHCR refs, no tag), `compose_file` *(docker-compose.e2e.yml)*, `image_tag` *(`${{ github.sha }}`)*, `migrate_cmd` *(alembic upgrade head)*, `seed_cmd` *('')*, `health_url` *(http://localhost:8001/health)*, `base_url` *(http://localhost:5174)*, `test_dir` *(frontend)*, `shard_count` *(5)*, `journey_project` *(journey)*, `stories_project` *(stories)*, `test_grep` *('')*.
- Outputs: `passed`. Secrets: `GITHUB_TOKEN` (auto). Runs on `[self-hosted, <runner_label>]`.

### `migration-shadow-check.yml` — pre-deploy migration safety gate
Snapshots live prod postgres into a throwaway DB, runs the candidate image's migrations, fails on row loss / orphan FKs.
- Inputs: **`prod_postgres_container`**, **`backend_image`** (no tag), **`backend_tag`**, **`critical_tables`** (space-separated), `app_db` *(app)*, `db_user` *(postgres)*, `migrate_cmd` *(alembic upgrade head)*, `orphan_check_sql` *('')*.
- Secrets: `GITHUB_TOKEN` (auto). Runs on `[self-hosted, nuklaut]`.

### `runner-up.yml` — provision a one-shot ephemeral Hetzner runner
Mints a 1-hour registration token and boots a fresh VM; exposes its unique label.
- Inputs: all defaulted — see the [full inputs table](#runner-upyml-inputs) below (`org`, `server_type`, `location`, `image`, `runner_image`, `ssh_key`, `firewall`, `cache_volume`, `cache_volume_size`, `template_ref`, `wait_for_runner`).
- Outputs: `label`, `server`, `ipv4`.
- Secrets: **`HCLOUD_TOKEN`**, **`RUNNER_PAT`** (both required). Runs on `ubuntu-latest`.

### `runner-down.yml` — delete an ephemeral runner
Idempotent teardown; any cache volume is preserved. Always call with `if: always()`.
- Inputs: **`server`** (from `runner-up`'s `server` output).
- Secrets: **`HCLOUD_TOKEN`** (required). Runs on `ubuntu-latest`.

### `runner-sweep.yml` — hourly orphan-runner backstop *(internal)*
Not a reusable workflow — runs on `schedule` (`17 * * * *`) and `workflow_dispatch` in this repo. Deletes `role=gha-runner` servers older than `max_age_minutes` *(120)*.
- Secrets: `HCLOUD_TOKEN` (set as an org secret, or a repo secret on `.github`).

## Ephemeral Hetzner CI runners

One **unified, workload-agnostic** ephemeral runner for the whole org. A consumer
workflow brings up a fresh Hetzner VM, runs one job on it (any toolchain, via
`container:`), then deletes it — billed only for the minutes it exists.

## How it works

```
 consumer repo (.github/workflows/e2e.yml)
 ┌───────────────────────────────────────────────────────────────┐
 │ up    ── uses: runner-up.yml ──► ubuntu-latest                 │
 │          • gh api → 1h registration token  (PAT stays in CI)   │
 │          • hcloud server create --user-data (cloud-init)       │
 │          • outputs: label, server, ipv4                        │
 │ build ── needs: up ──► runs-on:[self-hosted, <label>]          │
 │          • the ephemeral VM registers with <label>, runs ONE   │
 │            job (EPHEMERAL=true), de-registers, container exits  │
 │ down  ── needs:[up,build], if: always() ── uses: runner-down   │
 │          • hcloud server delete <server>   (cache vol survives)│
 └───────────────────────────────────────────────────────────────┘
```

The VM is a stock `debian-13` box. cloud-init installs Docker and launches
[`myoung34/github-runner`](https://github.com/myoung34/docker-github-actions-runner)
as a one-shot, org-scoped, ephemeral runner. Workloads run in their own
containers against the host Docker daemon (Docker-out-of-Docker via the mounted
socket), so the host stays generic — the same runner serves e2e, builds, `.tex`,
Playwright, etc.

## Security model

The **only** GitHub credential that ever reaches the VM is a **1-hour
registration token** (`RUNNER_TOKEN`), minted just-in-time in the `up` job. The
long-lived PAT is used only inside CI (`ubuntu-latest`) and is **never** baked
into the image or passed to the box.

- `up`/`down` run on **GitHub-hosted** `ubuntu-latest`, so ephemeral CI is
  decoupled from the controller's health.
- The runner mounts the host Docker socket = root-equivalent on the VM. That's
  acceptable because the VM is single-use and destroyed after one job. **Do not
  run untrusted PRs** on it without isolation.

> **Alternative (stricter PAT posture).** If you'd rather the PAT never become a
> GitHub secret at all, run `up`/`down` on the controller's `[self-hosted, nuklaut]`
> runner and read the PAT from `/etc/nuklaut` (SOPS) instead of `secrets.RUNNER_PAT`.
> Trade-off: every CI run then depends on the controller being up. Switching is a
> `runs-on` change + a token-source change in `runner-up.yml`/`runner-down.yml`.

## Required org Actions secrets

| Secret | What | Notes |
|---|---|---|
| `HCLOUD_TOKEN` | Hetzner Cloud API token | Used by `setup-hcloud`; the CLI has no `--token` flag, it reads this env var. |
| `RUNNER_PAT` | Fine-grained PAT | Org permission **Self-hosted runners: Read and write**. The token owner **must be an org owner** or org registration fails. Classic-PAT alternative: scope `admin:org`. |

`runner-sweep.yml` also needs `HCLOUD_TOKEN` available to `astrojones/.github`'s
own workflows (set it as an org secret, or a repo secret on `.github`).

## Consuming it

Copy `examples/consumer-e2e.yml` into a consumer repo as
`.github/workflows/<name>.yml`. The three-job shape is required because a
reusable-workflow `uses:` job cannot contain `steps:` — so the consumer owns the
middle `build` job. Two details that bite if you skip them:

- **`runs-on` must use the dashed block-sequence form**, not the inline bracket
  array (`[self-hosted, "${{...}}"]`), which is widely reported to misparse:
  ```yaml
  runs-on:
    - self-hosted
    - ${{ needs.up.outputs.label }}
  ```
- **`down` must be `if: always()` with `needs: [up, build]`**, or a failed build
  skips teardown and leaks a server (the hourly sweep is the backstop, not the
  primary).

### `runner-up.yml` inputs

| Input | Default | Purpose |
|---|---|---|
| `org` | `astrojones` | Org to register under. |
| `server_type` | `cpx32` | 4 vCPU / 8 GB / 160 GB. |
| `location` | `fsn1` | Hetzner location (`--datacenter` is deprecated). |
| `image` | `debian-13` | Base OS. |
| `runner_image` | `myoung34/github-runner:2.334.0` | Pinned runner (see cadence below). |
| `ssh_key` | `astrojones-deploy` | Debug SSH key name; empty to omit. |
| `firewall` | `""` | Hetzner Cloud Firewall name to attach (recommended hardening). |
| `cache_volume` | `""` | Docker-layer cache volume name; empty = cold boot each time. |
| `cache_volume_size` | `50` | GB, if the volume must be created. |
| `template_ref` | `main` | Ref of `<org>/.github` to read the cloud-init template from. |
| `wait_for_runner` | `true` | Block `up` until the runner registers (fail fast on boot errors). |

## Cache volume (opt-in)

Set `cache_volume:` to reuse Docker image layers (e.g. a multi-GB Playwright
image) across ephemeral boxes — cloud-init relocates Docker's `data-root` onto
the volume and **formats it only if blank**, so the cache survives.

- Hetzner volumes are **single-attach**: only one runner can use a given cache
  volume at a time. Don't share one cache across **concurrent** runs (sharded
  e2e) — give each lane its own volume, or run cold.
- **Flush the cache** by deleting + recreating the volume (`hcloud volume delete`).
  Server deletion never touches it.

## Version pin & bump cadence

`actions/runner` ships ~monthly and GitHub **refuses jobs to runners >30 days
behind** the latest release. We pin `myoung34/github-runner:2.334.0`
(actions/runner v2.334.0, 2026-04-21) with `DISABLE_AUTO_UPDATE=true` — correct
for one-job boxes (skips the 30–90 s self-update). **Bump `runner_image` within
~30 days of each new release** (Renovate/Dependabot on the tag, or a reminder).
This is the same drift trap that bit the controller runners — don't pin-and-forget.

## Manual test (Phase-1 measurement)

Before wiring CI, validate the cloud-init and measure cold vs warm boot. Mint a
token, render, and create by hand (requires `HCLOUD_TOKEN` + a registration
token in env):

```bash
export ORG_NAME=astrojones RUNNER_WORKDIR=/tmp/github-runner-astrojones
export RUNNER_IMAGE=myoung34/github-runner:2.334.0
export LABELS=ephemeral,manual-test
export RUNNER_TOKEN="$(gh api -X POST /orgs/astrojones/actions/runners/registration-token --jq .token)"
export VOLUME_ID=""   # or a real volume id to test the warm path

envsubst '$ORG_NAME $RUNNER_TOKEN $LABELS $RUNNER_IMAGE $RUNNER_WORKDIR $VOLUME_ID' \
  < cloud-init/runner.cloud-init.tpl.yaml > user-data.yaml

hcloud server create --name eph-manual --type cpx32 --image debian-13 \
  --location fsn1 --ssh-key astrojones-deploy \
  --user-data-from-file - < user-data.yaml

# time until it appears under org settings → Actions → Runners, then:
hcloud server delete eph-manual
```

Compare cold (`VOLUME_ID=""`) vs warm (real volume, second run) to decide whether
a prebaked snapshot/Packer image is ever worth it. Expectation: cold ≈ Docker
install + image pull each time; warm ≈ Docker install only (layers cached).

## Leak protection

- `down` (`if: always()`) is the primary teardown.
- `runner-sweep.yml` runs hourly and deletes any `role=gha-runner` server older
  than 120 min — the backstop for hard-cancelled runs.
