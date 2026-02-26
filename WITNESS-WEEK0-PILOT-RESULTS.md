# Witness Week 0 Pilot Results

**Date:** 2026-02-26
**Protocol:** WITNESS-WEEK0-EXTRACTION-PROTOCOL.md
**Extractor:** Single (no inter-rater calibration)

---

## Summary

| Document | Candidates | Extractable | Rate | Func / Perf | T1 / T2 / T3 / T4 | Numeric % | Cross-fn % |
|---|---|---|---|---|---|---|---|
| RFD 110 (CockroachDB) | 86 | 53 | 61.6% | 41 / 12 | 12 / 35 / 6 / 0 | 30.2% | 1.9% |
| RFD 445 (Backpressure) | 32 | 26 | 81.3% | 22 / 4 | 9 / 10 / 5 / 1 | 53.8% | 26.9% |
| RFD 490 (Packed Extents) | 45 | 31 | 68.9% | 29 / 2 | 14 / 11 / 5 / 0 | 38.7% | 25.8% |
| RFD 58 (Rack Switch) | 115 | 88 | 76.5% | 78 / 10 | 55 / 25 / 7 / 0 | 17.0% | 37.5% |
| RFD 63 (Network Arch) | 75 | 53 | 70.7% | 52 / 1 | 19 / 24 / 10 / 0 | 13.2% | 17.0% |
| **Aggregate** | **353** | **251** | **71.1%** | **222 / 29** | **109 / 105 / 33 / 1** | **25.9%** | **23.1%** |

**Go/No-Go: Go.** 5/5 documents (100%) yield ≥5 extractable properties. Threshold was 70%.

### Aggregate Metrics

| Metric | Value |
|---|---|
| Documents yielding ≥5 properties | 5/5 (100%) |
| Mean extractable properties per document | 50.2 |
| Median extractability rate | 70.7% |
| Functional / Performance ratio | 7.7 : 1 |
| Tier 1–2 share | 85.3% (214/251) |
| Tier 3–4 share | 13.5% (34/251) |
| Ambiguity rate (T?) | 0.0% (0/251) |

### Secondary Signals

| Signal | Observed | Threshold | Implication |
|---|---|---|---|
| Ambiguity rate | 0% | >30% triggers concern | Tier rubric was sufficient without disambiguation |
| Performance properties | 11.6% of total | >60% triggers concern | Most verification is environment-independent |
| Tier 3/4 properties | 13.5% of total | >40% triggers concern | Cost model holds — cheap verification dominates |
| Cross-function rate | 23.1% | >50% triggers concern | Manageable integration testing load |
| Numeric threshold rate | 25.9% overall (varies 13–54%) | <50% triggers note | Extraction tool needs structural/behavioral analysis, not just regex-for-numbers |

### Five Hardest Extraction Decisions

1. **Equilibrium convergence (RFD 445, P2.17):** `T_ack + T_bp = T_done` is a mathematical invariant of the feedback loop. Functional (algorithm property, not hardware-dependent) but Tier 4 (proving stability for all workloads requires control-theory analysis). Empirical testing (Tier 2) can observe convergence but not prove universality.

2. **ZFS transaction splitting (RFD 490, P3.09):** Crucible's crash consistency depends on ZFS splitting writes at recordsize boundaries. External to Crucible's codebase, but Crucible implicitly asserts this property. Extracted as cross-function/Tier 3 rather than rejecting as external — it is a relied-upon kernel contract, not an external service.

3. **Cluster convergence after failure (RFD 110, P1.28):** "The cluster always converged to sufficient replication without operator intervention." Liveness property over all failure types and cluster states. Tier 3 (state-machine analysis), not Tier 2, because the "always" quantifier over diverse failure modes exceeds what property-based testing can prove.

4. **Goal-level requirements without thresholds (RFD 63, N1–N9):** "Feel performant," "minimize resets," "handle failures with degradation." Tempting targets but consistently lack decidable bounds. Rejected all as non-extractable (vague).

5. **Bandwidth properties under normal vs. failure (RFD 58, P4.4/P4.5/P4.6):** Three closely related but distinct properties with different preconditions (both ASICs active, one failed, per-flow limit). Extracted separately because they require different test procedures and have different implications.

---

## Document 1: RFD 110 — CockroachDB for the Oxide Control Plane

**Source:** https://rfd.shared.oxide.computer/rfd/0110
**Domain:** Distributed database / control plane infrastructure

### Extracted Properties (53)

