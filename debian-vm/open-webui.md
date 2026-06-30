# Open WebUI

Web interface for chatting with Ollama models, providing a ChatGPT-style UI instead of the terminal.

## Where this runs

Open WebUI runs on the **Debian GPU VM, alongside Ollama** — not on the docker-host LXC. This is a deliberate choice, not a workaround placed here by accident: the docker-host LXC actually runs Podman under the hood (see `lxc-containers/docker-host.md`), which has incomplete Docker Compose compatibility. Open WebUI's stack got stuck indefinitely at "deployment in progress" there, while simpler services like AdGuard Home worked fine. Running it directly alongside Ollama also avoids any network hop between the UI and the inference engine.

Prerequisite: Docker installed on the Debian VM — see `debian-vm/setup.md`.

## Deployment

```bash
mkdir ~/openwebui
cd ~/openwebui
nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  open-webui:
    image: ghcr.io/open-webui/open-webui:latest
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "3001:8080"
    volumes:
      - openwebui_data:/app/backend/data

volumes:
  openwebui_data:
```

```bash
docker compose up -d
```

Access at `http://<debian-vm-ip>:3001`.

### "Unhealthy" status on first start

`docker ps` may show the container as `Up X minutes (unhealthy)` right after starting. This is normal — the backend takes a few minutes to fully initialize on first boot, during which the healthcheck fails. It resolves on its own; no action needed. If it doesn't resolve after several minutes, a custom healthcheck with a longer grace period can help diagnose further:

```yaml
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 20s
```

## Connecting to Ollama

In Open WebUI: **Settings → Models → Add Ollama Backend**, then enter Ollama's API URL.

### Important: `host.docker.internal` does not work on Linux

This is the conventional Docker hostname for a container to reach services on its host — but it only resolves automatically on Docker Desktop (Windows/macOS). On native Linux Docker Engine, it does not resolve at all, producing errors like "cannot connect to host.docker.internal:11434" / "failed to fetch models."

**Correct approach on Linux:** use the host machine's actual LAN IP address, found via `ip a` on the Debian VM itself — not `localhost`, not `127.0.0.1`, not `host.docker.internal`.

```
http://<debian-vm-lan-ip>:11434
```

This also requires Ollama to actually be listening on that LAN IP rather than only `127.0.0.1` — see `debian-vm/ollama.md` for the `OLLAMA_HOST=0.0.0.0` systemd override needed to make that true. Both fixes are required together: binding Ollama to the right address, and using that address (not a Docker-specific hostname) in Open WebUI.

### Diagnostic commands

```bash
ip a                                          # find the VM's actual LAN IP
curl http://localhost:11434/api/tags          # confirm Ollama responds locally
curl http://<vm-lan-ip>:11434/api/tags        # the actual test that needs to pass
ss -tulpn | grep 11434                        # confirm what address Ollama is bound to
```

If `curl` to `0.0.0.0` or `localhost` succeeds but the actual LAN IP doesn't, check `ss -tulpn` to see the literal bound address — don't assume binding to the wildcard address is confirmed just because a loopback-style test passed.

## Known limitations

Only Ollama's default pre-quantized models (e.g. `llama3:8b` at Q4_0) can currently be used through this setup — see `debian-vm/ollama.md` for the Modelfile/`QUANTIZE` limitation that prevents running custom-quantized larger models. A possible future fix using an `llama.cpp` backend instead of Ollama was discussed but not implemented or tested.
