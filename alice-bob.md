# alice / bob — Ryzen 9950X + ConnectX-8 (400 G)

A symmetric pair: two identical hosts, each with **one ConnectX-8**, cabled back-to-back
port-to-port (`p0↔p0`, `p1↔p1`) at **400 G**. Either host can be the generator; the other
receives. Measured 2026-10-02 … 2026-10-04.

**Headline:** one ConnectX-8 sends **300 Mpps at 64 B**, on one port or split over two
(150 + 150). That is the **card's packet-rate ceiling**, not TRex, the CPU, the IOMMU or
PCIe: NVIDIA's own DPDK report shows the same ~300 Mpps flat from 64 to 256 B on a CX-8
with *two* PCIe Gen5 x16 links (see [Why 300 Mpps](#why-300-mpps)).

## Server configuration

| Item | Value |
|------|-------|
| CPU | AMD Ryzen 9 **9950X**, 16 cores / 32 threads, Zen 5 |
| Core→thread map | physical core `N` → logical threads `N` and `N+16` |
| RAM | 30 GB |
| NIC | **1× NVIDIA ConnectX-8** (`15b3:1023`), dual-port 400 G, `01:00.0` / `01:00.1` |
| PCIe | **Gen5 x16** (`32GT/s (downgraded)` — the card is Gen6; the second x16 extension is not cabled) |
| NIC NV config | `CQE_COMPRESSION=AGGRESSIVE`, `PCI_WR_ORDERING=force_relax` |
| Peer (receiver) | the other host of the pair — same hardware |

## Environment & versions

| Component | Version |
|-----------|---------|
| OS | Ubuntu 24.04 LTS |
| Kernel | 6.8.0-146-generic, boot with `iommu=pt` (see below) |
| NIC driver | `mlx5_core` (inbox) — no MLNX_OFED |
| NIC firmware | 40.50.1002 (`MT_0000001222`) |
| TRex | built from **trex-core master `27e0153b`** (DPDK 25.07; banner says v3.08) |
| Link | 400 Gb/s per port |

> **No TRex release knows the ConnectX-8.** Releases up to v3.06 do not recognise
> `15b3:1023`; build trex-core from source. Two build gotchas: GCC 13 fails on `-Werror`
> (delete `'-Werror',` from `linux_dpdk/ws_main.py`), and the `scripts/` binaries are
> symlinks into the build directory (dereference them when copying the install).

> **MTU:** TRex's default port MTU (65518) is rejected by the CX-8 — set `port_mtu: 9000`.

## Platform config

One instance, both ports, 30 workers (all cores but 0 and its sibling 16):

```yaml
# /etc/trex_cfg.yaml
- port_limit      : 2
  version         : 2
  port_mtu        : 9000
  interfaces      : ["0000:01:00.1", "0000:01:00.0"]
  port_info       :
    - ip          : "10.0.1.1"
    - ip          : "10.0.2.2"
  platform        :
    master_thread_id  : 0
    latency_thread_id : 16
    dual_if           :
      - socket         : 0
        threads        : [1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,17,18,19,20,21,22,23,24,25,26,27,28,29,30,31]
```

```bash
echo 2048 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
cd /opt/trex && ./t-rex-64 -i --iom 0 --no-scapy-server --no-ofed-check --no-watchdog -c 30
```

Profile: one continuous UDP stream per port at `percentage=100`, source IP incremented over
65,536 values (field engine, `cache_size=255`) so the receiver's RSS spreads it.

> **Frame sizes on this page include the FCS** (64 B = the canonical 64-byte wire frame; 400 G
> line rate = 595 Mpps), unlike the 64 B-before-FCS convention of the flame1/lava1 pages.
> 300 Mpps is therefore ~50 % of one 400 G port's line rate.

## Frame-size sweep (pair test)

TX = generator port counters (`tx_packets_phy`); RX = receiver port counters
(`rx_packets_phy`); "delivered" = RX minus what the receiving NIC discarded before host
memory (`rx_discards_phy` + `rx_out_of_buffer`, receiver also running TRex). Both
directions measured; they agree within 1–2 %.

| Frame (incl. FCS) | Ports | TX | L1 rate | RX at peer | Delivered to peer's memory |
|-----------------|-------|----|---------|------------|----------------------------|
| 64 B | 1 | **300.0 Mpps** | 202 Gbps | 300.0 | 268.8 |
| 64 B | 2 | **300.0 Mpps** (150 + 150) | 202 Gbps | 300.0 | 290.5 |
| 128 B | 1 | 200.0 Mpps | 237 Gbps | 200.0 | 200.0 |
| 128 B | 2 | 200.0 Mpps | 237 Gbps | 200.0 | 200.0 |
| 256 B | 1 | 120.0 Mpps | 265 Gbps | 120.0 | 120.0 |
| 256 B | 2 | 120.0 Mpps | 265 Gbps | 120.0 | 120.0 |
| 512 B | 1 | 65.1 Mpps | 277 Gbps | 65.1 | 65.1 |
| 512 B | 2 | 84.4–85.9 Mpps | 359–365 Gbps | same | same |
| 1518 B | 1 | 32.2 Mpps | 396 Gbps | 32.2 | 32.2 |
| 1518 B | 2 | 32.2–33.4 Mpps | 396–411 Gbps | same | same |

- **64 B is capped at 300 Mpps per card** whether one port or two — a second port adds nothing.
- **128 / 256 B fall short of 300** (200 / 120): these fit `Mpps × (frame + 64 B) ≈ 38.4 GB/s`,
  a per-packet transfer cost on the sending card. NVIDIA reaches ~300 Mpps here with
  `txq_inline_mpw=64`, `MaxReadReq=4096` and a second x16 link (not tried yet).
- **Large frames reach ~400 Gbps per card in total**, consistent with one Gen5 x16 link.
- **Receiving at 64 B:** all 300 Mpps arrive; the receiving card discards ~31 Mpps before
  host memory with one port (~10 Mpps with two).

## Maximum bandwidth — both ports

The most one CX-8 sends, both ports saturated, is **~400–411 Gbps total** (L1) at 1518 B —
about **50 % of the 800 G** the two 400 G ports could carry. Measured in both directions
(2026-10-02 and 2026-10-04), zero receiver drops:

| Frame | 1 port | 2 ports (total) | Share of 800 G |
|-------|--------|-----------------|----------------|
| 64 B | 202 Gbps (300 Mpps) | 202 Gbps (300 Mpps) | 25 % |
| 512 B | 275–277 Gbps | 359–365 Gbps | ~46 % |
| **1518 B** | 365–396 Gbps | **396–411 Gbps** (32.2–33.4 Mpps) | **~51 %** |
| 4096 B | 361 Gbps | 372–383 Gbps | ~47 % |
| 9000 B | 333–334 Gbps | 339–354 Gbps | ~43 % |

- **The second port adds almost nothing at large frames** — one port already reaches ~390 Gbps.
  The ceiling is the card's single **PCIe Gen5 x16** link (~512 Gb/s raw), not the 400 G ports.
- **Jumbo frames are slower, not faster.** 4096 B and 9000 B fall to 340–380 Gbps (TRex
  splits frames above its 2 KB buffers into multi-segment packets, adding descriptors per
  frame). 1518 B is the best case.
- **The way to ~800 G is the second x16 link.** NVIDIA's CX-8 report (two Gen5 x16 links)
  reaches **100 % of 800 G from 512 B up** (188 Mpps at 512 B, 65 Mpps at 1518 B). With the
  card's x16 extension cabled, both ports should approach 2×400 G at ≥ 512 B; 64 B stays at
  the 300 Mpps packet-rate ceiling.

## Why 300 Mpps

Ruled out on this box, one at a time (64 B, sender = bob):

| Changed | Result |
|---------|--------|
| TRex workers 10 / 16 / 20 / 30 | 300 Mpps every time — not CPU; TRex reports millions of `queue_full` (waiting on the NIC) |
| mlx5 devargs (`dpdk_devargs` in `trex_cfg.yaml`), 8 variants: `txq_inline_max=0,txq_inline_mpw=0`, `txqs_min_inline=0`, `txq_inline_mpw=256`, `tx_db_nc=0`, `tx_db_nc=2`, `txq_mpw_en=1,txq_inline_mpw=128,txqs_min_inline=0` | 296–301 Mpps on 1 port, 299–303 on 2 |
| `txq_mpw_en=0` (multi-packet send off) | **221 Mpps** on 1 port (worse); 308 on 2 |
| IOMMU translated → **passthrough** (`iommu=pt`) | 300.0 Mpps on 1 and 2 ports — no change |
| PCIe check | Gen5 x16, MaxPayload 512, MaxReadReq 1024 — as DPDK's mlx5 guide recommends |

NVIDIA's [DPDK 25.03 NIC performance report](https://fast.dpdk.org/doc/perf/DPDK_25_03_NVIDIA_NIC_performance_report.pdf)
(Test #16, Table 50 — a ConnectX-8 in Socket Direct mode, i.e. **two** PCIe Gen5 x16 links,
testpmd macswap on 32 cores, zero packet loss):

| Frame | Mpps | % of 800 G line rate |
|-------|------|----------------------|
| 64 B | 299.61 | 26.37 |
| 128 B | 299.54 | 44.34 |
| 256 B | 299.66 | 82.72 |
| 512 B | 187.96 | 100 |
| 1518 B | 65.01 | 100 |

Flat ~300 Mpps from 64 to 256 B with twice our PCIe bandwidth: **300 Mpps is the CX-8's packet
rate**. Cabling the second x16 link will not raise 64 B; more than 300 Mpps needs a **second
card**. (For comparison, one ConnectX-7 tops out at ~279 Mpps — see [lava1](lava1.md).)

## TRex and RDMA

TRex has no separate "RDMA mode" — on ConnectX it **already runs on the RDMA verbs stack**, so
every number on this page is the verbs-path result:

- TRex sends and receives through DPDK's **mlx5 PMD**, which is built on **rdma-core**
  (`libibverbs` + the mlx5 provider) and opens each port through its kernel verbs device —
  not through VFIO/UIO. DPDK's mlx5 guide: load `ib_uverbs mlx5_core mlx5_ib`; *"User space I/O
  kernel modules (UIO, VFIO) are not used"*; interfaces must be *"linked to kernel verbs"*.
- On bob/alice each CX-8 port has its verbs device (`enp1s0f0np0 → uverbs0`,
  `enp1s0f1np1 → uverbs1`, RDMA device `rocep1s0f0`), `mlx5_ib` / `ib_uverbs` are loaded and
  no `vfio-pci` / `uio` module is in use. The TRex image installs `rdma-core`,
  `libibverbs1` and `ibverbs-providers`.
- Verbs (and the mlx5 Direct Verbs / DevX interfaces) set up the queues and flow steering;
  after that the PMD writes send descriptors straight into the NIC's queues from user space
  and polls completions — the same kernel-bypass datapath RDMA applications use. The kernel
  netdev stays up alongside it (*bifurcated driver*), which is why `ethtool -S` counters can be
  read during a run.

So the 300 Mpps / ~400 Gbps ceilings above are measured on the verbs datapath of this card and
PCIe link. (RDMA benchmarks such as perftest's `ib_write_bw` / `raw_ethernet_bw` were not run.)

## Randomised attack profiles

The pair test above uses one cheap stream per port. Heavier per-packet field-engine profiles
(randomised addresses/ports on every packet, used for DUT tests) are CPU-bound in TRex:

| Profile | Ports | Offered |
|---------|-------|---------|
| fixed-size flood, randomised fields, 64 B before FCS | 1 | 200 Mpps requested → **200 Mpps** sent |
| same | 2 (200 + 200 requested) | **~180 Mpps total** |
| IMIX 7:4:1 of 64/570/1518 | 1 | ~129 Mpps (~390 Gbps) |

So for a two-port 64 B attack the generator, not the card, is the limit at ~180 Mpps.

## Tweaks

Same as [tuning-checklist.md](tuning-checklist.md), plus:

- **`iommu=pt`** on the kernel command line. It did not change TRex's rate; after switching,
  a 15-worker VPP on the same host started in ~72 s instead of ~170 s.
- 2 MB hugepages only (`2048 × 2 MB`), no 1 G pages on a 30 GB box.
- Inbox mlx5, skip MLNX_OFED (`--no-ofed-check`).
- Watch the CX-8 temperature under sustained load (`/sys/class/hwmon/*` with name `mlx5*`):
  ~61–65 °C idle, ≤ 66 °C during all runs here, critical 105 °C.