```
P1.01  Functional  T1  All data stored in sorted KV store; keyspace partitioned into contiguous non-overlapping Ranges
P1.02  Functional  T1  Default Range size threshold for splitting: 512 MiB
P1.03  Functional  T2  Every Range has replicas = configured replication factor (default 3) in steady state
P1.04  Functional  T2  Each Range operates independent Raft consensus; writes committed only after Raft majority
P1.05  Functional  T2  Reads for Range R served by leaseholder node for R
P1.06  Functional  T2  In steady state, leaseholder == Raft leader for each Range
P1.07  Functional  T3  When leaseholder != Raft leader, reads delayed to guarantee linearizability
P1.08  Functional  T2  Every write processed through Raft before acknowledgment; no write acked without majority
P1.09  Functional  T2  Any node can serve as gateway for SQL queries, routing KV ops to leaseholders
P1.10  Functional  T1  PostgreSQL wire protocol; standard pg client can connect and issue SQL
P1.11  Functional  T2  Node declared dead after 5 minutes without heartbeat (logical timeout, not perf)
P1.12  Functional  T2  Dead node triggers under-replication; cluster auto-replicates to restore factor
P1.13  Functional  T2  Range with factor N requires floor(N/2)+1 replicas for availability
P1.14  Functional  T3  Losing K nodes causes unavailability for any Range with >floor((R-1)/2) replicas on failed nodes
P1.15  Functional  T2  Node self-terminates when clock offset >500ms from cluster mean
P1.16  Performance T2  chrony/NTP keeps clock sync within 1ms under normal network conditions
P1.17  Functional  T2  Server auto-retries transactions when statements batched and results bufferable
P1.18  Functional  T2  Auto-retry requires: (a) batched statements AND (b) bufferable results
P1.19  Functional  T2  Non-auto-retryable conflicts return retryable error code to client [cross-fn]
P1.20  Functional  T2  Auto range splitting (size), splitting (load), and merging (size)
P1.21  Functional  T2  Auto replica rebalancing across nodes based on load
P1.22  Functional  T2  Online schema changes complete without client-visible errors
P1.23  Performance T2  p95 latency jumps from ~3ms to ~100ms during I/O-heavy schema changes
P1.24  Performance T2  After SIGKILL, node recovers and serves requests within <10 seconds
P1.25  Performance T2  After OS reboot, node resumes within ~90 seconds (dominated by OS boot)
P1.26  Functional  T2  Transient partition <5min produces zero client errors on non-partitioned nodes
P1.27  Performance T2  Extended partition: multi-second latency outliers confined beyond p99
P1.28  Functional  T3  After any single-node failure, cluster converges to sufficient replication without intervention
P1.29  Performance T2  Backup of 81 GiB / 43M records via cockroach dump completes in ~33 minutes
P1.30  Performance T2  Restore of same dataset via cockroach sql: ~6 hours, largely single-threaded
P1.31  Functional  T2  After backup/restore cycle, record count matches original
P1.32  Functional  T2  During rolling upgrade, only the upgrading node's requests affected
P1.33  Functional  T1  Graceful shutdown includes drain phase with configurable timeout (default 60s)
P1.34  Performance T2  Default 60s drain timeout insufficient under tested workloads
P1.35  Functional  T1  PebbleDB writes 32 KiB blocks with compression before filesystem write
P1.36  Performance T2  Query latency varies 10–100% depending on gateway-to-leaseholder distance
P1.37  Performance T2  Queries at leaseholder gateway save at least one internal RTT vs non-leaseholder
P1.38  Functional  T3  No committed data lost during single-node failures with quorum available
P1.39  Functional  T3  Strong consistency (linearizability) for all committed operations
P1.40  Functional  T2  Only designed crash mode is clock offset exceeding threshold
P1.41  Performance T2  Expansion on small instances: 2-3min near-unavailability, 20-30min 2-3x latency
P1.42  Performance T2  Expansion on large instances: p95 doubles, no complete unavailability
P1.43  Functional  T2  240-hour continuous workload: no data degradation
P1.44  Functional  T1  GC period is configurable at runtime
P1.45  Functional  T1  cockroach debug zip collects: settings, events, rangelogs, nodes, queries, stacks, env, logs, range metadata, schema, pprof
P1.46  Functional  T1  cockroach debug zip excludes threads.txt on illumos (Linux/Glibc only)
P1.47  Functional  T1  Supports mutual TLS for inter-node and client-server connections
P1.48  Functional  T2  Range-based sharding preserves key ordering; efficient range scans without sort
P1.49  Functional  T1  BSL license converts to Apache 2.0 three years post-release
P1.50  Functional  T1  Distributed backup/restore under CCL, not BSL; unavailable in OSS-only builds
P1.51  Functional  T1  cockroach dump output format: SQL command file
P1.52  Functional  T2  Range replica movement observable through built-in metrics during expansion/contraction
P1.53  Functional  T2  Node decommission auto-migrates all replicas before removal
```

### Non-Extractable Candidates (33)

**Vague (11):** "Solid enough" (subjective judgment), "Strong focus on hands-off operation" (qualitative), "Excellent Jepsen report" (subjective), "Superior operational simplicity" (comparative/subjective), "Good tooling" (subjective), "Comprehensive metrics" (unquantified), "Obviously isn't fast" (subjective), "Generally stable with occasional latency spikes" (unquantified), "performance inconsistent over time" (vague — partially captured in P1.36), "No apparent mechanism to manually rebalance leases" (admits uncertainty), "Operator moves might be undone" (speculative).

**Meta (10):** Ad hoc test execution path dependencies, automated data collection recommendation, node count determination, replication factor determination, load balancing approach, operationalize rolling upgrades, plan TLS testing, develop custom load balancer, extended production workloads, confirm load balancing approach.

