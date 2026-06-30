# Debian VM Setup (LLM Compute VM)

This VM runs Debian 12 with the GTX 1080 passed through for compute use (see `proxmox/gpu-passthrough.md` for the Proxmox-side configuration). This document covers the guest OS installation and NVIDIA driver setup inside the VM itself.

## Installation

Debian 12 (Bookworm), installed via netinst ISO, with the GNOME desktop environment selected during install (the "Debian desktop environment" + GNOME task).

**Critical:** the VM's BIOS must be set to SeaBIOS before installing, not OVMF — see `proxmox/troubleshooting/ovmf-black-screen.md` for why. During installation, the installer should offer to install the GRUB bootloader to the MBR. If this prompt doesn't appear, the VM is still in UEFI mode and the BIOS setting needs fixing before continuing.

## NVIDIA driver inside the VM

Enable non-free repos in `/etc/apt/sources.list`:

```
deb http://deb.debian.org/debian bookworm main contrib non-free non-free-firmware
deb http://deb.debian.org/debian-security/ bookworm-security main contrib non-free non-free-firmware
deb http://deb.debian.org/debian bookworm-updates main contrib non-free non-free-firmware
```

Install the driver:

```bash
apt update
apt install -y nvidia-driver firmware-misc-nonfree
reboot
```

Note: unlike on the Proxmox host, this apt-based method works fine here — there's no `proxmox-ve` meta-package to conflict with inside a plain Debian guest.

Resulting driver version in this VM: 535.261.03, CUDA 12.2 (this is independent of the host's driver version — they don't need to match).

Verify:

```bash
nvidia-smi
```

Should show the GTX 1080, 8192MiB total, idle power state (P8) when nothing is running.

## Console access note

Once GPU passthrough is active, the SPICE console option in Proxmox no longer works — SPICE requires a virtual display device, which conflicts with/is unavailable once passthrough is configured. Use SSH instead for anything requiring copy/paste:

```bash
ip a                      # find the VM's IP, run inside the VM
ssh user@<vm-ip>          # from the host machine
```

`curl` is not installed by default on a fresh Debian install — install it before using any install script that depends on it:

```bash
apt install curl -y
```

## Docker (if needed for other services on this VM)

Standard official Docker repo method:

```bash
apt install ca-certificates curl gnupg -y
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list
apt update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

If apt reports a GPG signature error (`NO_PUBKEY`) on the Docker repo, the key wasn't saved in proper binary form — re-run the `curl | gpg --dearmor` line above rather than trying to fix it any other way; this exact error recurred multiple times across this project and was always fixed the same way.

Verified working version: Docker 29.5.3.

### A note on the NVIDIA Container Toolkit

If you're tempted to set up the NVIDIA Container Toolkit so Docker containers can access the GPU (`docker run --gpus all`) — it is **not needed** for running Ollama directly on this VM (outside Docker). Ollama uses the host's CUDA libraries and driver directly when run natively. The toolkit is only relevant if you specifically need GPU access from *inside* a Docker container. See `debian-vm/ollama.md` for how Ollama is actually run here (natively, via systemd).
