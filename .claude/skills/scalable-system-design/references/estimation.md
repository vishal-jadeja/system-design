# Estimation — do this before any design decision

The point is not precision. The point is discovering **what class of system you are building**, so you don't build a distributed one when a single node would do (or a single node when the numbers say otherwise).

## The five-line template

```
DAU / events per day        = ...
QPS (avg)                   = events/day ÷ 10^5      # 86,400 s ≈ 10^5
QPS (peak)                  = avg × peak multiplier
Bytes/record × records/day  = storage/day → × retention → × replication
Working set (hot data)      = ...   → does it fit one machine's RAM?
Read:write ratio            = ...   → decides where the effort goes
```

Round aggressively (99,987 / 9.1 → 100,000 / 10). **Always label units** ("5 MB", never "5"). Write assumptions next to the numbers.

## Peak multipliers

| Traffic shape | Multiplier |
|---|---|
| Steady consumer product | **2×** average |
| Spiky / event-driven / market-hours / launches | **5×** average |

Market-hours systems concentrate into their open window, not 24 h: 1B orders/day over a 6.5 h session = 43K QPS avg, 215K peak.

## Funnel-back estimation

When only the last step's volume is known, work backward through the conversion funnel:

> 3 TPS bookings, ~10% step-through ⇒ booking page 30 QPS ⇒ detail page 300 QPS. Read:write ≈ 100:1.

Same trick forward: 1B DAU × 1 click = 1B events/day = 10K QPS avg, 50K peak, 0.1 KB each = 100 GB/day ≈ 3 TB/month.

## Growth

30% YoY ⇒ **traffic doubles every ~3 years**. Design for one doubling, not ten. Rethink the architecture roughly every order of magnitude of load — an architecture fit for 1× rarely survives 10×.

## Latency numbers (Jeff Dean, 2020-refreshed)

| Operation | Time |
|---|---|
| L1 cache ref | 0.5 ns |
| Branch mispredict | 5 ns |
| L2 cache ref | 7 ns |
| Mutex lock/unlock | 100 ns |
| Main memory ref | 100 ns |
| Compress 1 KB | 10 µs |
| Send 2 KB over 1 Gbps | 20 µs |
| SSD random read | 16 µs |
| 1 MB sequential from memory | 3 µs |
| 1 MB sequential from SSD | 49 µs |
| **Round trip in same datacenter** | **500 µs** |
| Disk seek (HDD) | 2–10 ms |
| 1 MB sequential from disk | 0.8 ms (SSD-era) – 30 ms (HDD) |
| 1 MB sequential from network | 10 ms |
| **Packet CA → Netherlands → CA** | **150 ms** |

Consequences: memory is fast, disk seeks are slow — avoid them. Compression is cheap relative to network — compress before sending. Cross-region traffic is expensive. **Several network hops is already single-digit milliseconds**; a disk-backed event store is tens of ms.

## Availability / SLA

| Target | Downtime per year | Per day |
|---|---|---|
| 99% | 3.65 days | 14.4 min |
| 99.9% | 8.77 h | 1.44 min |
| 99.99% | 52.6 min | **8.64 s** |
| 99.999% | 5.26 min | 0.86 s |

At four nines, "restart it in a minute" is already a violation — recovery must be automatic. AWS per-region compute SLA is only 99.95%; know your provider's number and that they cap *their* liability, not your loss.

## Storage / capacity constants

- Powers of two: 2^10 KB · 2^20 MB · 2^30 GB · 2^40 TB · 2^50 PB.
- One SATA 7200 rpm disk ≈ **100–150 random IOPS**; sequential + RAID gives several hundred MB/s. Rotational disks are only slow for *random* access.
- Disk block typically 4 KB — sub-4KB objects waste a whole block, and **inode count is fixed at disk init**, so millions of small files exhaust it. Merge small objects into large append-only files.
- 6 TB drive @ 150 MB/s sequential ≈ 11 hours to fill — useful for sizing log retention buffers.
- A relational DB node on typical datacenter hardware: "a few thousand" TPS. Use 1,000 TPS/node as a conservative planning constant, then remember **a transfer is 2 operations** (debit + credit).
- InfluxDB-class TSDB with 8 cores / 32 GB: >250K writes/s, >1M unique series.
- Redis Pub/Sub: idle channel ≈ 20 bytes of pointers per subscriber; a conservative 100K subscriber-pushes/s per server on a gigabit NIC.
- Erlang/BEAM process ≈ 300 bytes (millions per box) — relevant only if you can staff it.
- WebSocket connection ≈ 10 KB of memory ⇒ 1M concurrent ≈ 10 GB.

## Worked examples to copy the *shape* of

**Twitter-like posting.** 300M MAU, 50% daily → 150M DAU; 2 posts/user/day → 150M×2 / 86,400 ≈ 3,500 QPS; peak ≈ 7,000. 10% carry 1 MB media → 30 TB/day → 55 PB over 5 years.

**Nearby friends.** 1B users, 100M DAU, **concurrent ≈ 10% of DAU** = 10M. Location refresh every 30 s (a human walking makes finer pointless) → 334K location-update QPS. 400 friends avg, ~10% online+nearby → **14M forwarded updates/s** — the fan-out, not the ingest, is the bottleneck. Then: memory needed 200 GB (2 servers) but CPU/network needed 140 servers ⇒ **the system is CPU-bound, not memory-bound**. Size both; scale the binding one; say which it is.

**Quadtree geo index.** Leaf 832 B, internal 64 B, internal ≈ ⅓ of leaves; 200M businesses ⇒ 2M leaves ⇒ **1.71 GB total, fits one server** ⇒ read replicas, not sharding. Build time a few minutes — which makes slow startup an availability risk.

**Object storage.** 100 PB at 40% usage ratio, size mix 20% <1 MB / 60% 1–64 MB / 20% >64 MB ⇒ 0.68B objects ⇒ at ~1 KB metadata each ⇒ 0.68 TB of metadata. The metadata store and data store then get sized and scaled separately.

**Sanity check that kills work:** 5,000 hotels × 20 room types × 2 years × 365 days = **73M rows** and ~3 TPS ⇒ "single DB is fine; replicate for HA, not capacity." Do this check every time before proposing distribution.

## Client-side batching: the cheapest order of magnitude

Choose an update cadence; **do not inherit the sensor's cadence.** GPS every second at 5B nav-minutes/day = 3M QPS. Batching client writes every 15 s = 200K QPS — a 15× reduction before a single server is added. Adapt the interval to context (slower when the user is stationary).

## Estimation pitfalls

- Sizing only memory when throughput is the binding constraint (or vice versa).
- Forgetting the replication factor in storage math.
- Forgetting that one logical operation may be 2+ physical DB ops.
- Using the average when the design must survive the peak.
- Using a strict latency budget everywhere — real-time bidding needs sub-1s, billing aggregation on the *same events* needs minutes. Different subsystems get different budgets; inheriting the strictest one everywhere is how over-engineering starts.
- Precision theater: 6 significant figures on an assumption you invented.
</content>