**External (12):** Rust client library support, Cockroach Labs ZFS experience, illumos metrics gaps (partially → P1.46), haproxy integration, PostgreSQL comparison issues (5 items), TiDB description, YugabyteDB description, overload behavior (untested), schema change throttling (open question), GC visibility (open question), cluster splitting (open question), manual rebalancing (open question), risk assessments (subjective), logical replication (vague), CCL legal interpretation, patent grants, pricing changes.

### Metrics

| Metric | Value |
|---|---|
| Total candidates | 86 |
| Extractable | 53 |
| Rate | 61.6% |
| Functional / Performance | 41 / 12 |
| T1: 12, T2: 35, T3: 6, T4: 0, T?: 0 | |
| Numeric threshold rate | 30.2% (16/53) |
| Cross-function rate | 1.9% (1/53) |

### Hard Decisions

1. **P1.11 (5-min dead node timeout): Functional vs Performance.** The 5-minute duration involves time but is a logical state-machine transition threshold, not a hardware-dependent measurement. Same as "session expires after 30 min inactivity." Decision: Functional.

2. **P1.28 (cluster convergence): Tier 2 vs Tier 3.** The "always converged" claim over all failure types requires state-machine analysis, not just empirical testing. Decision: Tier 3.

3. **P1.26 (zero errors during transient partition): Functional vs Performance.** Binary outcome (zero errors vs nonzero) with a logical time boundary (dead node timeout). Decision: Functional — the time bound is a configured threshold, not a performance measurement.

---

## Document 2: RFD 445 — Crucible Upstairs Backpressure

**Source:** https://rfd.shared.oxide.computer/rfd/0445
**Domain:** Distributed storage / flow control

### Extracted Properties (26)

```
P2.01  Functional   T2  Bytes backpressure activates (delay > 0) at >= 1 GiB in-flight write bytes
P2.02  Performance  T2  At 2 GiB in-flight, bytes backpressure delay ≈ 10ms (quadratic from 1 GiB onset)
P2.03  Functional   T2  Job-count backpressure activates at >= 0.05 * IO_OUTSTANDING_MAX (2,850 jobs)
P2.04  Performance  T2  Job-count backpressure max delay: 5ms at IO_OUTSTANDING_MAX
P2.05  Functional   T1  Effective delay = max(bytes_delay, job_count_delay)
P2.06  Functional   T1  Writer holds mutex lock during backpressure delay (prevents bypass)
P2.07  Functional   T3  Backpressure applies only to writes, never to reads or flushes
P2.08  Functional   T2  Per-client in-progress job limit: 2,600 (MAX_ACTIVE_COUNT)
P2.09  Functional   T3  At IO_OUTSTANDING_MAX (57,000) live jobs, Downstairs declared faulted [cross-fn]
P2.10  Functional   T1  Write path stages: Guest::write → backpressure → Guest::reqs → consume_req → process_new_io → ds_new [cross-fn]
P2.11  Functional   T1  IOP/BW limiting operates on consume_req rate, disconnected from backpressure
P2.12  Functional   T3  With IOP/BW limits active but disconnected, Guest::reqs queue unbounded
P2.13  Functional   T2  Jobs above MAX_ACTIVE_COUNT in ds_new still counted for backpressure
P2.14  Functional   T1  Bytes backpressure is unclamped (no max delay, unlike job-count's 5ms cap)
P2.15  Performance  T2  Large writes + non-responsive Downstairs + unclamped bytes BP: >100 days to fault [cross-fn]
P2.16  Performance  T2  Small writes + non-responsive Downstairs: ~47 seconds to fault [cross-fn]
P2.17  Functional   T4  Equilibrium invariant: T_ack + T_bp = T_done (queue length stabilizes) [cross-fn]
P2.18  Functional   T1  Both backpressure components use quadratic (degree-2) scaling
P2.19  Functional   T3  Without backpressure, no upper bound on Upstairs write buffer RAM per guest
P2.20  Functional   T3  Flush blocks until ALL preceding write operations complete [cross-fn]
P2.21  Functional   T1  Propolis exposes configurable MDTS limiting max single guest write size
P2.22  Functional   T2  1M and 4M writes stabilize at same job count (upstream splitting) [cross-fn]
P2.23  Functional   T1  Backpressure implemented as artificial delay in write return path (not rejection)
P2.24  Functional   T3  Fast-ack: writes acked before persistence to any Downstairs [cross-fn]
P2.25  Functional   T1  IOP/BW limiting never enabled in production
P2.26  Functional   T1  After crucible#1047, job-count onset at 5,700 (changed from 28,500)
```

### Non-Extractable Candidates (6)

**Vague (5):** Literary epigraph, "robust and well-tuned" (no threshold), "makes backpressure robust against small and large writes" (captured as P2.05), "measuring throughput is generally problematic" (general observation), "speeding up Upstairs can paradoxically increase latency" (qualitative — mechanism captured in P2.17).

**Meta (6):** Document purpose statement, 4 recommendations for future changes, "does not introduce new security considerations."

