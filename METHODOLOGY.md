# Methodology

## What we measure

These reports document **generator TX capability** — the packet rate a TRex
generator can *send* — not a device's No-Drop-Rate (NDR). We report the peak
sustained TX pps for a given profile and frame size. (For a DUT's zero-loss
throughput, use TRex's built-in NDR binary search — see
[trex_ndr_bench_doc](https://github.com/cisco-system-traffic-generator/trex-core/blob/master/doc/trex_ndr_bench_doc.asciidoc).)

## Frame-size convention

Our stream builders emit **64 B before FCS**; the NIC appends the 4 B FCS, so the
wire frame is **68 B**. Per-packet wire cost is `68 + 8 (preamble) + 12 (IFG) =
88 B = 704 bit`, so 100 GbE line rate for these frames is:

```
100 Gb/s ÷ 704 bit = 142.05 Mpps  =  100 % line rate
```

The **148.8 Mpps** figure only applies to canonical 64-B-*including*-FCS frames
(60 B before FCS), which measure ~147.4 Mpps (99 %).

## How a run is taken

1. Start TRex with the machine's platform config and the host/NIC tuning applied.
2. Push the profile at `mult="100%"`.
3. **Warm** ~5–6 s (let clocks/queues settle), then **sample** per-port `tx_pps`
   from `get_stats` over ~10–35 s and average.
4. For dual-port, drive both instances from one client and hold traffic while
   sampling (so host-side counters can be read in parallel).

## Determinism / variance

- High-cardinality profiles (syn/ack/icmp/frag/fixed floods) exercise the full
  path on every packet and are **stable run-to-run to well under 1 %**.
- `udp-rand`'s short runs vary a few percent and should not be used for A/B
  comparison of small tweaks.
- Source IPs are randomised across a /16 so the receiver's L3-src RSS spreads
  traffic evenly across all RX queues (measured flatter than hand-picked
  per-queue IPs — CoV ~0.1 %).

## Caveats

- Numbers are **generator-side**; DUT drop/forward rates are a separate measurement.
- Results reflect the documented HW/SW versions; re-verify after firmware, kernel,
  or TRex changes.
