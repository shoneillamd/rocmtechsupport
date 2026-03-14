# ROCm TechSupport Troubleshooting Guide

**Based on `amddcgpuce/rocmtechsupport` — `rocm_techsupport.sh` V1.41**

---

## Overview

`rocm_techsupport.sh` collects a comprehensive snapshot of your system state for AMD ROCm GPU environments. This guide explains how to collect logs, interpret each section, and diagnose common issues.

---

## Step 1: Collect Logs

### Enable Persistent Boot Logging (do this first)

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald.service
```

Without this, logs from previous boots will not be captured.

### Download the Script

```bash
wget -O rocm_techsupport.sh --no-cache --no-cookies --no-check-certificate \
  https://raw.githubusercontent.com/amddcgpuce/rocmtechsupport/master/rocm_techsupport.sh
```

### Run and Capture Output

```bash
# Standard run (recommended — captures dmidecode, lspci -vvv, etc.)
sudo sh ./rocm_techsupport.sh > $(hostname).$(date +"%y-%m-%d-%H-%M-%S").rocm_techsupport.log 2>&1

# With specific ROCm version path
sudo ROCM_VERSION=/opt/rocm-6.2.0 sh ./rocm_techsupport.sh > $(hostname).$(date +"%y-%m-%d-%H-%M-%S").rocm_techsupport.log 2>&1