**External (5):** ZFS write throttle formula (prior art), "pings occur every 5 seconds" (health check, not backpressure), "rack exists within bounded resource constraints" (truism), tuning recommendation (future), "While unusual, this approach functions correctly" (vague — specific behavior captured in P2.13).

### Metrics

| Metric | Value |
|---|---|
| Total candidates | 32 |
| Extractable | 26 |
| Rate | 81.3% |
| Functional / Performance | 22 / 4 |
| T1: 9, T2: 10, T3: 5, T4: 1, T?: 0 | |
| Numeric threshold rate | 53.8% (14/26) |
| Cross-function rate | 26.9% (7/26) |

### Hard Decisions

1. **P2.17 (equilibrium equation): Tier 2 vs Tier 4.** The universal claim "the system stabilizes" for all workloads requires stability proof of the feedback loop, not just empirical observation. Decision: Tier 4.

2. **P2.03 vs P2.26 (threshold evolution): Which is current?** RFD documents both the design formula (0.05 * IO_OUTSTANDING_MAX = 2,850) and the post-PR value (5,700). Extracted both as separate properties of different system states.

3. **P2.12 (unbounded RAM): Extractable or vague?** "Unbounded amounts of RAM" sounds imprecise, but the mechanism is specific: Guest::reqs has no size limit when IOP/BW limits slow consume_req. Decision: Extractable as a negative property (absence of bound).

---

## Document 3: RFD 490 — Packed Crucible Extents

**Source:** https://rfd.shared.oxide.computer/rfd/0490
**Domain:** Storage / file format optimization

### Extracted Properties (31)

```
P3.01  Functional   T1  Raw format: block i at byte offset i * 4096
P3.02  Functional   T1  Raw format per-block context: 96 bytes (18.75% overhead for 512-byte blocks)
P3.03  Functional   T1  Context: 48 bytes per slot, two slots (A/B ping-pong), 96 bytes total per block
P3.04  Functional   T1  PackedBlockContext serializes to exactly 32 bytes (bincode, any variant)
P3.05  Functional   T1  Packed format overhead: 6.25% for 512-byte blocks (32/512)
P3.06  Functional   T1  Each packed row fills exactly one ZFS record (recordsize bytes)
P3.07  Functional   T3  Packed format: block + context persist atomically or not at all [cross-fn]
P3.08  Functional   T3  Block data and context always in same ZFS transaction group (txg) [cross-fn]
P3.09  Functional   T3  ZFS splits writes at recordsize boundaries; non-crossing write = one txn [cross-fn]
P3.10  Functional   T1  Packed file metadata stores recordsize; validated against filesystem on open
P3.11  Functional   T1  PackedBlockContext: enum {None, Unencrypted(u64), Encrypted(nonce:[u8;12], tag:[u8;16])}
P3.12  Functional   T1  Packed format removes on_disk_hash; relies on decryption or ZFS checksums
P3.13  Functional   T2  PackedWriteHeader.checksum: digest of data payload; receiver verifies [cross-fn]
P3.14  Functional   T1  PackedWriteHeader fields: upstairs_id, session_id, job_id, dependencies, eid, offset, checksum
P3.15  Functional   T2  Packed layout: block/context interleaved within each row, padded to recordsize
P3.16  Functional   T1  Read-only regions stay raw format (not migrated to packed)
P3.17  Functional   T3  All read-write regions migrated to packed (at startup or first write) [cross-fn]
P3.18  Functional   T2  Raw extent reads auto-translated to PackedReadResponse (transparent to Upstairs) [cross-fn]
P3.19  Functional   T2  Encrypted blocks: integrity validated by decryption (success = valid) [cross-fn]
P3.20  Functional   T3  Raw format crash safety: after any crash, at least one context slot valid
P3.21  Functional   T3  Raw format: defragmentation when context slot usage becomes mixed A/B
P3.22  Functional   T1  Packed Downstairs treats context as opaque blob (no interpretation, only size)
P3.23  Functional   T1  No backward wire compatibility; all rack nodes updated simultaneously [cross-fn]
P3.24  Functional   T2  Downstairs uses pwritev with scatter-gather: alternating block/context segments
P3.25  Functional   T2  blocks_per_row = floor(recordsize / (block_size + context_size)); remainder is padding
P3.26  Functional   T1  EncryptionContext: nonce [u8; 12] + tag [u8; 16]
P3.27  Performance  T2  Packed extents: >= 1.30x throughput for 1-block random reads on reference config
P3.28  Performance  T2  Packed extents: no throughput regression for any tested read size (1–1024 blocks)
P3.29  Functional   T1  Default ZFS recordsize: 128 KiB
P3.30  Functional   T2  Raw format: reading one block requires at least two separate I/O ops (block + context)
P3.31  Functional   T2  Packed format: reading one block requires at most one I/O op (co-located in record)
```

### Non-Extractable Candidates (14)

**Vague (5):** "Maximizes mechanical sympathy with ZFS," "potential for further optimization," "best gains for small reads" (no threshold), "possible for context slots to get fragmented" (possibility, not invariant), "venture beyond POSIX into DMU layer" (hypothetical).

