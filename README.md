# Open WebUI + Ollama

Docker Compose stack that runs [Ollama](https://ollama.com) (local LLM server) with [Open WebUI](https://github.com/open-webui/open-webui) (chat UI), optionally fronted by Apache and exposed through a Tailscale Funnel.

## Contents

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Defines the services, network and bind mounts |
| `docker-compose.sh` | Wrapper that creates the `data/` directories, then runs `docker compose` with your arguments |
| `.env.sample` | Template for the required `.env` settings |
| `httpd-open-webui.conf` | Apache reverse-proxy vhost (installed to `/etc/httpd/conf.d/`) |

## Services

- **ollama**: serves models on port `11434`. Uses the AMD iGPU through `/dev/dri` (Vulkan), keeps models loaded for 30 minutes, with a 2048-token context. Memory is limited to 24 GB.
- **ollama-pull-qwen**: one-shot job that pulls `qwen2.5:3b` and `qwen2.5-coder:7b` once Ollama is healthy. Add more `ollama pull` lines to its `command` to fetch other models.
- **open-webui**: the web UI on port `3000`. Starts after the model pull succeeds. `qwen2.5:3b` is the default model and `qwen2.5-coder:7b` is pinned.

## Setup

1. Create your environment file and fill in the admin credentials:

   ```sh
   cp .env.sample .env
   ```

   `WEBUI_ADMIN_EMAIL` and `WEBUI_ADMIN_PASSWORD` are required; Compose refuses to start without them. They only apply on the first start, before any user exists.

2. Start the stack with the wrapper, which creates the data directories first:

   ```sh
   ./docker-compose.sh up --detach
   ```

3. Open <http://localhost:3000> and sign in with the admin account.

The first start is slow because the models are downloaded.

## Everyday commands

The wrapper passes its arguments straight to `docker compose`:

```sh
./docker-compose.sh ps                  # status and health
./docker-compose.sh logs -f ollama      # follow logs
./docker-compose.sh pull                # fetch newer images
./docker-compose.sh up --detach         # apply changes
./docker-compose.sh down                # stop and remove containers (data is kept)
docker exec ollama ollama ps            # loaded models and whether they run on GPU or CPU
```

## Data

Everything persistent lives under `data/` (git-ignored):

- `data/ollama/`: downloaded models
- `data/open-webui/`: Open WebUI database, uploads and settings

Use `./docker-compose.sh` rather than plain `docker compose` the first time, because Ollama fails to start if `data/ollama/models` doesn't exist.

## Configuration

Edit the `environment:` blocks in `docker-compose.yml`:

- `WEBUI_URL` and `CORS_ALLOW_ORIGIN`: must include every URL users browse from, including the Tailscale Funnel URL.
- `DEFAULT_MODELS`, `DEFAULT_PINNED_MODELS`: Open WebUI defaults. Open WebUI generally applies these on first start; change them later in its admin settings.
- `pull_policy`: set to `always` to refresh images on every `up`, or `missing` to pull only when absent.

## Reverse proxy (optional)

`httpd-open-webui.conf` is an Apache virtual host listening on port 80 that proxies to Open WebUI on `127.0.0.1:3000`, including WebSocket traffic. Tailscale Funnel can then forward `https://<host>.<tailnet>.ts.net` to `http://127.0.0.1:80`.

```sh
sudo cp httpd-open-webui.conf /etc/httpd/conf.d/
sudo httpd -t && sudo systemctl reload httpd
```

Requires `mod_proxy`, `mod_proxy_http` and `mod_headers`.

## Troubleshooting

- **Ollama is unhealthy**: check `./docker-compose.sh logs ollama`, and that `data/ollama/models` exists and is writable.
- **GPU not used**: the Ollama log should show `library=Vulkan` and your GPU. Confirm `/dev/dri/renderD128` exists and is accessible.
- **Compose error about `WEBUI_ADMIN_*`**: fill in `.env`.