# Optional: full journalctl dump
sudo journalctl -b > $(hostname).$(date +"%y-%m-%d-%H-%M-%S").journalctl.log
```

> **Note:** Running without `sudo` is allowed but `dmidecode`, `lspci -vvv`, and some network tools will produce incomplete output.

---

## Step 2: Understand the Log Sections

The log is divided into labeled sections. Use `grep "===== Section:"` to navigate:

```bash
grep "===== Section:" <logfile>
```

| Section | What It Contains |
|---|---|
| OS Distribution | `uname -a`, `/etc/os-release` |
| Kernel Boot Parameters | `/proc/cmdline` — kernel args, iommu settings |
| dmesg GPU/DRM/ATOM/BIOS | Filtered kernel messages for GPU, PCIe, errors, MCE, EDAC |
| CPU Information | `lscpu` — core count, NUMA, architecture |
| Memory Information | `lsmem` — memory ranges and size |
| Hardware Information | `lshw` — full hardware inventory |
| lsmod loaded module | Currently loaded kernel modules |
| amdgpu modinfo | `amdgpu` driver version and parameters |
| dkms status | DKMS-managed kernel module build status |
| amdgpu udev rule | `/etc/udev/rules.d/70-amdgpu.rules` |
| lsinitrd lsinitramfs | Contents of initramfs — confirms amdgpu is included |
| Hardware Topology | `lstopo` — CPU/NUMA/PCIe topology |
| dmidecode Information | BIOS version, system board, memory DIMMs |
| lspci verbose output | PCIe tree and verbose device capabilities |
| ROCm Repo Setup | Configured apt/yum/zypp repos for ROCm/AMDGPU |
| ROCm Packages Installed | Installed ROCm package list |
| ROCm ldconfig entries | `/etc/ld.so.conf.d/` ROCm entries |
| ROCm ldcache entries | Dynamic linker cache for ROCm libs |
| ROCm environment variables | `HSA_*`, `HIP_*`, `ROCM_*`, `MPI_*` env vars |
| Available ROCm versions | Installed ROCm paths under `/opt/rocm*` |
| rocm-bandwidth-test Topology | P2P bandwidth topology between GPUs |
| AMD SMI (various) | GPU list, static info, firmware, bad pages, processes, topology, XGMI |
| ROCm SMI (various) | Hardware info, clocks, RAS, XGMI errors, partitioning |
| rocminfo | HSA agent enumeration — GPUs and CPUs visible to ROCm |
| clinfo | OpenCL platform and device info |
| NVIDIA Mellanox Info | NUMA topology, IB status, IB devices, OFED version |
| Ethernet IP ADDR | `ip addr`, routes, neighbors |
| Ethernet ethtool | Per-interface link speed, driver, ring settings |
| rdma information | RDMA link and statistics |
| nicctl | Pensando/AMD NIC firmware, QoS, DCQCN, PCIe ATS |

---

## Step 3: Diagnose Common Issues

### GPU Not Detected / Missing GPUs

**Sections to check:** `lspci verbose output`, `dmesg GPU/DRM`, `lsmod`, `AMD SMI list`, `rocminfo`

```bash
# Quick checks in log
grep -i "amdgpu\|GPU\|gfx" <logfile> | grep -i "error\|fail\|not found"
grep "ROCmTechSupportNotFound" <logfile>
```

**Symptoms and causes:**

| Symptom | Likely Cause | Action |
|---|---|---|
| GPU absent from `lspci` | PCIe slot/cable issue, GPU not powered | Check physical seating, power connectors |
| GPU in `lspci` but not in `rocminfo` | `amdgpu` module not loaded or KFD not bound | Check `lsmod` for `amdgpu`, check dmesg for bind errors |
| `amdgpu` in `lsmod` but 0 agents in `rocminfo` | KFD not initialized, unsupported GPU | Check dmesg for `kfd` errors |
| Fewer GPUs than expected | IOMMU grouping issue, PCIe AER error causing device removal | Check `pcieport.*AER` in dmesg |

---

### ROCm Installation Issues

**Sections to check:** `ROCm Packages Installed`, `ROCm Repo Setup`, `Available ROCm versions`, `ROCm ldcache entries`

```bash
grep "ROCm Packages Installed" -A 100 <logfile> | head -60
grep "Available ROCm versions" -A 10 <logfile>
```

**Common problems:**

| Issue | What to look for | Fix |
|---|---|---|
| Missing packages | Key packages (`hip-runtime-amd`, `rocm-smi-lib`, `hsa-rocr`) absent | Re-install via `apt install rocm` or target package |
| Wrong ROCm version picked | `ROCM_VERSION` env var pointing to wrong path | Set `ROCM_VERSION=/opt/rocm-X.Y.Z` explicitly |
| Library not in ldcache | ROCm lib path missing from ldcache | Run `sudo ldconfig` or check `/etc/ld.so.conf.d/` |
| Repo not configured | No `rocm` entries in repo setup section | Follow ROCm install docs for your distro |

---

### Driver / Kernel Module Issues

**Sections to check:** `lsmod`, `amdgpu modinfo`, `dkms status`, `lsinitrd lsinitramfs`, `dmesg`

```bash
grep -E "amdgpu|dkms|initramfs" <logfile>
grep -i "fail\|error" <logfile> | grep -i "dkms\|module"
```

**Common problems:**

| Issue | Signal in log | Fix |
|---|---|---|
| amdgpu-dkms not built | `dkms status` shows `module not found` or `build failed` | `sudo dkms install amdgpu/<version>` |
| amdgpu not in initramfs | `lsinitrd` output missing `amdgpu.ko` | `sudo update-initramfs -u` |
| Module version mismatch | `modinfo` version differs from installed package | Reinstall `amdgpu-dkms` matching kernel |
| udev rules missing | `70-amdgpu.rules` section empty | Install `amdgpu` package or copy rules manually |

---

### RAS / ECC / Memory Errors

**Sections to check:** `dmesg GPU/DRM`, `ROCm SMI showrasinfo`, `AMD SMI bad-pages`

```bash
grep -i "ras\|ecc\|bad.page\|mce\|edac\|uncorrect" <logfile>
```

**What to look for:**

- `ras_mask` in dmesg — RAS feature enable bits
- `rocm-smi --showrasinfo all` — per-block RAS error counts (UE = uncorrectable, CE = correctable)
- `amd-smi bad-pages` — retired memory pages; high count = GPU memory degradation
- MCE/EDAC lines in dmesg — CPU/system memory errors separate from GPU ECC

> **Escalate to AMD support** if: uncorrectable errors (UE) > 0, bad pages growing across reboots, or MCE storms.

---

### XGMI / GPU Interconnect Issues

**Sections to check:** `ROCm SMI showxgmierr`, `AMD SMI xgmi`, `AMD SMI topology`, `rocm-bandwidth-test Topology`

```bash
grep -i "xgmi\|hive\|topo" <logfile>
```

**What to look for:**

- `showxgmierr` — XGMI error counters; any non-zero values indicate link issues
- `amd-smi topology` — expected matrix showing peer GPU connections
- `rocm-bandwidth-test -t` — bandwidth between GPU pairs; low B/W between peers = XGMI link problem
- `lspci` tree — GPU bridge devices should appear between GPUs

---

### PCIe Link Degradation

**Sections to check:** `GPU PCIe Link Config`, `lspci verbose output`, `dmesg`

```bash
grep -E "current_link_width|current_link_speed|PCIe Link" <logfile>
grep "pcieport.*AER\|AER.*error" <logfile>
```

**What to look for:**

- Expected: `Width x16`, Speed `16 GT/s` (PCIe Gen 4) or `32 GT/s` (Gen 5)
- Degraded: `Width x8` or `x4`, Speed `2.5 GT/s` or `5 GT/s` — check slot, retrain PCIe
- AER correctable errors in dmesg — cable/slot signal integrity problem
- AER uncorrectable errors — hardware fault, reseat or replace

---

### ROCm SMI / AMD SMI Not Working

```bash
grep "NOT FOUND\|ROCmTechSupportNotFound" <logfile>
```

| Message | Meaning |
|---|---|
| `rocm-smi NOT FOUND` | ROCm SMI binary missing; install `rocm-smi` package |
| `amd-smi NOT FOUND` | AMD SMI binary missing; install `amd-smi` package (ROCm 5.6+) |
| `rocminfo` not run | `rocminfo` package not installed |
| `clinfo` not found | OpenCL runtime missing |

---

### Network / InfiniBand / RDMA Issues

**Sections to check:** `NVIDIA Mellanox IB output`, `rdma information`, `Ethernet ethtool`, `nicctl`

```bash
grep -i "ibstat\|ACTIVE\|DOWN\|rdma\|ethtool" <logfile>
```

**What to look for:**

- `ibstat`: port state should be `Active`, physical state `Polling` or `LinkUp`
- `ibv_devinfo`: confirms HCA is visible to RDMA stack
- `ethtool`: link speed, duplex — should match switch configuration
- `rdma stat`: packet/byte counters; errors = fabric issue
- `nicctl show dcqcn`: DCQCN parameters for RoCE congestion control

---

### Persistent Logging Warning

If the log contains:

```
WARNING: Persistent logging possibly disabled.
```

Boot logs from previous sessions are not available. Enable them:

```bash
sudo mkdir -p /var/log/journal
sudo systemctl restart systemd-journald.service
```

Then reproduce the issue and re-run the script.

---

## Step 4: Share Logs with AMD Support

1. Compress the log file:
   ```bash
   gzip *.rocm_techsupport.log
   ```
2. Attach to your support ticket or GitHub issue with:
   - Description of the problem (workload, error message, frequency)
   - ROCm version in use
   - Any recent changes (driver update, kernel upgrade, hardware change)

---

## Quick Reference: Key grep Patterns for Log Triage

```bash
# All errors and failures
grep -i -E "error|fail|not found|WARNING" <logfile>

# GPU visibility
grep -i "gfx\|amdgpu\|kfd" <logfile> | grep -i "error\|fail"

# PCIe AER
grep "AER" <logfile>

# ECC/RAS/bad pages
grep -i "ras\|ecc\|bad.page\|mce\|edac" <logfile>

# XGMI errors
grep -i "xgmi" <logfile>

# Missing tools
grep "ROCmTechSupportNotFound" <logfile>

# ROCm version detected
grep "Using.*rocm" <logfile>

# PCIe link width/speed
grep -A2 "GPU [0-9]* PCIe Link" <logfile>
```
