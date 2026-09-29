# AGENTS.md — pod-openwebui

Standalone candy repo for the `openwebui` candy — the Open WebUI LLM chat
front-end with an auto-configuring entrypoint, served as a supervised HTTP
service on port `8080`. The candy lives in `charly.yml` at the repo root plus its
pixi environment and entrypoint.

Canonical files:

- `charly.yml` — the `openwebui:` candy entity (description, `require`, `env`,
  `env_require`, `env_accept`, `secret_require`, `secret_accept`, `mcp_accept`,
  `port`, `volume`, `alias`, `secret`, `service`, `plan`) and its `skill:`
  entity.
- `openwebui-entrypoint` — the auto-config runtime wrapper (provider wiring,
  admin credentials, then `open-webui serve`).
- `pixi.toml` / `pixi.lock` — the Python environment (`open-webui`).
- `.github/workflows/tag-on-merge.yml` — CalVer tag + `CHANGELOG/` on merge.
- `CHANGELOG/` — per-CalVer history.
- `README.md` — user overview only; never agent guidance.

## Load these skills first (R0)

- `/charly-openwebui:openwebui` — the owning skill: the auto-config entrypoint,
  the secrets, `env_accept`, and the MCP connection format. Load before editing,
  building, deploying, or troubleshooting this candy.
- `/charly-build:secrets` — the credential store backing the `secret_require` /
  `secret_accept` entries.
- `/charly-pod:pod` — the `kind: pod` / deploy schema reference (this candy is
  composed into a box; tree-position nesting, volumes, ports).
- `/charly-check:check` — the check/R10 framework: the `check:` step verbs and
  `charly check run <bed>`.
- `/charly-image:layer` — the candy authoring reference (`charly.yml` schema,
  `plan:` step verbs, `env_accept` / `secret_accept` / `mcp_accept`, service
  declarations).
- `/charly-internals:git-workflow` — before any git/PR action.

## Build / validate / test

- `charly box validate` at the repo root — the structural check: the manifest
  must parse and validate at the installed charly.
- The live R10 witness is a composing box's `check` bed; the candy's own `check:`
  steps assert the entrypoint script, the running `openwebui` service, the HTTP
  `200` on the published port, the injected `WEBUI_ADMIN_EMAIL`, and the
  reachable port.
- The merge gate is the **org-wide** `charly/pr-validator` (required check
  `validate / validate`, defined in `opencharly/.github`); this repo has **no**
  per-repo candy gate. Its only workflow file is
  `.github/workflows/tag-on-merge.yml`.

## Modify this repo

- Edit the `openwebui:` candy entity in `charly.yml` and the
  `openwebui-entrypoint` wrapper together; the `skill:` entity in the same file
  is the owning skill's source — a candy change and its skill change land
  together.
- `WEBUI_ADMIN_EMAIL` is `env_require` (mandatory); `WEBUI_ADMIN_PASSWORD` is
  `secret_require` (auto-generated on first deploy). Never put credentials in the
  plan — provider keys are `secret_accept` entries injected via podman secrets.
- Python dependency changes belong in `pixi.toml` / `pixi.lock`, not in the plan.
- The `data` volume at `/opt/data`, the `DATA_DIR` / `PORT` env, and the
  `mcp_accept` server names must stay in step with the entrypoint's wiring.
- The `skill:` entity is the source for `/charly-openwebui:openwebui`; never edit
  the generated `SKILL.md` — regenerate it.

## Landing

- PR-only. Every change lands through a pull request; the org-required
  `charly/pr-validator` validates the diff and body and arms native auto-merge on
  PASS. Direct pushes to `main` are blocked.
- History lives in `CHANGELOG/` (written by `tag-on-merge` at merge time); the PR
  body IS the changelog.
- The authoritative rulebook is the umbrella `AGENTS.md` in
  `opencharly/opencharly` and `charly/AGENTS.md` in the charly repo. Do not
  restate its rules here.
