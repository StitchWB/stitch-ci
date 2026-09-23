# stitch-ci

Free-public-minutes CI/CD farm for the private monorepo
`StitchWB/Stitch-Account-Manager` (the hub) and friends. The hub carries no
trigger workflows and has GitHub Actions disabled repo-wide: every pipeline
lives here, checks the hub out read-only and reports commit statuses back to
hub SHAs (visible in hub PR Checks and `gh pr checks`).

## Control plane

| Workflow | Trigger | What it does |
|---|---|---|
| `poll.yml` | cron `*/10` + manual | Orchestrator: polls hub HEAD, tags and open PRs; dispatches the workflows below whose path filters match the change set; records state in `state/monorepo.json`. GitHub throttles cron on this org to ~4-5h in practice — use the manual button when a push must be verified now. |
| `ci.yml` | poll dispatch + nightly cron + manual | Runs each repo's `.ci.yml` contract (python tests, gates) + frontend checks; posts `stitch-ci/python` and `stitch-ci/frontend` statuses. With `pr_base` input also runs the `guard-mirrors` job (`stitch-ci/guard-mirrors` status). |
| `build-smoke.yml` | poll dispatch on build-relevant paths + manual | Nuitka + PyInstaller desktop build verification (Windows runner). |
| `export-services.yml` | poll dispatch on server/bot paths + manual | Standalone service export checks. |
| `publish-service-plugins.yml` | poll dispatch on `plugins-src/` + manual | Service plugin publishing. |
| `template-sync.yml` | poll dispatch on `template/`, `python/stitch_plugin_tools/` + manual | Check: regenerated template matches the committed scaffold. |
| `push-template.yml` | same paths + manual | Regenerates and pushes the public `stitch-plugin-template` repo. |
| `publish-engine-pack.yml` | poll dispatch on engine-pack/captcha/vendor paths + manual | Engine-pack publish. |
| `publish-provider-plugins.yml` | manual | Compile (Nuitka) + publish provider plugins. |
| `publish-compiled-plugins.yml` | manual | Publish artifacts built by stitch-build. |
| `release.yml` | poll dispatch on new `v*` tag + manual | Full private release. |
| `release-public.yml` | poll dispatch on new `v*` tag + manual | Open-core release built from the public `Stitch-Manager` repo at the tag. |
| `auto-release.yml` | manual | Version bump + tag on the hub (needs `MONOREPO_WRITE_TOKEN`). |
| `sync-from-public.yml` | daily cron + manual | Mirrors Zone-1 public -> hub as an auto-PR. |
| `sync-service-plugins.yml` | daily cron + manual | Mirrors public service plugin repos -> hub `plugins-src/`. |
| `holone-rules-sync.yml` | weekly cron + manual | HoloNe rules sync into the hub as an auto-PR. |
| `deploy.yml` | manual | VDS deploy. |

Workflows that write to the hub skip gracefully (exit 0) when their write
secret is absent — a missing secret never turns a run red.

## Secrets

Configured:

| Secret | Purpose |
|---|---|
| `MONOREPO_READ_TOKEN` | fine-grained PAT: read hub + submodules, post commit statuses, dispatch farm workflows |
| `SUBMODULE_READ_TOKEN` | recursive submodule checkout |
| `RELEASES_PAT` | public `Stitch-Manager` checkout + release uploads there |
| `SERVICE_EXPORT_TOKEN` | push to `stitch-plugin-template` / service export repos |
| `STITCH_SIGNING_KEY`, `STITCH_PUBLISH_URL`, `STITCH_ADMIN_KEY` | plugin signing + distribution server |
| `VDS_SSH_KEY` | deploy |

Must be added for the write-back pipelines to activate (until then they skip):

| Secret | Purpose |
|---|---|
| `MONOREPO_WRITE_TOKEN` | fine-grained PAT on the hub: `contents: write` + `pull-requests: write` (sync auto-PRs, auto-release tags, holone sync) |
| `ARTIFACT_PRIVATE_KEY` | encrypted artifact publishing (`publish-compiled-plugins.yml`) |

## Trust model

Write access to this repo equals access to its secrets, and the read token
equals the hub's source. Grant write here only to people who already have
write on the hub. Fork PRs never receive secrets (no `pull_request` triggers
exist); all triggers are `schedule` / `workflow_dispatch`.
