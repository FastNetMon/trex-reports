# flame1 — Ryzen 5800X + ConnectX-5 Ex (100 G)

Single-port 100 GbE, 64-byte line rate (142 Mpps).

## Server configuration

| Item | Value |
|------|-------|
| CPU | AMD Ryzen 7 **5800X**, 8 cores / 16 threads, up to 4.7 GHz |
| Motherboard | MSI **B450 Tomahawk** (MS-7C02, v1.0) — 1× PCIe 3.0 x16 slot |
| Core→thread map | physical core `N` → logical threads `N` and `N+8` |
| NIC | Mellanox **ConnectX-5 Ex**, dual-port 100 GbE (single card, one NUMA node) |
| PCIe | Gen3 x16 (B450 chipset caps the slot at PCIe 3.0 — ~126 Gbps usable, still clears 100 G line rate) |
| Driver | inbox `mlx5` (or MLNX_OFED) |
| Ports | `enp38s0f0np0` (26:00.0, sender) · `enp38s0f1np1` (26:00.1, receiver/sink) |

## Environment & versions

| Component | Version |
|-----------|---------|
| OS | Ubuntu 24.04.4 LTS |
| Kernel | 6.8.0-136-generic |
| NIC driver | `mlx5_core` (inbox) |
| NIC firmware | ConnectX-5 Ex (record the exact string with `ethtool -i <if>` when the port is kernel-bound) |
| TRex | 3.06 (bundles its own DPDK) |
| Link | 100 Gb/s per port |

## TRex configuration (`/etc/trex_cfg.yaml`)

14 data-plane workers (7 physical cores + their HT siblings); master + latency on
core 0 and its sibling.

```yaml
- port_limit      : 2
  version         : 2
  interfaces      : ["0000:26:00.0", "0000:26:00.1"]
  port_info       :
    - ip          : "10.0.1.1"
    - ip          : "10.0.2.2"
  platform        :
    master_thread_id  : 0
    latency_thread_id : 8
    dual_if           :
      - socket         : 0
        threads        : [1,2,3,4,5,6,7,9,10,11,12,13,14,15]
```

## Launch

```bash
# 2 MB hugepages (800 × 2 MB = 1.6 GB, fits 14 workers)
echo 800 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# flush kernel IPs off the data-plane NICs (bifurcated driver shares the NIC)
ip addr flush dev enp38s0f0np0
ip addr flush dev enp38s0f1np1

# start TRex (interactive/STL, 14 data cores)
cd /opt/trex
./t-rex-64 -i --iom 0 --no-scapy-server --no-ofed-check -c 14
```

## Peak numbers (64 B)

| Frame builder | Wire size | Rate | % of line rate |
|---------------|-----------|------|----------------|
| 64 B pre-FCS  | 68 B | **142.0 Mpps** | 100 % of 142.05 |
| 60 B pre-FCS (canonical 64 B incl. FCS) | 64 B | **147.4 Mpps** | 99 % of 148.8 |

Single-port line rate is reached with all 14 workers active; ~4–5 fast cores per
port is the practical minimum.

## Tweaks

Host + NIC tuning is common to both generators — see [tuning-checklist.md](tuning-checklist.md).
The single biggest one is NIC **CQE compression = AGGRESSIVE**.
