# Tuning checklist — host + NIC tweaks for 64 B line rate

Applied on both generators. In rough order of impact.

## NIC (mlx5 / ConnectX)

| Tweak | Command | Why |
|-------|---------|-----|
| **CQE compression = AGGRESSIVE** | `mlxconfig -y -d <pci> set CQE_COMPRESSION=1` | Halves RX completion (CQE) traffic — the single biggest small-packet win. Persists in NIC NV config. |
| Link speed forced, autoneg off | `ethtool -s <if> speed 100000 autoneg off` | Pins 100 G when a 200 G port links to a 100 G peer. |
| Pause frames off | `ethtool -A <if> rx off tx off` | No flow-control back-pressure skewing the TX rate. |
| Flush kernel IPs | `ip addr flush dev <if>` | Bifurcated driver shares the NIC with the kernel; a kernel IP interferes with the DPDK queues. |

## CPU / power

| Tweak | Command |
|-------|---------|
| Governor → performance | `echo performance > /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor` |
| Disable C-states above C1 | `echo 1 > /sys/devices/system/cpu/cpu*/cpuidle/state*/disable` (skip POLL/C0/C1/C1E); or `cpupower idle-set -D 1` |

## PCIe

| Tweak | Command | Why |
|-------|---------|-----|
| MaxReadRequest → 1024 B | `setpci -s <pci> cap_exp+0x8.w=<val>` | Larger read bursts; measured +~1 Mpps on the high-cardinality profiles. |

## Memory / scheduler

| Tweak | Command |
|-------|---------|
| 2 MB hugepages | `echo <N> > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages` |
| No 1 G hugepages | leave `hugepages-1048576kB/nr_hugepages` at 0 (TRex uses 2 M only) |
| NUMA reclaim off | `sysctl vm.zone_reclaim_mode=0` |
| Swappiness off | `sysctl vm.swappiness=0` |
| irqbalance off | `systemctl stop irqbalance` (avoid IRQ migration jitter) |

Optional (needs GRUB edit + reboot): `nohz_full=<cores> rcu_nocbs=<cores> rcu_nocb_poll`
on the TRex worker cores — removes the kernel timer tick, reducing scheduler jitter.

## BIOS settings (AMD Ryzen / EPYC generators)

Set once in firmware; the OS tweaks above assume these.

| Setting | Value | Why |
|---------|-------|-----|
| SMT / Hyper-Threading | **Enabled** | TRex uses the HT sibling of each physical core as a worker |
| Global C-states / C6 | **Disabled** | deep sleep adds wake-up latency spikes under bursty poll load |
| Core Performance Boost | **Enabled** | high clocks — clock beats core count for TX |
| Power / Determinism | **Performance** | AMD "Determinism Slider = Performance" (or "Power Supply Idle Control = Typical") for stable clocks |
| NUMA (NPS) | **NPS1** | single memory domain (Ryzen is single-die; on EPYC generators NPS1/NPS2 keep per-domain bandwidth high) |
| Above 4G Decoding | **Enabled** | required for large PCIe BARs on modern NICs |
| PCIe link speed | **Auto / max (Gen4/Gen5)** | let the NIC negotiate full width and generation |
| Memory profile | **EXPO/XMP, max supported speed** | memory-bandwidth headroom |

Verify at runtime: `LnkSta` in `lspci -vvv` for the negotiated PCIe gen/width, and
`turbostat`/`cpupower frequency-info` for sustained clocks.

## TRex launch flags

```
./t-rex-64 -i --iom 0 --no-scapy-server --no-ofed-check -c <n_data_cores>
```

- `-i` — interactive / STL API mode.
- `--iom 0` — no I/O monitor (headless).
- `--no-scapy-server` / `--no-ofed-check` — skip startup dependencies not needed for STL.
- `-c` — number of data-plane cores (must match the `threads` list in the cfg).

## Core budget

- ~**35 Mpps per fast core** for TX; ~**4–5 fast cores per 100 G port** for line rate.
- **Clock beats core count** past ~40 cores. A 5.7 GHz Ryzen with 14 threads reaches line
  rate where a 2.1 GHz dual-Xeon with 36 threads tops out ~135 Mpps.
- Master + latency threads sit on one physical core and its HT sibling; data-plane workers
  get the rest (one instance per port pair, disjoint cores — see [lava1.md](lava1.md)).