**Meta (4):** "This RFD proposes a packed layout," determinations/open questions/security sections (no content), "I anticipate..." (future intent — technical content extracted as P3.10), "still updating rack simultaneously" (process — consequence extracted as P3.23).

**External (4):** "ZFS checksums protect unencrypted data" (ZFS property — consequence in P3.12), "writes split at max_blksz boundaries" (ZFS internal — extracted as P3.09 with cross-fn flag), "decryption validates data" (crypto algorithm property — consequence in P3.19), individual benchmark MiB/sec values (environment-specific data points — ratios extracted as P3.27/P3.28).

**Diffuse (1):** "Packed layout reduces I/O from 2 to 1" (spread across multiple sections — synthesized as P3.30 + P3.31).

### Metrics

| Metric | Value |
|---|---|
| Total candidates | 45 |
| Extractable | 31 |
| Rate | 68.9% |
| Functional / Performance | 29 / 2 |
| T1: 14, T2: 11, T3: 5, T4: 0, T?: 0 | |
| Numeric threshold rate | 38.7% (12/31) |
| Cross-function rate | 25.8% (8/31) |

### Hard Decisions

1. **P3.09 (ZFS transaction splitting): External vs extractable.** Crucible's entire crash model depends on this ZFS behavior. Extracted as cross-function/Tier 3 (relied-upon kernel contract, not external service call).

2. **Performance benchmarks: Exact values vs conservative bounds.** Individual MiB/sec numbers are meaningless outside the test environment. Extracted two conservative properties: 1.30x speedup for small reads (P3.27) and no-regression guarantee (P3.28). Rejected 13 individual data points.

3. **P3.07 vs P3.20: Current format vs proposed format.** Both are verifiable properties of their respective implementations, and both coexist during migration. Extracted both.

---

## Document 4: RFD 58 — Rack Switch Architecture

**Source:** https://rfd.shared.oxide.computer/rfd/0058
**Domain:** Hardware / network switch design

### Extracted Properties (88)

