# Witness Week 0: Property Extraction Protocol

**Purpose:** Define the procedure, scoring rubric, and decision criteria for the manual property extraction pilot described in WITNESS-DESIGN-DOCUMENT.md, Section 6 (Week 0: Extraction Pilot).

**Duration:** 2–3 days.

**Prerequisite:** Read the Witness design document through Section 3 (Architecture) before beginning extraction. The definitions below assume familiarity with the four verification tiers and the functional/performance property distinction.

---

## 1. Definitions

### 1.1 Verifiable Property

A **verifiable property** is a statement extracted from a design document that satisfies all three criteria:

1. **Bounded:** It refers to a specific, finite aspect of system behavior (not "the system should be fast").
2. **Decidable:** A procedure exists — test, static check, symbolic execution, or proof — that produces a PASS/FAIL verdict for a given implementation.
3. **Traceable:** The property can be linked to a specific clause, sentence, or table cell in the source document.

A property need not contain an explicit numeric threshold to be verifiable. "The system must not accept requests without authentication" is verifiable (Tier 1 or Tier 3). "The system should be reliable" is not.

### 1.2 Extractable vs. Non-Extractable

| Classification | Definition | Example |
|---|---|---|
| **Extractable** | The document sentence can be mechanically rewritten as a property with a PASS/FAIL procedure | "Rate limit: 100 requests per second per client" |
| **Non-extractable (vague)** | The sentence expresses intent but lacks sufficient precision for a decidable check | "The system should handle high load gracefully" |
| **Non-extractable (meta)** | The sentence describes process, organization, or intent rather than system behavior | "The team will review performance quarterly" |
| **Non-extractable (external)** | The property depends on systems outside the specified scope | "The upstream DNS provider will resolve within 5ms" |

When in doubt, attempt to write the property as a test assertion. If you can write the assertion, the property is extractable. If you cannot without inventing information not present in the document, it is not.

### 1.3 Functional vs. Performance

| Type | Definition | Verification depends on... | Verdict format |
|---|---|---|---|
| **Functional** | True or false regardless of hardware, OS, load, or timing | Only the code under test | PASS / FAIL |
| **Performance** | True or false only relative to a specific hardware configuration, load profile, or timing window | Code + environment + load | PASS / FAIL *on [environment spec]* |

**Classification rules:**

- If the property contains units of time (ms, s), throughput (req/s, MB/s, IOPS), or resource consumption (memory, CPU%) → **Performance** (unless the time bound is a logical timeout, e.g., "session expires after 30 minutes of inactivity" is functional).
- If the property describes what the system *does* or *does not do* regardless of load → **Functional**.
- If the property describes *how fast* or *how much* the system handles → **Performance**.
- If ambiguous, classify as performance (conservative — avoids overclaiming deterministic verification on environment-dependent properties).

### 1.4 Verification Tiers

| Tier | Name | Assign when the property... | Examples |
|---|---|---|---|
| **1** | Static analysis | Can be checked by type system, import analysis, or AST inspection without executing code | Type constraints, nullability, enum membership, forbidden imports, structural shapes |
| **2** | Property-based testing | Describes a relationship between inputs and outputs testable with random/generated inputs | "Output is always sorted," "No negative balances," "Response contains all requested fields" |
| **3** | Symbolic execution | Requires path-sensitive reasoning, state machine analysis, or exhaustive branch coverage | "No code path allows unauthenticated access," "Every allocated resource is freed on every path," "State machine never enters [forbidden state]" |
| **4** | SMT proof | Requires universal quantification, mathematical invariants, or cryptographic properties | "For all inputs satisfying P, the output satisfies Q," "The hash function is collision-resistant under [model]" |

**Tier assignment rules:**

- Assign the **lowest sufficient tier.** If Tier 1 can verify it, do not assign Tier 2.
- If the property spans multiple tiers (e.g., "type is correct AND value is bounded"), assign the **highest required tier** — the pipeline verifies lower tiers as prerequisites.
- If you cannot determine the tier, mark it **T?** and record your reasoning. Ambiguous tier assignments are data, not errors.

---

## 2. Pilot Document Selection

### 2.1 Selection Criteria

Documents must satisfy all of the following:

