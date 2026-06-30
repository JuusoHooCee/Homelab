# Docker Host (LXC Container)

This LXC container hosts Docker for lightweight services (Portainer, AdGuard Home). It is separate from the Debian GPU VM, which runs Ollama natively.

## Container setup

- Base: Debian 12 standard template, downloaded via Proxmox UI (Node → local storage → CT Templates)
- Resources used: 2 cores, 2048–4096 MB RAM, 16–32GB root disk, DHCP networking

### Required LXC features

Docker will not function correctly inside an LXC container without these enabled:

- **Nesting** — required for running containers inside the container
- **FUSE** — required by storage drivers / overlay filesystems
- **Keyctl** — required for kernel keyring operations Docker uses

Set via Proxmox GUI: CT → Options → Features → check Nesting, FUSE, Keyctl.

Or directly in the container's config file (`/etc/pve/lxc/<ID>.conf`):

```
features: keyctl=1,nesting=1,fuse=1
```

```bash
pct stop <ID>
pct start <ID>
```

Verify inside the container:

```bash
lsmod | grep fuse      # modprobe fuse if empty
keyctl show            # should not error
```

## Docker installation

```bash
apt update && apt upgrade -y
apt install ca-certificates curl gnupg -y
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" > /etc/apt/sources.list.d/docker.list
apt update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
docker run hello-world
```

If apt reports a GPG signature error (`NO_PUBKEY`) on the Docker repo, the key wasn't saved in proper binary form. Re-run:

```bash
rm -f /etc/apt/keyrings/docker.gpg
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
apt update
```

The critical detail: pipe curl's output through `gpg --dearmor` rather than saving the raw download directly as if it were already a keyring file. This exact error has come up more than once across this homelab — always fixed the same way.

## Important: this LXC actually runs Podman, not Docker

Despite installing via the `docker-ce` packages above, this container's runtime turned out to actually be **Podman** underneath (Docker-API-compatible, which is why `docker` commands and Portainer worked at all) — confirmed when a later `docker ps` failed with `sh: docker: not found`, despite everything having seemingly worked up to that point.

This matters because **Podman has incomplete Docker Compose compatibility**. Simple single-container services (like AdGuard Home) deploy fine through Portainer here. More complex multi-feature stacks (Open WebUI, in this case) can get stuck indefinitely at "deployment in progress" with no clear error — not because of a config mistake, but because Podman doesn't support everything a Compose file might use.

**Practical takeaway:** verify early with `docker ps` / `docker --version` whether this LXC is genuinely running Docker or Podman before spending time debugging a stuck stack. If a stack hangs indefinitely with no error, suspect Podman compatibility before assuming the Compose file is wrong. For anything beyond simple services, consider deploying on a VM with genuine Docker instead (see `debian-vm/` for how Open WebUI was eventually run there instead).
