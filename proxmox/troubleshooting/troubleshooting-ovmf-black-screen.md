# Black Screen in Proxmox Console After Adding GPU Passthrough (OVMF)

## Problem

A VM that booted and displayed normally suddenly shows a black screen with only a blinking cursor in the Proxmox console after adding GPU passthrough (`hostpci` lines). The VM appears to be running (no crash, no error on start), but no console output is visible. `qm start` succeeds, but the noVNC/SPICE console stays black.

Tried and did not fix it:
- Changing `vga:` between `qxl`, `virtio`, `none`
- Adding `args: -device qxl-vga` (this can even prevent the VM from starting at all, with `kvm: -device qxl-vga: primary qxl-vga device must be console 0`)
- Adding `x-vga=0` to the hostpci line
- Reinstalling the guest OS

## Root cause

The VM's BIOS is set to OVMF (UEFI). OVMF combined with a passthrough GPU and a virtual display device is unreliable — it does not produce a working framebuffer for the Proxmox console, regardless of which virtual display backend is configured.

## Fix

Switch the VM's BIOS from OVMF to SeaBIOS (Legacy), and remove the EFI disk.

```
VM → Options → BIOS → SeaBIOS
VM → Hardware → EFI Disk → Remove
```

Set the virtual display to `vga: virtio` (or `std`).

Important: a guest OS that was installed under OVMF/UEFI cannot boot under SeaBIOS — the installation needs to be redone with SeaBIOS already active. During reinstallation, the installer should now offer to install GRUB to the MBR; if that prompt doesn't appear, the VM is still in UEFI mode and the BIOS setting hasn't taken effect.

## Why this isn't just a display preference

This isn't a cosmetic choice between BIOS types — for a VM doing GPU passthrough where the GPU itself is not used as the display (i.e. it's dedicated to compute, as with an LLM inference VM), OVMF's UEFI display handling and the passthrough GPU appear to conflict in a way that SeaBIOS's simpler legacy display path avoids entirely.
