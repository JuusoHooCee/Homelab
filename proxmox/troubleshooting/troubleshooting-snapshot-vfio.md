# Snapshots Fail on VMs with GPU Passthrough

## Problem

Taking a Proxmox snapshot of a VM that has GPU passthrough configured fails:

```
TASK ERROR: VM 100 qmp command 'savevm-start' failed - 0000:01:00.1: VFIO migration is not supported in kernel
```

## Root cause

This is a kernel-level VFIO limitation, not a Proxmox bug or a misconfiguration. VFIO-assigned PCI devices (like a passthrough GPU) cannot be included in a live snapshot's saved state, because the kernel has no way to migrate/serialize the device's state. This affects any hypervisor using VFIO passthrough, not just Proxmox.

## Fix

There isn't one — snapshots are simply not available for VMs with active `hostpci` passthrough devices. Use Proxmox's Backup feature instead, with mode set to **Stop**:

```
VM → Backup → Backup Now → Mode: Stop
```

A stop-mode backup shuts the VM down, copies the full disk state, and restarts it — it doesn't rely on VFIO state migration the way a live snapshot would, so it works regardless of passthrough devices.

## Side note: storage space

Snapshot attempts may also separately fail or warn about thin pool space exhaustion if the `pve/data` thin pool doesn't have enough free space relative to allocated thin volumes. This is a different, storage-capacity issue and unrelated to the VFIO limitation above — but both can surface around the same time if storage is already tight. If backups/snapshots are a priority, plan for dedicated backup storage (e.g. a separate HDD) rather than relying on the same thin pool used for VM disks.
