# Sophos NDR on Proxmox VM

This guide documents how to run Sophos NDR as a virtual machine on Proxmox VE with a switch port mirror as the traffic source. Sophos NDR is designed for specific appliances, so running it in a generic QEMU/KVM VM requires several workarounds to get the DPDK-based capture engine (Dragonfly) working.

> This project is not affiliated with or endorsed by Sophos.
> Use at your own risk. Configuration files may break on 
> Sophos NDR software updates.
---
> AI helped me create this ReadMe
---

## Repository Files

| File | Description |
|------|-------------|
| `README.md` | This guide |
| `vfio-noiommu.service` | Systemd service for VFIO noiommu mode and NIC binding |
| `proxmox-span-interfaces` | Proxmox SPAN bridge config snippet for `/etc/network/interfaces` |
| `platform_profiles_entry.json` | Profile entry to add to `/etc/dragonfly/platform_profiles.json` |

---

## Architecture Overview

```
Core Switch
    │
    │ Port Mirror (SPAN)
    ▼
Proxmox Host (nic0)
    │
    │ SPAN Bridge (MAC learning disabled, flooding enabled)
    ▼
Sophos NDR VM (span0 → vfio-pci → DPDK → Dragonfly)
```

---

## Prerequisites

- Proxmox VE 7.x or 8.x
- Sophos NDR ISO (Hardware option) installed as a VM
- Switch with port mirroring capability
- A dedicated NIC on the Proxmox host for the SPAN mirror destination

---

## Part 1 — Proxmox VM Configuration

### VM Settings

Configure the VM with these settings. The most critical point is **do not enable viommu** — it creates real IOMMU groups that put both NICs in the same group, which breaks VFIO.

| Setting | Value |
|---------|-------|
| Machine | `q35` (NOT q35 + viommu=intel) |
| BIOS | OVMF (UEFI) |
| CPU | host*, flags=+pdpe1gb | 8 cores
*Host CPU isnt ideal, working on a fix for this
| RAM | 32GB minimum |
| DISK | 256GB minimum |
| SCSI Controller | virtio-scsi-single |
| Balloon | Disabled |
| NIC 0 | VirtIO, Management bridge |
| NIC 1 | VirtIO, SPAN bridge |

Example `/etc/pve/qemu-server/<vmid>.conf`:

```
balloon: 0
bios: ovmf
cores: 8
cpu: host,flags=+pdpe1gb
machine: q35
memory: 32000
net0: virtio=BC:24:11:XX:XX:XX,bridge=Management
net1: virtio=BC:24:11:XX:XX:XX,bridge=SPAN
numa: 0
ostype: l26
sata0: <storage>:vm-<vmid>-disk-2,size=256G,ssd=1
scsihw: virtio-scsi-single
sockets: 1
```

> **Note:** Proxmox may not accept `pcie=1` for network devices via the GUI or API. The standard VirtIO NIC without pcie=1 works because viommu is disabled, which means there are no real IOMMU groups to conflict.

---

## Part 2 — Proxmox Host SPAN Bridge

The mirror destination port connects to a physical NIC on the Proxmox host. This NIC is added to a dedicated bridge for the Sophos NDR VM.

### Why Standard Bridge Settings Don't Work

A Linux bridge learns MAC addresses and only forwards unicast frames to the port where it believes the destination MAC lives. With SPAN/mirror traffic, the bridge learns hundreds of MACs on the physical NIC port and never forwards unicast to the VM tap interface. Only broadcast and multicast make it through.

The fix is to disable MAC learning on the physical NIC and enable flooding on both ports so all traffic is forwarded to the VM regardless of destination MAC.

### Configuration

Add to `/etc/network/interfaces` on the Proxmox host. Replace `nic0` with your actual NIC name and `tap116i1` with your VM's tap interface (format: `tap<vmid>i<nic_index>`):

```
auto SPAN
iface SPAN inet manual
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
    post-up ip link set SPAN promisc on
    post-up ip link set nic0 promisc on
    post-up bridge link set dev nic0 learning off flood on
    post-up bridge link set dev tap<vmid>i1 flood on
```

Apply without rebooting:

```bash
ifdown SPAN && ifup SPAN
bridge link set dev nic0 learning off flood on
bridge link set dev tap<vmid>i1 flood on
```

Verify traffic is reaching the VM tap:

```bash
tcpdump -i tap<vmid>i1 -n -c 50 --immediate-mode
```

---

## Part 3 —  Port Mirror

In switch config, go to **Port Mirroring** and configure:

- **Source:** The uplink or trunk port carrying all VLAN traffic
- **Destination:** The switch port connected to `nic0` on the Proxmox host


> **Note:** my switch strips VLAN 802.1Q tags on mirrored traffic. All traffic arrives on the SPAN bridge untagged regardless of source VLAN. Sophos NDR still sees traffic from all VLANs, the different IP subnets in the flow data confirm multi-VLAN capture.

---

## Part 4 — Hugepages (Inside Sophos NDR VM)

Sophos NDR's Dragonfly DPDK engine requires 1GB hugepages. These must be allocated at boot time.

Edit `/etc/default/grub` inside the VM:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet hugepagesz=1G hugepages=8"
```

Regenerate grub:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
# or
grub-mkconfig -o /boot/grub/grub.cfg
```

Reboot and verify:

```bash
grep -i huge /proc/meminfo
```

Expected output:

```
HugePages_Total:       8
HugePages_Free:        8
Hugepagesize:    1048576 kB
```