1. **External authorship.** Not written by the Witness project author. This prevents selection bias from documents optimized for the Design Document Standard.
2. **Real engineering intent.** Written to propose or specify a real system, not as a tutorial, example, or demonstration.
3. **Public availability.** Accessible without credentials, enabling reproducibility.
4. **System scope.** Describes a software or hardware system with behavioral requirements (not a policy document, process guide, or organizational proposal).
5. **Non-trivial length.** At least 1,000 words of technical content (excluding headers, metadata, boilerplate).

### 2.2 Recommended Pilot Documents

The following Oxide Computer RFDs are recommended based on preliminary assessment of property density. Oxide RFDs are real engineering documents written in AsciiDoc by practicing engineers, publicly available at `rfd.shared.oxide.computer`, and authored independently of the Design Document Standard.

| # | Document | Domain | Preliminary property estimate | Why selected |
|---|---|---|---|---|
| 1 | [RFD 110](https://rfd.shared.oxide.computer/rfd/0110) | CockroachDB for Control Plane | 10–15 | Exceptionally rich in empirically measured properties: replication factors, latency bounds, recovery times, clock synchronization tolerances. Contains benchmark results with specific numbers. |
| 2 | [RFD 445](https://rfd.shared.oxide.computer/rfd/0445) | Crucible Upstairs Backpressure | 8–12 | Most specification-like of all candidates. Defines explicit activation thresholds with precise numeric boundaries and mathematically derived fault detection timing. |
| 3 | [RFD 490](https://rfd.shared.oxide.computer/rfd/0490) | Packed Crucible Extents | 8–10 | Contains before/after benchmark comparisons with specific throughput numbers and space efficiency calculations. Claims are directly verifiable. |
| 4 | [RFD 58](https://rfd.shared.oxide.computer/rfd/0058) | Rack Switch Architecture | 10–14 | Hardware-bound properties: port counts, bandwidth capacities, power budgets, oversubscription ratios. Numbers constrained by physics and silicon specs. |
| 5 | [RFD 63](https://rfd.shared.oxide.computer/rfd/0063) | Network Architecture | 8–12 | Addressing scheme with mathematically derivable capacity bounds. IPv6 hierarchy defines exact maximum counts per tier. |

**Minimum pilot size:** 3 documents. **Recommended:** all 5.

**Alternative sources** (if Oxide RFDs prove unsuitable due to format or accessibility changes): Google's publicly shared design doc examples, Jepsen test reports (which contain explicit correctness properties), or CockroachDB design documents from their public RFC repository.

---

## 3. Extraction Procedure

### 3.1 Per-Document Steps

For each pilot document, perform the following in order:

**Step 1: Read the full document.** Do not extract during the first read. Note the document's scope, the system it describes, and the type of claims it makes.

**Step 2: Second pass — mark candidate sentences.** On the second read, highlight or mark every sentence, table cell, or list item that contains a claim about system behavior. At this stage, include anything that *might* be a property. Over-inclusion is better than under-inclusion; filtering happens in Step 3.

**Step 3: Filter candidates.** For each marked sentence, apply the three-criteria test from Section 1.1:
- Is it bounded? → If no, mark as non-extractable (vague) and record.
- Is it decidable? → If no, mark as non-extractable and record reason.
- Is it traceable? → If no (e.g., implied across multiple paragraphs with no single anchor), attempt to synthesize a single property statement with a citation range. If the range exceeds one section, mark as non-extractable (diffuse).

**Step 4: Classify surviving properties.** For each extractable property:
- Assign **functional** or **performance** per Section 1.3 rules.
- Assign **tier** (1–4 or T?) per Section 1.4 rules.
- Write the property as a **one-line assertion** in the format: `[ID] [Type] [Tier] — [Property statement]`

**Step 5: Record cross-function properties.** Flag any property that spans multiple functions or components (e.g., "end-to-end latency < 200ms" involves multiple layers). These are important data — they predict integration testing requirements.

**Step 6: Record ambiguous cases.** For each property where type or tier classification was uncertain, write a brief note explaining the ambiguity. These notes feed directly into the extraction tool's disambiguation logic.

### 3.2 Property Statement Format

Each extracted property should be recorded in this format:

```
ID:       P<document-number>.<property-number>  (e.g., P1.03)
Source:   <section name or heading> — <quoted source text or summary>
Type:     Functional | Performance
Tier:     1 | 2 | 3 | 4 | T?
Assertion: <one-line decidable statement>
Cross-fn: Yes | No
Notes:    <optional — ambiguity, assumptions, tier rationale>
```

**Example (from RFD 110 — CockroachDB):**

```
ID:       P1.01
Source:   "CockroachDB Configuration" — "default replication factor of 3"
Type:     Functional
Tier:     1
Assertion: Database configuration sets replication_factor = 3
Cross-fn: No
Notes:    Verifiable by inspecting cluster configuration. Tier 1 because
          it is a static configuration property, not runtime behavior.
```

```
ID:       P1.02
Source:   "Failure Modes" — "p95 latency jumps from 3ms to 100ms during
          schema changes"
Type:     Performance
Tier:     2
Assertion: Under schema change, p95 query latency ≤ 100ms
Cross-fn: Yes (involves query path + schema change path)
Notes:    Performance property — requires load generation on reference
          hardware. The "3ms to 100ms" defines both baseline and degraded
          ceiling. Tier 2 because it is testable with generated queries
          under schema change load.
```

```
ID:       P1.03
Source:   "Clock Synchronization" — "500ms max offset before node crash"
Type:     Functional
Tier:     3
Assertion: If clock_offset > 500ms, the node terminates (does not
           continue serving requests with stale time)
Cross-fn: No
Notes:    Functional because the behavior (crash vs. continue) is
          deterministic regardless of hardware. Tier 3 because verifying
          "all code paths respect the clock offset check" requires
          path-sensitive analysis.
```

---

## 4. Scoring Rubric

### 4.1 Per-Document Metrics

For each document, record:

| Metric | Definition |
|---|---|
| **Total candidates** | Number of sentences/cells marked in Step 2 |
| **Extractable properties** | Number surviving the Step 3 filter |
| **Extractability rate** | Extractable / Total candidates × 100% |
| **Functional count** | Properties classified as functional |
| **Performance count** | Properties classified as performance |
| **Tier distribution** | Count per tier (T1, T2, T3, T4, T?) |
| **Numeric threshold rate** | Properties with explicit numeric bounds / Extractable × 100% |
| **Testable rate** | Properties expressible as test or constraint / Extractable × 100% |
| **Cross-function rate** | Cross-function properties / Extractable × 100% |
| **Ambiguity count** | Properties with T? tier assignment or uncertain type |

### 4.2 Aggregate Metrics

After all documents are scored, compute:

| Metric | Definition | Purpose |
|---|---|---|
| **Documents yielding ≥5 properties** | Count and percentage | Primary Go/No-Go input |
| **Mean extractable properties per document** | Average across all documents | Predicts per-document yield |
| **Median extractability rate** | Median of per-document extractability rates | Central tendency (robust to outliers) |
| **Functional / Performance ratio** | Total functional / Total performance | Predicts verification cost distribution |
| **Tier distribution (aggregate)** | Weighted across all documents | Predicts which tiers carry the verification load |
| **Ambiguity rate** | Total T? / Total extractable × 100% | Predicts disambiguation complexity |

### 4.3 Inter-Rater Calibration (If Two Extractors Available)

If a second extractor is available, have both extract independently from the same document(s), then compute:

- **Agreement rate:** Properties identified by both / Union of properties identified by either × 100%
- **Classification agreement:** For shared properties, percentage with matching Type and Tier
- **Krippendorff's alpha:** ≥0.67 required for adequate reliability (per the design document's evaluation criteria)

If only one extractor is available, skip this step but note it as a limitation in the pilot report.

---

## 5. Go/No-Go Decision

### 5.1 Thresholds (from WITNESS-DESIGN-DOCUMENT.md, Section 6)

| Outcome | Condition | Action |
|---|---|---|
| **Go** | ≥70% of documents yield ≥5 extractable properties | Proceed to Week 1 build |
| **Go with caution** | 50–70% of documents yield ≥5 extractable properties | Proceed but raise K1 threshold from ≥80% to ≥70% |
| **No-Go** | <50% of documents yield ≥5 extractable properties | Halt. Revise thesis before build investment |

### 5.2 Secondary Signals

Even if the primary Go threshold is met, the following secondary signals inform build priorities:

| Signal | Implication |
|---|---|
| **Ambiguity rate > 30%** | The extraction tool will need a disambiguation module. Budget extra time in Weeks 1–2. |
| **Performance properties > 60% of total** | Most verification will require reference environments. Adjust Tier 2 infrastructure early. |
| **Tier 3/4 properties > 40% of total** | Verification costs will be high. Reassess cost model assumptions. |
| **Cross-function rate > 50%** | Integration-level verification dominates. Architecture must handle multi-function property graphs. |
| **Numeric threshold rate < 50%** | Many properties are qualitative-but-decidable. The extraction tool needs more than regex-for-numbers. |

### 5.3 Reporting

The pilot report is a table (not a document — per File Output Policy, the repository is for the project). Record it as a commit message or inline in the design document's Week 0 section. The table must contain:

1. Per-document metrics (Section 4.1) for each pilot document.
2. Aggregate metrics (Section 4.2).
3. Go/No-Go decision with rationale.
4. Secondary signals observed and their implications.
5. List of the five hardest extraction decisions encountered, with reasoning. These become test cases for the automated extractor.

---

## 6. Worked Example: RFD 445 (Crucible Upstairs Backpressure)

This section demonstrates the full extraction procedure on one candidate document to calibrate expectations and illustrate edge cases.

### 6.1 First Read Summary

RFD 445 describes a backpressure mechanism for Crucible's "Upstairs" component (the client-facing layer of Oxide's virtual disk system). When a Downstairs (backend storage node) falls behind, Upstairs must slow incoming I/O to prevent unbounded queue growth. The RFD proposes two complementary backpressure dimensions: bytes in flight and live job count.

### 6.2 Extracted Properties

```
ID:       P2.01
Source:   "Backpressure based on write bytes" — "1 GiB in-flight write bytes"
Type:     Functional
Tier:     2
Assertion: When total in-flight write bytes across all Downstairs reaches
           1 GiB, backpressure delay is applied to new write requests
Cross-fn: No
Notes:    The threshold is a configuration constant. The *behavior* (delay
          applied at threshold) is functional. Testing: issue writes until
          aggregate exceeds 1 GiB, assert delay > 0.

ID:       P2.02
Source:   "Backpressure based on write bytes" — "10ms delay at 2 GiB"
Type:     Performance
Tier:     2
Assertion: At 2 GiB in-flight write bytes, the applied delay is
           approximately 10ms (quadratic scaling from onset at 1 GiB)
Cross-fn: No
Notes:    Performance because the exact delay depends on scheduling
          precision. Tier 2: testable by measuring delay at known byte counts.

ID:       P2.03
Source:   "Backpressure based on live job count" — "5% of IO_OUTSTANDING_MAX"
Type:     Functional
Tier:     2
Assertion: Backpressure based on live job count activates when
           live_job_count > 0.05 * IO_OUTSTANDING_MAX
Cross-fn: No
Notes:    Functional — threshold activation is deterministic. Testable
          with synthetic job injection.

ID:       P2.04
Source:   "Backpressure based on live job count" — "maximum delay of 5ms"
Type:     Performance
Tier:     2
Assertion: The maximum delay applied by job-count backpressure does not
           exceed 5ms
Cross-fn: No
Notes:    Performance — delay measurement is environment-dependent.

ID:       P2.05
Source:   "Per-client limit" — "2,600 per-client in-progress job limit"
Type:     Functional
Tier:     1
Assertion: Each client connection is limited to ≤ 2,600 concurrent
           in-progress jobs
Cross-fn: No
Notes:    Tier 1 — verifiable by inspecting the client connection
          configuration or constant definition.

ID:       P2.06
Source:   "Fault threshold" — "IO_OUTSTANDING_MAX (57,000) triggers
          Downstairs fault"
Type:     Functional
Tier:     3
Assertion: When total live jobs for a Downstairs reaches IO_OUTSTANDING_MAX
           (57,000), Upstairs declares that Downstairs faulted
Cross-fn: Yes (involves Upstairs fault detection + Downstairs state)
Notes:    Functional — fault declaration is deterministic. Tier 3 because
          verifying that *all paths* leading to IO_OUTSTANDING_MAX trigger
          the fault requires path-sensitive analysis.

ID:       P2.07
Source:   "Analysis" — "47 seconds for small-write fault detection"
Type:     Performance
Tier:     2
Assertion: With small writes (backpressure active), time from Downstairs
           stall to fault detection ≤ 47 seconds
Cross-fn: Yes (timing depends on write rate + backpressure interaction)
Notes:    Performance — derived from queue depth / drain rate calculation.
          The 47s is a mathematical bound, but real timing is
          environment-dependent.

ID:       P2.08
Source:   "Analysis" — "107 days fault detection without byte-based
          backpressure"
Type:     Functional
Tier:     2
Assertion: Without byte-based backpressure, large writes can accumulate
           jobs for >100 days before triggering the job-count fault
           threshold (demonstrating necessity of the byte-based mechanism)
Cross-fn: No
Notes:    This is a negative property — it demonstrates why byte-based
          backpressure exists. Extractable as a simulation/calculation
          test confirming the queue model.
```

### 6.3 Scoring

| Metric | Value |
|---|---|
| Total candidates | 14 |
| Extractable properties | 8 |
| Extractability rate | 57% |
| Functional | 5 |
| Performance | 3 |
| Tier distribution | T1: 1, T2: 5, T3: 1, T4: 0, T?: 0 |
| Numeric threshold rate | 8/8 = 100% |
| Testable rate | 8/8 = 100% |
| Cross-function rate | 2/8 = 25% |
| Ambiguity count | 0 |

### 6.4 Edge Cases Encountered

1. **Derived vs. stated properties.** The "47 seconds" fault detection delay is derived from a mathematical model in the RFD, not stated as a requirement. Decision: extract it, because the document presents it as a verifiable consequence of the design. If the implementation yields a different delay, the design or implementation has a bug.

2. **Negative properties.** The "107 days without byte-based backpressure" is a property of the *absence* of a feature, not a requirement. Decision: extract it as a simulation test — it validates the motivation for the design, not the design itself.

3. **Configuration vs. behavior.** "IO_OUTSTANDING_MAX = 57,000" is a configuration constant. "Reaching IO_OUTSTANDING_MAX triggers a fault" is a behavioral property. Decision: extract the behavior (P2.06), not the constant alone. The constant is a parameter; the behavior is the property.

4. **Proposed alternatives.** The RFD mentions "100 MiB" as an alternative onset threshold. Decision: extract only the proposed/accepted value (1 GiB), not alternatives under discussion. Alternatives are design context, not specification.

---

## 7. Recording Template

Use this template for each pilot document. Copy and fill in.

```markdown
## Document: [Title]
**Source:** [URL or path]
**Word count:** [approximate]
**Domain:** [e.g., storage, networking, database]

### Extraction Results

| ID | Source Section | Type | Tier | Assertion | Cross-fn | Notes |
|----|---------------|------|------|-----------|----------|-------|
| P_.01 | | | | | | |
| P_.02 | | | | | | |
| ... | | | | | | |

### Document Metrics

| Metric | Value |
|---|---|
| Total candidates | |
| Extractable properties | |
| Extractability rate | |
| Functional / Performance | / |
| Tier distribution | T1: T2: T3: T4: T?: |
| Numeric threshold rate | |
| Testable rate | |
| Cross-function rate | |
| Ambiguity count | |

### Hard Decisions

1. [Describe a difficult extraction decision and how you resolved it]
```

---

## 8. Checklist

Before declaring the pilot complete:

- [ ] ≥3 documents extracted (≥5 recommended)
- [ ] Every extractable property has ID, Source, Type, Tier, and Assertion fields
- [ ] Non-extractable candidates are recorded with classification (vague, meta, external, diffuse)
- [ ] Per-document metrics computed for all documents
- [ ] Aggregate metrics computed
- [ ] Go/No-Go decision recorded with rationale
- [ ] Secondary signals assessed
- [ ] ≥5 hardest extraction decisions documented (these seed the automated extractor's test suite)
- [ ] Results recorded per Section 5.3 (commit message or inline update, not a standalone report file)