```
P4.01  Functional   T1  No single point of failure in rack switch hardware [cross-fn]
P4.02  Functional   T1  Exactly two switching ASICs, both active under normal operation
P4.03  Functional   T1  Each ASIC connected to each compute node via 100GBASE-CR4
P4.04  Performance  T2  200Gb/s non-blocking aggregate bandwidth between compute nodes (both ASICs) [cross-fn]
P4.05  Performance  T2  100Gb/s per flow during single-ASIC failure; connectivity maintained [cross-fn]
P4.06  Performance  T2  Per-flow limit: 100Gb/s (single-path forwarding)
P4.07  Functional   T1  32 backplane ports per ASIC for compute nodes
P4.08  Functional   T1  Up to 32 QSFP28 optical uplink ports per ASIC
P4.09  Functional   T1  QSFP28 ports support 100GBASE-SR (>=100m), -LR4 (>=10km), -ER4 (>=40km)
P4.10  Performance  T1  Tofino 2: 12.8Tb/s switching capacity, 64x200G ports
P4.11  Functional   T1  ASIC programmable via P4 language
P4.12  Functional   T2  ASIC supports VxLAN and Geneve encap/decap
P4.13  Functional   T1  Tofino 2 SKUs: 32+1, 48+1, 64+1 port configurations
P4.14  Functional   T1  Full-mesh supports up to 32 racks per cell [cross-fn]
P4.15  Functional   T1  31 inter-rack links per rack in 32-rack full mesh [cross-fn]
P4.16  Performance  T2  Oversubscription 1:16 at 32 compute nodes [cross-fn]
P4.17  Performance  T2  Oversubscription 1:12 at 24 compute nodes [cross-fn]
P4.18  Functional   T1  SP control plane: L2 Ethernet (frame-based)
P4.19  Functional   T2  SP control: unicast + broadcast (broadcast for discovery)
P4.20  Functional   T2  SP broadcast confined within rack switch domain (no uplink leakage) [cross-fn]
P4.21  Functional   T1  SP broadcast domain operational without external config dependencies
P4.22  Functional   T2  Rate-limiting and broadcast storm protection on SP control traffic
P4.23  Functional   T3  SP control has highest forwarding priority of all traffic classes [cross-fn]
P4.24  Functional   T1  Host control plane: L3 routed (not L2 switched)
P4.25  Functional   T1  Switch implements L3 routing for host control traffic
P4.26  Functional   T2  Switch classifies packets by ECN bits and VLAN priority
P4.27  Functional   T2  Host control broadcast domain isolated from SP control domain [cross-fn]
P4.28  Functional   T2  Application traffic uses VxLAN/Geneve encapsulation
P4.29  Functional   T2  Switch implements floating IPs, load balancing, packet filtering [cross-fn]
P4.30  Functional   T3  Stateful load balancing and filtering (per-flow/per-connection state) [cross-fn]
P4.31  Functional   T2  Software and config attestation detects unauthorized modifications [cross-fn]
P4.32  Functional   T3  Single-ASIC failure: all compute nodes retain connectivity via remaining ASIC [cross-fn]
P4.33  Functional   T1  ASIC-to-host via PCIe x4
P4.34  Functional   T2  Switch board supports same attestation as compute node board [cross-fn]
P4.35  Functional   T1  SP has GPIO control of PRSNT#, PERST#, PWREN#, PWRFLT#, SMBus
P4.36  Functional   T3  During PCIe hot-plug, ASICs continue forwarding without interruption [cross-fn]
P4.37  Functional   T1  ASIC power rails independent of PCIe hot-plug power signals
P4.38  Functional   T1  VSC7444-02: 24x 1G SGMII + 2x 10G SerDes + 1x SGMII NPI
P4.39  Functional   T1  Two VSC7444-02 ASICs together provide >= 34 SGMII ports
P4.40  Functional   T1  One 10G SerDes connects VSC7444 device A to Tofino 2 [cross-fn]
P4.41  Functional   T1  Two VSC7444-02s interconnected via 10G SerDes [cross-fn]
P4.42  Functional   T1  32 management ports (100M/1G) for compute node SPs
P4.43  Functional   T1  2 management ports for PSC
P4.44  Functional   T1  1 BASE-T port for operator console
P4.45  Functional   T1  1 port for auxiliary power equipment (ATS/BBU)
P4.46  Functional   T1  Optional inter-rack switch port (100M/1G/10G) [cross-fn]
P4.47  Functional   T1  KSZ8463FRL per compute node: 3-port switch (1 RMII + 2 SGMII)
P4.48  Functional   T1  Management backplane uses DC-balanced SGMII encoding (AC-coupled)
P4.49  Functional   T1  PHY-to-backplane: 100FX SerDes (no transformers needed)
P4.50  Functional   T1  SP connects to KSZ8463FRL via RMII
P4.51  Functional   T1  2x SGMII backplane connections per compute node (one to each switch) [cross-fn]
P4.52  Functional   T1  Dedicated I2C segment per QSFP28 module
P4.53  Functional   T1  QSFP28 uses fixed I2C address; read/write via control word
P4.54  Functional   T1  QSFP28 power classes 1-8 supported; class 8 max 10W
P4.55  Performance  T2  QSFP28 temperature polling interval <= 1 second
P4.56  Functional   T2  Thermal management for up to 32 simultaneous QSFP28 modules [cross-fn]
P4.57  Functional   T2  QSFP28 asserts LOS indicator on signal loss
P4.58  Functional   T3  LOS triggers port disable; signal restore triggers port enable [cross-fn]
P4.59  Functional   T1  Tofino 2 includes on-die temperature sensing diode
P4.60  Functional   T1  External temp sensor with programmable trip point for Tofino 2
P4.61  Functional   T1  Separate inlet and exhaust air temperature sensors
P4.62  Functional   T1  Fan controller: programmable (set speed) and monitorable (read speed)
P4.63  Functional   T2  SP executes closed-loop thermal control (read temps → adjust fans) [cross-fn]
P4.64  Functional   T1  Failsafe sensor alarm threshold set near Tofino 2 Tj_max
P4.65  Functional   T1  Failsafe alarm wired directly to ASIC power sequencing (hardware shutdown) [cross-fn]
P4.66  Functional   T1  Fan power rails independent of ASIC emergency shutdown rails [cross-fn]
P4.67  Functional   T1  Fans: 48V, 80mm, dual-rotor
P4.68  Functional   T1  Rear fans: toolless CRU (customer replaceable unit)
P4.69  Functional   T2  Fans hot-serviceable: removal during operation, no service interruption [cross-fn]
P4.70  Functional   T1  One service LED per serviceable component
P4.71  Functional   T2  Service LEDs indicate health status, not traffic activity
P4.72  Functional   T1  Front-facing system-level service LED
P4.73  Functional   T1  Service LED per QSFP28 port
P4.74  Functional   T1  Service LED per serviceable fan
P4.75  Functional   T1  Service LED per front-facing RJ45 technician port
P4.76  Functional   T1  NC-SI not used; dedicated management network instead
P4.77  Functional   T1  Each sidecar: 3 Oxide Units (3OU) of rack space
P4.78  Functional   T1  Data plane: custom twinax backplane (not DAC cables), 100GBASE-CR4
P4.79  Functional   T1  Blind-mate connectors for compute node installation
P4.80  Functional   T2  SGMII auto-negotiation works over AC-coupled backplane
P4.81  Functional   T1  10G connection (10GBASE-KX or SFI) from management net to Tofino 2 [cross-fn]
P4.82  Functional   T1  Architecture supports >= 200 racks / >= 6400 compute nodes [cross-fn]
P4.83  Performance  T2  >= 51.2Tb/s cross-sectional bandwidth per network rack [cross-fn]
P4.84  Functional   T1  >= 256 fabric ports at 200Gb/s per network rack [cross-fn]
P4.85  Functional   T1  PCIe hot-plug conforms to ExpressModule specification
P4.86  Functional   T2  Single hot-plug event attaches/detaches multiple subsystems [cross-fn]
P4.87  Functional   T3  FPGA bitstream failure must not permanently prevent SP network access [cross-fn]
P4.88  Functional   T2  If PCA9543 I2C switches used, software implements bus error recovery
```

