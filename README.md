# pod-openwebui

The `openwebui` candy of the OpenCharly candy library, as a standalone repo
(kind-prefixed naming). It ships the Open WebUI LLM chat front-end with an
auto-configuring entrypoint, served as a supervised HTTP service on port `8080`.

## What it provides

Installs the Open WebUI server (the `open-webui` PyPI package via pixi) and copies
a bash auto-config entrypoint to `${HOME}/.local/bin/openwebui-entrypoint` (mode
0755). A supervised system service runs that entrypoint, which wires up LLM
providers from env/secrets (OpenRouter, Ollama Cloud, direct OpenAI, local
Ollama, MCP servers, and Jupyter code execution), enforces mandatory admin
credentials (`WEBUI_ADMIN_EMAIL` + `WEBUI_ADMIN_PASSWORD`), reads the
auto-generated `WEBUI_SECRET_KEY`, then execs `open-webui serve` on port `8080`.

| Property | Value |
|---|---|
| Service | `openwebui` (`%(ENV_HOME)s/.local/bin/openwebui-entrypoint`, `restart: always`) |
| Port | `8080` |
| Requires | `layer-python`, `layer-supervisord` |
| Volume | `data` at `/opt/data` |
| Env | `DATA_DIR=/opt/data`, `PORT=8080`, `ENABLE_CODE_EXECUTION=true`, `ENABLE_DIRECT_CONNECTIONS=true`, … |
| `env_require` | `WEBUI_ADMIN_EMAIL` |
| `secret_require` | `WEBUI_ADMIN_PASSWORD` (auto-generated on first deploy) |
| `secret_accept` | `OPENROUTER_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_API_KEY`, `IMMICH_API_KEY` |
| `mcp_accept` | `jupyter`, `chrome-devtools` |
| Alias | `open-webui` |

`WEBUI_ADMIN_PASSWORD` is auto-generated on first deploy as a 32-byte hex random
value via the credential store; retrieve it with
`charly secrets get charly/secret WEBUI_ADMIN_PASSWORD`. Provider API keys live in
the credential backend and are injected at runtime via podman secrets — never in
plaintext in the deploy.

## How to use it

```bash
charly box build openwebui
charly config openwebui -e WEBUI_ADMIN_EMAIL=you@example.com -e OPENROUTER_API_KEY=sk-or-xxx
charly start openwebui
# Open http://localhost:8080
```

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
