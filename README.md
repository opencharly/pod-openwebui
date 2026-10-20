# pod-openwebui

The `openwebui` candy of the OpenCharly candy library, as a standalone repo
(kind-prefixed naming). It ships the Open WebUI LLM chat front-end with an
auto-configuring entrypoint, served as a supervised HTTP service on port `8080`.

## What it provides

Installs Open Web UI (the `open-webui` PyPI package, pinned to **0.12.3** in
`pixi.toml`, via pixi) and copies a bash auto-config entrypoint to
`${HOME}/.local/bin/openwebui-entrypoint` (mode 0755). A supervised system service
runs that entrypoint, which wires up LLM providers from env/secrets (OpenRouter,
Ollama Cloud, direct OpenAI, local Ollama, MCP servers, and Jupyter code execution),
enforces mandatory admin credentials (`WEBUI_ADMIN_EMAIL` + `WEBUI_ADMIN_PASSWORD`),
reads the auto-generated `WEBUI_SECRET_KEY`, then execs
`open-webui serve --port $PORT` on port `8080`.

| Property | Value |
|---|---|
| Service | `openwebui` (`%(ENV_HOME)s/.local/bin/openwebui-entrypoint`, `restart: always`) |
| Port | `8080` |
| Requires | `layer-python`, `layer-supervisord` |
| Volume | `data` at `/opt/data` |
| Env | `DATA_DIR=/opt/data`, `PORT=8080`, `ENABLE_CODE_EXECUTION=true`, `ENABLE_DIRECT_CONNECTIONS=true`, … |
| `env_require` | `WEBUI_ADMIN_EMAIL` |
| `secret_require` | `WEBUI_ADMIN_PASSWORD` (credential-backed — supply it with `-e` on `charly config`) |
| `secret_accept` | `OPENROUTER_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_API_KEY` |
| `mcp_accept` | `jupyter`, `chrome-devtools` |
| Alias | `open-webui` |

`WEBUI_ADMIN_PASSWORD` is a required credential, so the `charly config` path will not
proceed without one: with no value it stops on `missing required credential-backed
secret(s)`, and `charly config openwebui -e WEBUI_ADMIN_PASSWORD=<32-byte hex>` imports
it into the credential backend, strips it from the deploy, and injects it as the podman
secret `charly-openwebui-webui-admin-password`. Provider API keys work the same way —
never in plaintext in the deploy.

Reading a provisioned value back is `charly secrets get charly/secret
WEBUI_ADMIN_PASSWORD` **where the `secrets` command plugin is available**; it is not
part of the default binary today (tracked in opencharly/charly#995), so until that
lands the authoritative copy is the credential backend itself
(`~/.config/charly/config.yml` → `keyring_keys: charly/secret/<NAME>`).

Open Web UI has no immich integration, so this candy accepts neither `IMMICH_API_URL` nor
`IMMICH_API_KEY`: they were dead declarations here (nothing in the image read them, and the
`immich` candy self-accepts its own URL), and a compose of the two generates and runs the same
without them. `hermes` remains the real consumer of the immich provide.

`WEBUI_SECRET_KEY` is a credential the image never ships: Open Web UI 0.12.3 refuses
to start without it once authentication is on, so the candy provisions it once per
deployment (podman secret `charly-openwebui-webui-secret-key`) and keeps it stable —
losing it invalidates every session and stored OAuth token. `PORT` is passed to
`open-webui serve` explicitly, because upstream's `serve` takes the listen port as an
argument and does not read the environment.

## How to use it

```bash
charly box build openwebui
charly config openwebui -e WEBUI_ADMIN_EMAIL=you@example.com \
  -e WEBUI_ADMIN_PASSWORD=<32-byte hex> \
  -e OPENROUTER_API_KEY=sk-or-xxx
charly start openwebui
# Open http://localhost:8080
```

Both admin values are mandatory on the `charly config` path — the deploy refuses to
continue without the credential rather than inventing one.

Deploy alongside provider containers for full functionality — Open WebUI consumes
`OLLAMA_HOST` from a deployed `ollama` and the `jupyter` / `chrome-devtools` MCP
servers, then auto-configures `CODE_EXECUTION_ENGINE=jupyter`. The candy's own
`check:` steps assert the entrypoint script, the running `openwebui` service, the
HTTP `200` on the published port, the injected `WEBUI_ADMIN_EMAIL`, and the
reachable port.

## Layout

- `charly.yml` — the `openwebui:` candy entity (description, `require`, `env`,
  `env_require`, `env_accept`, `secret_require`, `secret_accept`, `mcp_accept`,
  `port`, `volume`, `alias`, `secret`, `service`, `plan`) plus its `skill:`
  entity.
- `openwebui-entrypoint` — the auto-config runtime wrapper.
- `pixi.toml` / `pixi.lock` — the Python environment (`open-webui`).
- `.github/workflows/tag-on-merge.yml` — CalVer tag + `CHANGELOG/` on merge.
- `CHANGELOG/` — per-Calver history.
- `README.md` — this user overview.

## Related

- Owning skill: `/charly-openwebui:openwebui` — the auto-config entrypoint,
  secrets, `env_accept`, and the MCP connection format.
- `/charly-build:secrets` — `WEBUI_ADMIN_PASSWORD` and the provider API keys.
- `/charly-core:charly-config` — `-e WEBUI_ADMIN_EMAIL=...` deploy-time env setup.
- `/charly-ollama:ollama` — deploy alongside for local LLM inference.
- [`opencharly/charly`](https://github.com/opencharly/charly) — the charly CLI and image builder
- [`opencharly/opencharly`](https://github.com/opencharly/opencharly) — the umbrella
