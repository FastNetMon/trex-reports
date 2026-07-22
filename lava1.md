# lava1 — Ryzen 9950X + 2× ConnectX-7

**Current config (2026-07-22):** two *separate* CX-7 cards, **both ports of each cabled** —
the `.1` ports to server1's two CX-5 cards, the `.0` ports to a Mikrotik CRS504 switch.
Two engines reach **~282 Mpps = 100 % of 2×100 G** at 64 B on the server1 pair (see
[Two cards](#two-cards--200-g-at-true-line-rate-2-connectx-7)); driving **all four ports**
peaks at **~558 Mpps = 98.2 % of 400 G** (the 2× CX-7 engine ceiling — see
[Max total TX](#max-total-tx--all-four-ports-at-the-engine-ceiling-558-mpps)). Minimal-core
and 1500 B / 400 G results are below.

## Server configuration

| Item | Value |
|------|-------|
| CPU | AMD Ryzen 9 **9950X**, 16 cores / 32 threads, Zen 5, up to 5.7 GHz |
| Motherboard | MSI **MEG X870E ACE MAX** (AM5) — PCIe 5.0 x16 slot |
| Core→thread map | physical core `N` → logical threads `N` and `N+16` |
| RAM | 30 GB |
| NIC | **2× Mellanox ConnectX-7** (MT2910) — for 2-card generation, one cabled port per card: `01:00.1` (card 1) + `02:00.1` (card 2), linked at **100 G** to server1's two **ConnectX-5** cards. |
| PCIe | Gen5 — **x16** with one card; **x8 + x8** with two (the board splits the CPU x16; no small-packet penalty) |
| Driver | inbox `mlx5` (kernel 6.8) — no MLNX_OFED needed |
| Ports | card 1 `01:00.1` (enp1s0f1np1) · card 2 `02:00.1` (enp2s0f1np1) |
| Peer (receiver) | server1 — **2× ConnectX-5 Ex** (`01:00.1` + `81:00.1`), one cabled port per card at 100 G |

## Environment & versions

| Component | Version |
|-----------|---------|
| OS | Ubuntu 24.04.4 LTS |
| Kernel | 6.8.0-134-generic |
| NIC driver | `mlx5_core` (inbox, kernel 6.8) — no MLNX_OFED |
| NIC firmware | 28.44.1036 (`MT_0000000834`, ConnectX-7) |
| TRex | 3.06 (bundles its own DPDK) |
| Link | 100 Gb/s per port (200 G ports capped to the 100 G peer) |

> **Hugepages:** reserve **2 MB pages only** (`2048 × 2 MB = 4 GB`). Do **not** reserve 1 G
> pages — the default `32 × 1 GB` would consume the whole 30 GB box.

---

## Single-port / single-instance (up to ~185 Mpps aggregate)

30 data-plane workers (15 physical cores + HT siblings); master + latency on core 0
and its sibling 16.

```yaml
# /etc/trex_cfg.yaml
- port_limit      : 2
  version         : 2
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
ip addr flush dev enp1s0f1np1
ip addr flush dev enp1s0f0np0
cd /opt/trex && ./t-rex-64 -i --iom 0 --no-scapy-server --no-ofed-check -c 30
```

| Frame builder | Wire size | Rate |
|---------------|-----------|------|
| 64 B pre-FCS  | 68 B | **142.0 Mpps** single-port (100 % line rate) |
| 60 B pre-FCS  | 64 B | **147.4 Mpps** single-port (99 % of 148.8) |

**Both ports in one instance stalls at ~185 Mpps** — every core services two TX rings.
For full dual-port rate, run two instances (below).

---

## Dual-port ~278 Mpps — two TRex instances

TRex binds its data-plane cores to a *port pair*, so it cannot dedicate cores per port
inside one instance. The fix is **two independent instances**, each owning **one real
port + one `dummy` port**, with **disjoint core sets** and separate hugepage prefixes /
ZMQ ports:

- **Instance A** → port `0000:01:00.1`, master 0, latency 16, workers `1-7,17-23` (14)
- **Instance B** → port `0000:01:00.0`, master 8, latency 24, workers `9-15,25-31` (14)

```yaml
# /etc/trex_cfgA.yaml
- port_limit      : 2
  version         : 2
  interfaces      : ["0000:01:00.1", "dummy"]
  prefix          : trexA
  limit_memory    : 1024
  zmq_pub_port    : 4500
  zmq_rpc_port    : 4501
  port_info       :
    - ip          : "10.0.1.1"
    - ip          : "10.0.2.2"
  platform        :
    master_thread_id  : 0
    latency_thread_id : 16
    dual_if           :
      - socket         : 0
        threads        : [1,2,3,4,5,6,7,17,18,19,20,21,22,23]
```

```yaml
# /etc/trex_cfgB.yaml
- port_limit      : 2
  version         : 2
  interfaces      : ["0000:01:00.0", "dummy"]
  prefix          : trexB
  limit_memory    : 1024
  zmq_pub_port    : 4506
  zmq_rpc_port    : 4507
  port_info       :
    - ip          : "10.0.3.1"
    - ip          : "10.0.4.2"
  platform        :
    master_thread_id  : 8
    latency_thread_id : 24
    dual_if           :
      - socket         : 0
        threads        : [9,10,11,12,13,14,15,25,26,27,28,29,30,31]
```

```bash
echo 3072 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
ip addr flush dev enp1s0f1np1
ip addr flush dev enp1s0f0np0
rm -f /dev/hugepages/trexA* /dev/hugepages/trexB* /dev/hugepages/rtemap_*

cd /opt/trex
./t-rex-64 -i --iom 0 --no-scapy-server --no-ofed-check -c 14 --cfg /etc/trex_cfgA.yaml &
sleep 3
./t-rex-64 -i --iom 0 --no-scapy-server --no-ofed-check -c 14 --cfg /etc/trex_cfgB.yaml &
```

Drive both instances from one client (RPC ports 4501 / 4507), push the same 64 B
profile on each at `mult="100%"`.

### Result

| Setup | Per port | Aggregate |
|-------|----------|-----------|
| Both ports, one instance (shared cores) | ~93 Mpps | ~185 Mpps (65 %) |
| **Two instances, split cores** | **~139 Mpps** | **~277–280 Mpps (~98 %)** |

~278 Mpps is the **device ceiling** of the single dual-port ConnectX-7 (one shared
packet engine across both ports) — every host counter stays clean: zero pause frames,
zero PCIe stalls, zero discards, cores ~89 % with TRex's `queue_full` climbing (software
waiting on the NIC). NVIDIA's DPDK report shows the same shape: CX-7 single-port 267 Mpps,
dual-port 267 Mpps — the second port adds nothing.

## 400 G at 1500 B — dual-port against a 200 G peer

The 64 B numbers above were taken against a **100 G** peer, so single-port capped at
the 100 G line rate (142 Mpps). Against a **ConnectX-7 200 G peer** the link runs at
its native **200 G/port**, which exposes the generator's real ceilings — and shows how
they flip with frame size. Same platform configs; drive a 1500 B profile
(`PKTSIZE=1500`, `size 1500-1500`) for the large-frame runs.

| Frame | Setup | Per port | Aggregate |
|-------|-------|----------|-----------|
| 64 B  | single-port | **~274 Mpps** | — |
| 64 B  | two instances (both ports) | ~139 Mpps | **~278 Mpps** |
| 1500 B | single-port | **16.4 Mpps** (200 G, 100 %) | — |
| **1500 B** | **two instances (both ports)** | 16.4 Mpps | **32.8 Mpps = 400 G (100 %)** |

At **64 B** the wall is the CX-7's shared **packet engine** (~278 Mpps aggregate, ~274
single-port) — bandwidth is irrelevant (278 Mpps × 68 B ≈ 150 G of the 400 G of wire).
At **1500 B** the packet rate is low (~16.4 Mpps/port), the engine limit never bites,
and the generator delivers the **full 2×200 G = 400 G**. So the same box is
**packet-engine-limited at 64 B** but **true line rate at 1500 B on both ports**.

## Two cards — 200 G at true line rate (2× ConnectX-7)

The ~278 Mpps dual-port ceiling above is *one* card's **shared** packet engine. Give the
box **two separate CX-7 cards** (one cabled port each) and you get **two independent
engines**, so each card reaches its *full* line rate and the shared-engine deficit
disappears. lava now runs 2× CX-7 — `01:00.1` on card 1, `02:00.1` on card 2 — each
linked at 100 G to a separate peer; same two-instance split, one instance per card
(`flame/conf/twocard_setup.sh`).

| Setup | Per card | Aggregate | % of 2×100 G |
|-------|----------|-----------|--------------|
| One dual-port CX-7 (shared engine) | — | ~278 Mpps | 98 % |
| **Two CX-7 cards (two engines), 64 B** | **~141 Mpps** (100 % of 100 G) | **~282 Mpps** | **99.3 %** |
| Two CX-7 cards, 60 B | ~148 Mpps | **~296 Mpps** | 99.5 % of 297.6 |

Each card hits **141 Mpps = 100 % of its 100 G line rate**, so the aggregate is
essentially line rate — where the single dual-port card capped at 98 %. Two gotchas:

- **Flow control must be OFF.** With PAUSE on, the receiver throttled TX to **~99 Mpps
  per card** until `ethtool -A <if> rx off tx off` on both ends (`perf-tune` does this).
- **Gen5 x8 is fine.** With two cards the board splits its CPU x16 into **x8 + x8**;
  each card ran at Gen5 x8 with **no penalty** — small-packet TX is transaction- not
  bandwidth-bound, and x8 has ample transaction rate.

### Minimal cores + spare-CPU headroom

2×100 G line rate needs only **6 workers per instance = 12 of the 16 cores** — 4 gives
162 Mpps, 6 reaches the full 282, and more doesn't help. The core map (masters + latency
share cores 0 / 8 via HT):

```text
Ryzen 9950X — 16 physical cores (each = 2 HT threads)

 core:  0  │ 1  2  3  4  5  6 │  7   │ 8  │ 9 10 11 12 13 14 │ 15
        m+l│  card 1 TX (6)   │ FREE │ m+l│  card 2 TX (6)   │ FREE
         A │  -> 100 G        │      │ B  │  -> 100 G        │
```

→ **14 of 16 cores used, cores 7 & 15 free**, and every TX core's HT sibling is idle too
(**16 of 32 threads unused**). So 200 G leaves real spare CPU. `twocard_setup.sh` defaults
to `NWORK=6`.

### Max total TX — all four ports at the engine ceiling (~558 Mpps)

Each CX-7 card has two ports: the `.1` ports flood server1's CX-5 cards, the `.0` ports
flood a Mikrotik CRS504 switch. Drive **both ports of each card** (still one instance per
card — mlx5 needs one primary process per card, so 4 separate instances is impossible).

**The generation profile dominates the ceiling here.** A per-packet **field-engine VM**
(randomising the source IP on every packet — what a DUT RSS test needs) costs ~9 % of CPU
and caps the total at **~512 Mpps** (per-port 124–131, CPU-bound). A **static single
packet** (no VM — TRex's cheapest TX path) removes that cost and every port jumps to the
hardware **engine ceiling**:

| Port | Destination | 64 B randomised-src | 64 B static |
|------|-------------|---------------------|-------------|
| card 1 · `01:00.1` | server1 CX-5 | ~131 Mpps | 139.5 Mpps |
| card 1 · `01:00.0` | Mikrotik | ~131 Mpps | 139.5 Mpps |
| card 2 · `02:00.1` | server1 CX-5 | ~125 Mpps | 139.4 Mpps |
| card 2 · `02:00.0` | Mikrotik | ~125 Mpps | 139.4 Mpps |
| **TOTAL** | | **~512 Mpps** | **~558 Mpps** |

**558 Mpps = 98.2 % of the 568 Mpps (400 G) 4×100 G line rate** — every port at ~139.5,
i.e. each card at its full **~279 Mpps engine ceiling** (2× a single dual-port CX-7's 278).
That is the hardware wall, confirmed three ways:

- **Engine-bound, not CPU.** Only **6 workers per instance (12 of the 32 threads)** are
  needed for the full 558: 3 / 4 / 5 workers scale up CPU-bound (180 / 238 / 209 Mpps),
  6 reaches 558 and 8–14 hold it. `--mbuf-factor 4` adds nothing.
- **More cores per card *hurt*.** One card with 30 workers on its two TX rings drops to
  **259 Mpps** (ring over-subscription) vs **279** at 6–14 workers — throwing cores at a
  card past its engine only adds contention.
- **The last 1.8 % needs a third engine.** 568 Mpps would put each port at 142 = each card
  at 284, past the 279 engine. Two CX-7 engines cap at ~558; the full 400 G at 64 B needs a
  **3rd CX-7** or a **ConnectX-8** (~300 Mpps/card in NVIDIA's
  [DPDK 25.03 report](https://fast.dpdk.org/doc/perf/DPDK_25_03_NVIDIA_NIC_performance_report.pdf)).

So the box has **huge spare CPU**: the static max-TX (`flame/conf/maxtx_push.py`) hits 558
with just 6 workers/card, leaving cores 7 & 15 and every HT sibling idle (14 of 32 threads
unused). The randomised-VM flood (`bothports_push.py`) spends all 14 workers/card for its
512. Either way the NIC engine, not the 9950X, is the wall. (`flame/conf/bothports_setup.sh`.)

### Scaling beyond 558 — more cards?

Because throughput scales with the number of **engines (cards)**, not ports — each CX-7
delivers ~279 Mpps whether one port or two — **more cards is the only way up, and it scales
~linearly**. Each added CX-7 = +279 Mpps at 64 B. What stops us on *this* box is the host,
not the idea:

| Limit on the 9950X / AM5 | Ceiling it imposes |
|--------------------------|--------------------|
| **PCIe lanes** — board splits the CPU x16 into x8 + x8 for the 2 cards; AM5 has ~24 usable Gen5 lanes (GPU + NVMe take the rest) | ~2, maybe 3, x8 slots |
| **CPU cores** — static TX needs **6 cores/card** (the measured floor); 16 cores feed ~2 cards (12c) with 2 free | ~2 cards for static; a 3rd would need 18 cores |

So **2 cards (558 Mpps) is this platform's practical peak**. To go higher, change the
platform or the NIC:

- **ConnectX-8** — NVIDIA's DPDK 25.03 report shows ~**300 Mpps/card** at 64 B, so **2× CX-8
  = ~600 Mpps > 568**: two CX-8 cards would deliver the **full 400 G at 64 B** on a board
  with two x8 Gen5 slots (no extra cores needed — still engine-bound).
- **More CX-7 on a big platform** — a many-lane, many-core host (Threadripper / EPYC) with
  the cores to spare (6/card) and the slots (x8 each) scales ~linearly: **3× CX-7 ≈ 837, 4×
  ≈ 1116 Mpps**. The generator software already supports it (one TRex instance per card);
  it is purely a hardware question of lanes + cores.

Bottom line for the team: **on the 9950X we are at the 2-card ceiling (558 / 98 % of 400 G).
Full 400 G at 64 B, or beyond, needs either 2× ConnectX-8 or ≥3× ConnectX-7 on a
higher-lane/‑core platform.**

## Tweaks

Same as [flame1](flame1.md#tweaks) / [tuning-checklist.md](tuning-checklist.md); the only
lava-specific notes are the Zen 5 core-pair map (`N` / `N+16`), 2 MB pages only, and inbox
mlx5 (skip MLNX_OFED).
