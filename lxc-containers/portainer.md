# Portainer

Web UI for managing Docker containers on the docker-host LXC. See `lxc-containers/docker-host.md` for prerequisites.

## Installation

```bash
docker volume create portainer_data
docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access at `http://<lxc-ip>:9000`. First visit prompts creation of the admin account.

## Deploying stacks

Portainer → Stacks → Add Stack lets you paste a Docker Compose file directly and deploy it without using the command line. This works well for simple services (see `lxc-containers/adguard.md`).

Note: this LXC's container runtime is actually Podman under the hood (see `docker-host.md`), which has incomplete Compose compatibility. If a stack deploy hangs at "deployment in progress" indefinitely, this is the most likely cause rather than a mistake in the Compose file itself.

## Password recovery

If the admin password is lost, see `lxc-containers/troubleshooting/portainer-password-reset.md` for the BoltDB hash-replacement procedure.
