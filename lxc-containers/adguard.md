# AdGuard Home

Network-wide DNS filtering, deployed as a Portainer stack on the docker-host LXC. See `lxc-containers/docker-host.md` for prerequisites.

## Why AdGuard Home instead of Pi-hole

Both serve the same role and would conflict over port 53 and the DNS/DHCP role on the network if run together. AdGuard Home was chosen as the sole DNS filtering solution — this was a deliberate choice, not an oversight; Pi-hole was not installed.

## Deployment

Deployed via Portainer → Stacks → Add Stack:

```yaml
version: '3.8'

services:
  adguard:
    image: adguard/adguardhome:latest
    container_name: adguard
    restart: unless-stopped
    ports:
      - "3000:3000"   # Setup UI
      - "53:53/tcp"
      - "53:53/udp"
      - "80:80/tcp"
      - "443:443/tcp"
    volumes:
      - adguard_work:/opt/adguardhome/work
      - adguard_conf:/opt/adguardhome/conf

volumes:
  adguard_work:
  adguard_conf:
```

## Initial setup

1. Open `http://<lxc-ip>:3000`
2. Choose the network interface (eth0)
3. Accept the default DNS ports
4. Create an admin user

After setup completes, the main interface moves to port 80 (`http://<lxc-ip>/`).

## Pointing the home network to AdGuard

Find the LXC's IP:

```bash
pct exec <ID> -- ip a
```

Set this IP as the Primary (and Secondary) DNS server in the router/modem's WAN or LAN/DHCP settings. Verify it's working by checking AdGuard's Dashboard → Queries for incoming DNS logs.

Optional hardening (not necessarily implemented): blocking outbound port 53 on the router except from the AdGuard host's IP, to prevent devices bypassing it via a hardcoded DNS server (some Android devices, Chromecast, and smart TVs default to a public DNS like 8.8.8.8 regardless of DHCP settings).

## Password recovery

If locked out, the admin password can be reset by editing the config file directly inside the container:

```bash
docker exec -it adguard sh
cat /opt/adguardhome/conf/AdGuardHome.yaml
```

Check the `users:` section for the configured username first — a wrong remembered *username* can look identical to a wrong password. If an actual password reset is needed, remove the `password:` line under the relevant user entry in that file, then:

```bash
docker restart adguard
```

This forces AdGuard to prompt for a new password on next web access.

## Editor availability note

The AdGuard container (and minimal Docker images generally) may lack `nano`. `vi` is more often present by default. Quick reference: `i` to enter insert mode, `Esc` then `:wq` to save and quit, `Esc` then `:q!` to quit without saving.
