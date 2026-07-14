# NVIDIA RTX PRO 5000 "Blackwell" GPU Passthrough on Proxmox VE 9 — Three Bugs, Three Fixes

*Fixing the `kvm: segfault at 40` on VM start, the guest `NVRM: BAR0 is 0M`, and `RmInitAdapter failed (0x22:0x56:897)` — all on one card, in sequence.*

## TL;DR

Passing an **NVIDIA RTX PRO 5000 Blackwell (48 GB, PCI `10de:2bb3`)** through to a VM on Proxmox VE 9 failed three times in a row, each with a different root cause. If you are hitting any of these on a Blackwell card (RTX PRO 5000/6000, RTX 50-series), you probably need **all three** fixes:

1. **QEMU segfaults on VM start** (`kvm: segfault at 40`, NULL deref right after "reset done") → QEMU wrongly applies the legacy **GeForce BAR0 quirk** to a professional card. Fix: `x-no-geforce-quirks=on` on the `vfio-pci` device.
2. **Guest boots but GPU is dead** (`NVRM: BAR0 is 0M @ 0x0`, `nvidia-smi: No devices were found`) → the card's 64 MB 32-bit **BAR0 can't be assigned** because the (eGPU/OCuLink) PCIe root port only exposes a 1 MB 32-bit window. Fix: `pci=realloc` on the host kernel cmdline.
3. **Driver loads but won't init** (`NVRM: requires use of the NVIDIA open kernel modules`, `RmInitAdapter failed! (0x22:0x56:897)`) → Blackwell requires the **open** kernel modules, not the proprietary ones. Fix: install the `-open` DKMS variant.

## Environment

| | |
|---|---|
| Host | Mini-PC (AMD Ryzen AI 9 HX PRO 370, "Strix Point"), GPU attached via **OCuLink eGPU dock** |
| Hypervisor | Proxmox VE 9.0.3, kernel **6.17.9-1-pve**, `pve-qemu-kvm` **11.0.2-1** |
| IOMMU | `amd_iommu=on` (note: **no** `iommu=pt`) |
| GPU | NVIDIA RTX PRO 5000 Blackwell 48 GB — `c7:00.0` (`10de:2bb3`) + `c7:00.1` audio (`10de:22e8`) |
| Guest | Ubuntu 24.04, NVIDIA driver 580.159.03 |
| Passthrough | `hostpci0: 0000:c7:00,pcie=1`, efidisk with `pre-enrolled-keys=0` (Secure Boot off) |

The exact same config worked fine with an RTX 4080 in the same slot. Everything below is Blackwell-specific.

---

## Bug 1 — QEMU segfaults on VM start (`kvm: segfault at 40`)

### Symptom
```
# qm start 120
kvm: ... segfault at 40 ip ... sp ... error 4 in qemu-system-x86_64
```
The host `dmesg` shows a NULL-pointer dereference in the KVM/QEMU process, occurring **immediately after** the VFIO reset ("reset done"). Reproduces identically across `pve-qemu-kvm` **10.0.2, 10.2.1 and 11.0.2**, and on kernel 6.17.9. The card itself binds and resets cleanly at the kernel/vfio-pci level — this is purely a QEMU crash.

### Dead ends (so you don't waste time)
- The warning `VFIO dma-buf not supported in kernel: PCI BAR IOMMU mappings may fail` is a **red herring** — the segfault happens with or without it.
- **BAR1 size is not the cause.** The card exposes a 64 GB prefetchable BAR1 (Resizable BAR). Shrinking it to 256 MB (`echo 8 > /sys/bus/pci/devices/0000:c7:00.0/resource1_resize`, after unbinding from vfio-pci) did **not** stop the crash.
- The `szoran` kernel patch for `Firmware has requested this device have a 1:1 IOMMU mapping, rejecting…` does **not** apply here — that dmesg rejection never appears, and our cmdline has no `iommu=pt`.

### Root cause
A gdb backtrace on a minimal QEMU reproducer pins the crash to:
```
vfio_probe_nvidia_bar0_quirk   (hw/vfio/pci-quirks.c:928)
  → memory_region_add_subregion  with a NULL `mirror` MemoryRegion
```
QEMU applies its **legacy GeForce BAR0 quirk** to the card and dereferences NULL. The quirk is meant for consumer GeForce cards; on a professional Blackwell card the code path hits a NULL region instead of degrading gracefully.

### Fix
Disable the GeForce quirks on the `vfio-pci` device. The QEMU property description literally reads *"for NVIDIA Quadro/GRID/Tesla"* — i.e. professional cards:
```
-device vfio-pci,host=0000:c7:00.0,...,x-no-geforce-quirks=on
```
The minimal reproducer exits 0 (instead of 139) with this flag, and the VM starts for the first time.

**Making it persistent in Proxmox** — Proxmox builds the QEMU command line itself, so patch the generator. In `/usr/share/perl5/PVE/QemuServer/PCI.pm`, inside the primary-function block (`if ($j == 0)`, around line 715), append the property:
```perl
$devicestr .= ',x-no-geforce-quirks=on';
```
Back up the file first. Verify it took effect:
```
qm showcmd <vmid> | grep geforce
```
> ⚠️ **This patch does not survive an `apt` upgrade of `qemu-server`.** Re-apply it after any Proxmox package update that touches `PCI.pm`. (A cleaner long-term option is a systemd hook or waiting for a Proxmox/QEMU release that guards this quirk for professional cards.)

---

## Bug 2 — Guest sees the GPU but it's dead (`NVRM: BAR0 is 0M`)