### Non-Extractable Candidates (27)

**Vague (12):** "Low volume" (SP control, no number), "Moderate traffic volume" (host control), "Majority of traffic" (application, no threshold), "Major portion of traffic" (block storage), "Rudimentary rate-limiting" (no rate), "High-capacity data plane" (no number), "Rapid completion required" (no time bound), "Minimize firmware scope" (unbounded), "Reuse compute node design where feasible" (subjective), "Five to ten year life" (too broad), "Higher bandwidth than I2C" (comparative, no value), "Service complexity mitigation" (unmeasurable).

**Meta (9):** Leverage existing firmware work, integrator-controlled features, future-proofing, customer-specific flexibility, engineering support hours (Tomahawk 3), vendor support refusal (Broadcom), NC-SI deferral rationale, fan design provenance, LED design philosophy.

**External (6):** Customer network traffic origin, optical transceiver reach standards (property of optics, not switch — compatibility extracted as P4.09), IEEE 1588 pending decision, external RFD references (11 documents), TI auto-negotiation reference, "Reuse in subsequent rack generations" (future intent).

### Metrics

| Metric | Value |
|---|---|
| Total candidates | 115 |
| Extractable | 88 |
| Rate | 76.5% |
| Functional / Performance | 78 / 10 |
| T1: 55, T2: 25, T3: 7, T4: 0, T?: 0 | |
| Numeric threshold rate | 17.0% (15/88) |
| Cross-function rate | 37.5% (33/88) |

### Hard Decisions

1. **P4.23 (highest forwarding priority): Tier 2 vs Tier 3.** "Highest among all traffic classes under any congestion scenario" requires analyzing all congestion states, not just test cases. Decision: Tier 3.

2. **P4.55 (1s polling interval): Extractable from "straw-man" proposal?** Only numeric value given. Explicit, traceable, decidable. "Straw-man" noted but does not prevent extraction. Decision: Extractable/Performance.

3. **P4.4/P4.5/P4.6 (bandwidth properties): One property or three?** Different preconditions, different assertions, different test procedures. Decision: Three separate properties.

---

## Document 5: RFD 63 — Network Architecture

**Source:** https://rfd.shared.oxide.computer/rfd/0063
**Domain:** Network architecture / software-defined networking

### Extracted Properties (53)