If Kubernetes shows `insufficient hugepages-1Gi`, restart kubelet after hugepages are allocated:

```bash
systemctl restart kubelet
```

---

## Part 5 — VFIO noiommu Mode and NIC Binding (Inside Sophos NDR VM)

### Why This Is Needed

DPDK requires exclusive control of the capture NIC. It does this by binding the NIC to the `vfio-pci` kernel driver, removing it from kernel network stack control. Without a real IOMMU (viommu is disabled), VFIO requires `noiommu` mode to function.

### Critical Ordering Requirement

`noiommu` mode **must be enabled before** `vfio-pci` binds the device. If the device is bound first, a regular VFIO group is created instead of a noiommu group, and Dragonfly fails to open it with `Device or resource busy`.

### Systemd Service

Copy `vfio-noiommu.service` to `/etc/systemd/system/` and enable it:

```bash
sudo cp vfio-noiommu.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable vfio-noiommu.service
```

The service:
1. Enables noiommu mode on the vfio module
2. Loads the vfio-pci driver
3. Registers the VirtIO NIC device ID (`1af4:1000`) with vfio-pci

This automatically claims any unbound VirtIO NIC (the SPAN capture NIC, `span0`) while leaving the management NIC (`mgmt0`) bound to the kernel `virtio-pci` driver.

Verify after reboot:

```bash
cat /sys/module/vfio/parameters/enable_unsafe_noiommu_mode
# Should show: Y

ls -la /sys/bus/pci/devices/0000:06:13.0/driver
# Should show: -> .../vfio-pci

ip link show mgmt0
# Should show mgmt0 as UP
```

> **Note:** Your capture NIC PCI address may differ from `0000:06:13.0`. Find it with `lspci | grep -i virtio`.

---

## Part 6 — Dragonfly Configuration (Inside Sophos NDR VM)

### The Problem

Sophos NDR's hardware detection uses DMI product name to match a hardware profile in `/etc/dragonfly/platform_profiles.json`. The supported profiles are specific Intel NUC models. A QEMU VM identifies as `Standard PC (Q35 + ICH9, 2009)` which matches nothing, so the `##AUTO##` interface detection never resolves and Dragonfly reports `no capture interfaces found`.

### Fix 1 — Add VM Hardware Profile

Edit `/etc/dragonfly/platform_profiles.json` and add the entry from `platform_profiles_entry.json` inside the `systems` array. Make sure to add a comma after the preceding entry.

### Fix 2 — Update Interface Mapping Profile Name

Edit `/etc/dragonfly/interface_mapping.json` and change:

```json
"profile_product_name": "ISO Installer"
```

To:

```json
"profile_product_name": "Standard PC (Q35 + ICH9, 2009)"
```

---

## Part 7 — Verification

Check the Dragonfly pod logs:

```bash
sudo kubectl logs -f $(sudo kubectl get pods | grep dragonfly | awk '{print $1}')
```

A working capture shows:

```
EAL: Selected IOVA mode 'PA'
EAL: VFIO support initialized
EAL: Probe PCI driver: net_virtio (1af4:1000) device: 0000:06:12.0 (socket -1)
eth_virtio_pci_init(): Failed to init PCI device
EAL: Requested device 0000:06:12.0 cannot be used

[Int#0] UP   UnknownInterface   Packets: 10666937   Bytes: 9237477189
[Total] ActFlows: 6337   TotFlows: 86194   Exports: 82092   Drops: 0
```

Key indicators:

| Indicator | Meaning |
|-----------|---------|
| `IOVA mode 'PA'` | noiommu is active |
| Only `0000:06:12.0` in probe errors | Capture NIC is silently owned by DPDK via VFIO |
| Packets/flows/exports increasing | Traffic is being captured and analyzed |
| `Drops: 0` | No packet loss |

The `UnknownInterface` label is normal — the interface name cannot be resolved by the kernel since DPDK owns it exclusively.

Also verify from the Proxmox host that traffic is flowing to the tap:

```bash
tcpdump -i tap<vmid>i1 -n -c 100 --immediate-mode
```

You should see traffic from multiple IP subnets confirming all VLANs are being captured.

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `VFIO group is not viable` | viommu enabled, both NICs in same IOMMU group | Remove `viommu=intel` from Proxmox machine config |
| `Cannot open /dev/vfio/noiommu-0: Device or resource busy` | vfio-pci bound before noiommu enabled | Use the systemd service to ensure correct ordering |
| `no capture interfaces found` | Missing hardware profile or interface mapping | Add platform profile and update profile_product_name |
| `insufficient hugepages-1Gi` | 1GB hugepages not allocated | Add hugepage kernel params and restart kubelet |
| `unsupported hugepagesz 1gb` | CPU doesn't expose pdpe1gb flag | Ensure `cpu: host,flags=+pdpe1gb` in Proxmox config |
| Only native VLAN traffic visible | MAC learning not disabled on SPAN bridge | Apply `bridge link set dev nic0 learning off flood on` |

---

## Important Notes

- Sophos NDR is not officially supported in a generic VM. This guide makes it work with no guarantees from Sophos.
- Changes to `/etc/dragonfly/platform_profiles.json` and `interface_mapping.json` may be overwritten by Sophos NDR software updates and will need to be reapplied.
- The `echo "1af4 1000"` command registers ALL VirtIO NICs with vfio-pci. Unbound VirtIO devices will be claimed. The management NIC stays safe because it is already bound to `virtio-pci` when the command runs.