### Symptom
VM now boots. In the guest the card is visible (`lspci` shows `01:00.0 10de:2bb3`) and the 580 driver loads, but:
```
NVRM: GPU 0000:01:00.0: BAR0 is 0M @ 0x0 (PCI:0000:01:00.0)
NVRM: The system BIOS may have misconfigured your GPU.
nvidia-smi: No devices were found
```
On the **host**, the boot log reveals the real problem:
```
pci 0000:c7:00.0: BAR 0 [mem size 0x04000000]: can't assign; no space
pci 0000:c7:00.0: BAR 0 [mem size 0x04000000]: failed to assign
...
pci 0000:03.1: bridge window [mem 0xdd200000-0xdd2fffff 64bit] ... [size=1M]
```

### Root cause
BAR0 is **64 MB, 32-bit non-prefetchable**, so it must live below 4 GB. The PCIe root port the eGPU/OCuLink hangs off of (`00:03.1` here) only advertises a **1 MB** 32-bit memory window. 64 MB doesn't fit → BAR0 gets size 0 → vfio-pci can't pass it through → the guest sees no Region 0 → `BAR0 is 0M`.

Hotplug remove + rescan does **not** help (the window isn't enlarged without reallocation). BAR1/ReBAR resizing is irrelevant here — BAR0 is a separate, 32-bit BAR.

### Fix
Let the kernel reallocate PCI bridge windows so the 1 MB window can grow to ≥64 MB. Add `pci=realloc` to the host kernel cmdline in `/etc/default/grub`:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on pci=realloc"
```
```
update-grub && reboot
```
Verify after reboot:
```
cat /proc/cmdline | grep pci=realloc
dmesg | grep 'c7:00.0.*BAR 0'      # must now say "assigned", e.g. 0x80000000-0x83ffffff
cat /sys/bus/pci/devices/0000:c7:00.0/resource | head -1   # no longer 0x0 0x0 0x0
```
> If `pci=realloc` alone isn't enough, try adding `pci=nocrs` (ignores ACPI `_CRS` window limits — slightly riskier), and check **Above 4G Decoding / Resizable BAR** in the host BIOS.

---

## Bug 3 — Driver loads but won't initialize (open kernel modules required)

### Symptom
BAR0 is now assigned and passed through, but the guest still fails:
```
NVRM: The NVIDIA GPU 0000:01:00.0 (PCI ID: 10de:2bb3)
NVRM: installed in this system requires use of the NVIDIA open kernel modules.
NVRM: GPU 0000:01:00.0: RmInitAdapter failed! (0x22:0x56:897)
NVRM: GPU 0000:01:00.0: rm_init_adapter failed, device minor number 0
```

### Root cause
Blackwell (and newer) GPUs are **only supported by the NVIDIA open kernel modules**. The proprietary/closed kernel module (`license: NVIDIA`) will load but cannot initialize the card. You need the `-open` build (`license: Dual MIT/GPL`).

### Fix (Ubuntu 24.04, packaged driver)
Swap the closed DKMS module for the open one. This removes the closed kernel packages but keeps the 580 userspace libraries:
```
apt install nvidia-dkms-580-server-open
```
Then reload the stack **without rebooting** the VM (nothing is using the GPU yet, since it never initialized):
```
systemctl stop nvidia-persistenced
rmmod nvidia_drm nvidia_uvm nvidia_modeset nvidia
modprobe nvidia
systemctl start nvidia-persistenced
```
Verify the open module is loaded:
```
modinfo nvidia | grep license      # => license: Dual MIT/GPL   (open)  — NOT "NVIDIA" (closed)
nvidia-smi                         # => NVIDIA RTX PRO 5000 Blackwell, 48 GB
```
> If you install the driver via the `.run` installer instead, use `--kernel-module-type=open`. Make sure future DKMS rebuilds/driver upgrades keep the **open** flavor.

---

## Final verification

```
$ nvidia-smi
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.159.03      Driver Version: 580.159.03      CUDA Version: 13.0            |
|  0  NVIDIA RTX PRO 5000 Blac...   |   00000000:01:00.0 Off |   0MiB / 48935MiB |  ...    |
+-----------------------------------------------------------------------------------------+
```

## Order of operations (checklist)

1. Patch `PCI.pm` with `x-no-geforce-quirks=on` → VM starts without segfault.
2. Add `pci=realloc` to GRUB, `update-grub`, **reboot host** → BAR0 gets assigned.
3. In the guest: install `nvidia-dkms-<ver>-server-open`, reload modules → `nvidia-smi` works.

Bugs 1 and 2 are host-side; bug 3 is guest-side. Bug 2 requires a host reboot; the others don't.

## Notes / caveats

- The `PCI.pm` patch is fragile across Proxmox package upgrades — re-check `qm showcmd` after updates.
- `pre-enrolled-keys=0` (Secure Boot off in the VM's OVMF) avoids the separate MOK-signing rabbit hole for the DKMS module.
- Tested with `pve-qemu-kvm 11.0.2-1` / kernel `6.17.9-1-pve`. Newer Proxmox/QEMU may guard the GeForce quirk upstream, making fix #1 unnecessary — check before patching.

## References
- Proxmox forum thread #181195 (RTX 5080 Blackwell, PVE9 / kernel 6.17)
- `github.com/szoran53/proxmox-kernel-6.17-gpu-passthrough` (a *different* Blackwell issue — the 1:1 IOMMU reject — not the one fixed here)
- NVIDIA Developer Forum #339038 (PRO 6000 / 5090 host crash reports)
- QEMU source: `hw/vfio/pci-quirks.c` — `vfio_probe_nvidia_bar0_quirk`

---
*Written up after getting an RTX PRO 5000 Blackwell fully operational under Proxmox VE 9 in an OCuLink eGPU setup, 2026-07-14.*