```
P5.01  Functional  T1  AZ assigned one IPv6 /48; all rack prefixes are subnets of that /48
P5.02  Functional  T1  Rack assigned one /56 subnet of parent AZ /48 (8-bit rack ID)
P5.03  Functional  T1  Host assigned one /64 subnet of parent rack /56 (8-bit host ID)
P5.04  Functional  T1  Addressing supports 256 racks per AZ and 256 hosts per rack
P5.05  Functional  T2  Bootstrap: servers self-assign unique addresses within fdb0::/64 (MAC-derived, no collisions)
P5.06  Functional  T1  Bootstrap address derived solely from SP MAC address
P5.07  Functional  T1  Bootstrap agent assigns ::1 host address within derived /64
P5.08  Functional  T2  Routing daemon advertises bootstrap prefix to exactly both connected switches [cross-fn]
P5.09  Functional  T2  Switches re-advertise bootstrap prefixes only to servers, never externally [cross-fn]
P5.10  Functional  T1  Geneve VNI field: exactly 24 bits
P5.11  Functional  T1  Geneve options: max 252 bytes, 4-byte aligned
P5.12  Functional  T1  No Geneve-defined control plane; control plane entirely implementation-defined
P5.13  Functional  T1  VLAN IDs constrained to 12 bits (0-4095)
P5.14  Functional  T2  SP and PSC on separate VLANs; no frame leakage between domains [cross-fn]
P5.15  Functional  T1  Point-to-point data links use IPv6 link-local (fe80::/10) for routing
P5.16  Functional  T2  Off-rack destinations: routing table has exactly two equal-cost next-hops (one per switch)
P5.17  Functional  T1  Routing protocol rejects unsigned route advertisements
P5.18  Functional  T2  Separate trust chains: server-key for intra-rack, rack-key for inter-rack [cross-fn]
P5.19  Functional  T2  Per remote rack: one summarized /56 route (not individual /64s)
P5.20  Functional  T1  OPTE dataplane executes at kernel/semi-privileged level
P5.21  Functional  T2  Every packet through OPTE receives classification before any transformation
P5.22  Functional  T3  OPTE maintains per-connection firewall state (established connections use cached state)
P5.23  Functional  T3  OPTE NAT: per-connection state table; return traffic rewritten using same entry
P5.24  Functional  T2  OPTE intercepts guest ARP for gateway, synthesizes valid reply locally
P5.25  Functional  T2  Guests obtain config via DHCP/SLAAC without static configuration [cross-fn]
P5.26  Functional  T2  Geneve outer source port: deterministic hash of inner flow 5-tuple (consistent per flow)
P5.27  Functional  T3  OPTE evaluates VPC routing before VPC firewall; both must pass before forwarding
P5.28  Functional  T2  After encap, inner dst MAC rewritten to target VNIC MAC
P5.29  Functional  T3  NAT port allocation checked against instance quota; over-quota denied
P5.30  Functional  T3  Each NAT allocation assigns unique (external IP, port) — no conflicts with active mappings
P5.31  Functional  T2  Boundary Services drops packets for unmapped IPs (no state table entry → drop)
P5.32  Functional  T3  Floating IP: OPTE applies 1:1 NAT (dst IP rewrite, port preserved)
P5.33  Functional  T2  Intra-rack destination: two equal-cost next-hops via both switches
P5.34  Functional  T2  Switch forwards directly to destination server when directly connected
P5.35  Functional  T2  Remote rack: each switch maintains >= 2 ECMP paths
P5.36  Functional  T2  OPTE intercepts ARP for gateway; request never forwarded to physical network
P5.37  Functional  T2  DNS packets to reserved service address intercepted by OPTE, queued for control plane [cross-fn]
P5.38  Functional  T1  Control plane addresses derived from server /64, not instance/VM identifiers
P5.39  Functional  T2  Consensus service instances advertise individual /64 routes; count is odd (3, 5, or 7)
P5.40  Functional  T1  NIC offloads: inner/outer IPv4/UDP/TCP checksums + TSO for encapsulated packets
P5.41  Functional  T2  OPTE state serializable for live migration (serialize on A, deserialize on B, states equal) [cross-fn]
P5.42  Performance T2  OPTE tracks per-instance bandwidth and enforces configured limits
P5.43  Functional  T2  VPC traffic encapsulated in Geneve with VNI matching VPC's assigned ID
P5.44  Functional  T2  Outbound NAT: inner source IP rewritten from private VPC to external NAT address
P5.45  Functional  T3  Inbound NAT reply: dst IP:port rewritten from external to original private using state table
P5.46  Functional  T1  Boundary Services implements BGP for customer network integration [cross-fn]
P5.47  Functional  T2  Inbound floating IP: inner dst MAC set to target VNIC MAC [cross-fn]
P5.48  Functional  T1  VLAN-tagged frames use Ethertype 0x8100 (802.1Q)
P5.49  Functional  T1  Each Geneve TLV option starts with 4-byte option header
P5.50  Functional  T1  Each server contributes exactly 2 MACs to switch table (SP + host NIC)
P5.51  Functional  T2  Servers autonomously compute own /64 and advertise via routing protocol (no DHCP/SLAAC)
P5.52  Functional  T2  Guest gateway address is off-subnet (forces all non-local traffic through OPTE)
P5.53  Functional  T2  Boundary Services: state table maps external IPs to internal VPC locations; every inbound packet looked up
```

### Non-Extractable Candidates (22)

**Vague (10):** "Feel performant" (no threshold), "Handle failures with degradation, no complete loss" (no degradation bound), "Minimize spurious TCP resets" (no threshold), "Scale without bottlenecks" (no bound), "Accommodate existing allocations" (too vague), "No network constraints limiting placement" (aspirational), "Automated without manual config" (process/vague), "Accommodate future changes with minimal impact" (unbounded), "Routing updates should prevent packets reaching failed switches" (no convergence time), "Degraded bandwidth acceptable during failures" (no threshold).

**Meta (6):** "Sizing from cloud provider docs," "leverage LLDP," "neutral incident analysis," "end-to-end alerts," "current design won't satisfy all needs," "sizing limits is difficult."

**External (4):** "No mandatory encryption between instances" (Google Andromeda observation), "10-20K servers per segment" (range, external density), "Most intra-cluster traffic unencrypted" (Andromeda), "Network acts as traffic forwarder" (philosophical).

**Incomplete (2):** "MTU section initially forgotten" (explicitly incomplete), "Estimated capacity per /48: 10-20K servers" (vague range).

### Metrics

| Metric | Value |
|---|---|
| Total candidates | 75 |
| Extractable | 53 |
| Rate | 70.7% |
| Functional / Performance | 52 / 1 |
| T1: 19, T2: 24, T3: 10, T4: 0, T?: 0 | |
| Numeric threshold rate | 13.2% (7/53) |
| Cross-function rate | 17.0% (9/53) |

### Hard Decisions

1. **P5.22/P5.23 (stateful firewall/NAT): Tier 2 vs Tier 3.** The connection-oriented model requires reasoning about temporal ordering and state transitions (first packet creates state, subsequent packets use cached state). The mechanism, not just the outcome, matters. Decision: Tier 3.

2. **N1-N9 (goal-level requirements): Extractable vs non-extractable.** Seven goals sound like requirements but lack numeric bounds. "Handle failures with degradation" could be formalized, but the RFD treats them as design goals for subsequent RFDs. Decision: Reject all as vague.

3. **P5.42 (bandwidth enforcement): Functional vs Performance.** The existence of enforcement is functional, but the property (packets above bandwidth limit are dropped) involves rate measurement. Decision: Performance.
