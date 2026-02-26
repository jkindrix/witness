# Witness: Design-Document-Anchored Verified Code Synthesis

**Author:** Justin Kindrix
**Date:** 2026-02-26
**Status:** Draft
**Decision-maker:** Author
**Review deadline:** 2026-03-12

---

## 1. Thesis

**[Principle 1: Falsifiable Thesis | Principle 5: Verifiable Evidence]**

AI-generated code is mainstream and unverifiable. Formal verification works but requires formal specifications in languages most engineers do not know. The gap between these worlds is not technical — it is linguistic. Engineers already write structured specifications with quantitative rigor. They call them design documents.

Witness is a system that takes a design document conforming to the Design Document Standard and produces a *verified package* — source code, tests, property proofs, and a traceability matrix linking every generated artifact back to its originating specification clause — where generation and verification happen simultaneously. Proof failures do not produce error reports; they guide re-generation. The output is not "AI-generated code" in the contemporary sense (probabilistic, best-effort, requires human review). The output is code whose correctness properties are machine-checked against the author's own stated intentions.

**The falsifiable claim:** For TypeScript systems specified by a design document with quantitative kill criteria, cost model, and scope boundaries (as defined by the Design Document Standard), Witness-generated code will exhibit fewer defects per function than unverified AI-generated code (Copilot, Cursor, Claude Code baseline), measured by a blinded evaluation panel applying the design document's own kill criteria as the acceptance test. Specifically:

- **Property extraction accuracy:** Witness will correctly extract ≥80% of verifiable properties from a design document's kill criteria, cost model, and scope boundaries, as judged by the document's author.
- **Verification success rate:** ≥70% of extracted properties will be machine-verified (type-checked, property-tested, or SMT-proven) without human intervention.
- **Defect rate reduction:** Witness-generated code will have ≥40% fewer defects than unverified AI-generated code on the same specification, where "defect" means a violation of any property stated in the design document.
- **Cost efficiency:** The end-to-end cost of producing a verified function will be <$0.50 per function at Tier 2 (property testing) and <$5.00 per function at Tier 4 (SMT), compared to an estimated $20–$200 per function for LLM-assisted Dafny specification with human review (derived from ATLAS pipeline costs scaled to include human oversight, and cross-referenced with vericoding benchmark per-problem costs).

**Preliminary validation required.** The claim that design documents contain "formal-enough" properties is the load-bearing assumption of this entire system. Before committing to the 6-week build, a manual extraction pilot (Week 0, described in Section 6) must demonstrate that real-world design documents — not purpose-written examples — yield extractable properties at a rate that makes the system useful. If the pilot shows <50% extractability on existing documents, the thesis requires revision before the build begins.

These claims will be tested in a controlled evaluation described in Section 5.

---

## 2. Prior Art & Landscape

**[Principle 3: Gap Analysis | Principle 5: Verifiable Evidence]**

### The Verification Spectrum

The field of AI code generation exists on a spectrum from informal to formal, from unverified to verified. As of early 2026, a clear pattern has emerged: the tools occupy the extremes, and the middle is empty.

**Unverified generation from informal specifications** is the dominant paradigm. GitHub Copilot, Cursor, Claude Code, and similar tools take natural language prompts or inline comments and produce code that may or may not be correct. The user is the verifier. The implicit contract is "this looks plausible; you check it." A 2025 study by Bursuc et al. confirmed that including natural language descriptions alongside formal specifications produced no significant improvement in verification success rates (arXiv:2509.22908, POPL 2026) — the informal descriptions were noise, not signal.

**Verified generation from formal specifications** is an active research frontier. The vericoding benchmark (Bursuc et al., 2025) demonstrated that off-the-shelf LLMs can generate formally verified code from formal specifications at 82.2% success in Dafny, 44.3% in Verus/Rust, and 26.8% in Lean, using a union of all models tested (Claude Opus 4.1, GPT-5, Gemini 2.5 Pro, and others). Progress on Dafny verification alone accelerated from 68% (June 2024) to 96% (2025). The ATLAS toolkit (Baksys et al., arXiv:2512.10173, POPL 2026) generated 2,700 verified Dafny programs and showed that fine-tuning Qwen 2.5 7B on the resulting 19,000 training examples improved DafnyBench performance by 23 percentage points and DafnySynthesis by 50 percentage points.

**The empty middle.** No tool currently takes *structured natural-language specifications* — documents with quantitative thresholds, explicit scope boundaries, falsifiable claims, and cost models — and produces verified code. The structured design document is a specification format that already exists in engineering practice, already contains formal-enough properties for machine verification, and requires no knowledge of Dafny, Lean, or Verus to write. Yet no system treats it as an input format for verified synthesis.

### Gap Analysis Matrix

|                           | Informal Spec (NL prompt)           | Structured Spec (Design Document)          | Formal Spec (Dafny/Lean/Verus)              |
|---------------------------|-------------------------------------|--------------------------------------------|---------------------------------------------|
| **Unverified Generation** | Copilot, Cursor, Claude Code, etc.  | *No tool uses design docs as input*        | N/A (formal specs imply verification)       |
| **Verified Generation**   | *Empty — specs too vague to verify* | **Witness occupies this cell**             | Vericoding (82% Dafny, 27% Lean — active research) |

### The "Do Nothing" Baseline

Without Witness, the trajectory is predictable. AI code generation volume will continue increasing — it already accounts for a majority of new code at many organizations. Verification will remain a manual, post-hoc activity: code review by humans who are increasingly reviewing code they did not write, using specifications they must reconstruct from context. The vericoding research line will continue advancing formal verification for those who can write formal specifications, which Congdon estimates at "maybe a few hundred people in the world" (Congdon, "The Coming Need for Formal Specification," December 2025). The gap between code volume and verification capacity will widen.

Kleppmann identified this dynamic precisely: "Soon the limiting factor will not be the technology, but the culture change required for people to realise that formal methods have become viable in practice" (Kleppmann, "Prediction: AI will make formal verification go mainstream," December 2025). Witness addresses the culture change problem by meeting engineers where they already are — writing design documents — rather than requiring them to learn where formal methods practitioners already are.

### Adjacent Work

Several systems address parts of the problem without occupying Witness's specific cell:

- **Atlas Computing's FV Toolchain** (atlascomputing.org, 2025) decomposes formal verification into GenerateAndCheck, CorrectByConstruction, ProgramRepair, and ProgramEquivalence stages. Their Specification IDE maps formal specification sections to natural language, but takes formal specifications as input, not design documents.

- **Astrogator** (Councilman et al., arXiv:2507.13290, 2025) provides formal verification of LLM-generated code for Ansible, achieving 83% verification and 92% error detection. Its Knowledge Base reduces domain expertise requirements, but it targets a single domain (system administration) and requires a formal query language rather than design documents.

- **Harmonic's Aristotle** ($1.45B valuation, November 2025) and **Logical Intelligence** focus on mathematical reasoning and proof generation in Lean, targeting theorem proving rather than general software synthesis from structured specifications.

- **DeepSeek-Prover-V2** and **Seed-Prover** (ByteDance) have achieved 88.9% and 99.6% on MiniF2F-test respectively, demonstrating that LLMs can write proofs. But these systems prove mathematical theorems, not software properties derived from engineering specifications.

- **JetBrains Research** ("Can LLMs Enable Verification in Mainstream Programming?", March 2025) tested Claude 3.5 Sonnet on Dafny, Nagini, and Verus, finding promising results for making verification accessible to mainstream programmers. This is the closest prior work to Witness's goal — making verification accessible — but it still requires the programmer to work within a formal verification language. Witness removes that requirement entirely.

- **The "test, don't just verify" counterargument** (Keleş, December 2025) argues that property-based testing may be the right level of assurance for most code, and that full formal verification is unnecessary for non-critical systems. This argument is actually *favorable* to Witness's architecture: Tier 2 (property-based testing) is the cheapest and most widely applicable tier. Witness agrees that not everything needs SMT proofs — that is precisely why the cost gradient exists.

None of these systems take a design document and produce a verified package. Witness is not competing with them — it is composing their capabilities at a different abstraction level.

---

## 3. Architecture

**[Principle 4: Scope Boundaries | Principle 6: Architecture as Cost Gradient | Principle 7: Production-Grade Example]**

### Scope Boundaries

Witness is:

- **A verified synthesis system** that takes design documents and produces verified packages (code + tests + property proofs + traceability matrix).
- **A TypeScript-first tool** in V1. TypeScript has the largest design document authorship base, the strongest ecosystem for property-based testing (fast-check), and strong type-level verification capabilities for Tier 1.
- **A composition of known techniques, not novel research.** Witness does not invent a new LLM, a new proof assistant, or a new code representation. It combines property-based testing, content-addressed ASTs, LLM orchestration, and symbolic execution into a pipeline purpose-built for design-document-anchored verification.

Witness is **not**:

- **A proof assistant.** It does not help users write formal proofs. It generates proofs from extracted properties and uses proof failures to guide re-generation.
- **A replacement for formal methods in safety-critical systems.** Avionics, medical devices, and nuclear systems require DO-178C / IEC 62304 compliance with full formal specification in languages designed for that purpose. Witness targets the vast middle ground of software that would benefit from verification but will never receive formal specifications.
- **A design document linter or authoring tool in V1.** It consumes design documents; it does not help write them. (V2 may add property-preview-during-authoring as a feedback mechanism, but V1 is strictly consumption-only.)
- **A general-purpose code generator.** It generates code only for systems that have a conforming design document. No document, no generation.

### The Verification Cost Gradient

The central architectural insight is that not all properties require the same level of verification, and the cost of verification should be proportional to the criticality of the property being verified. Witness implements four tiers in a graduated verification pipeline:

| Tier | Verification Method | Handles | Latency | Cost per Function | Confidence |
|------|---------------------|---------|---------|-------------------|------------|
| **Tier 1:** Static analysis | TypeScript strict mode + branded types + import/AST analysis | ~40% of properties (type constraints, nullability, enum membership, forbidden imports, structural constraints) | <1s | $0.00† | Deterministic |
| **Tier 2:** Property-based | fast-check property tests (1,000+ random inputs) | ~30% of properties (input/output relationships, invariants, boundary conditions) | 2–10s | $0.02–0.10 | Probabilistic (high) |
| **Tier 3:** Symbolic | Symbolic execution with constraint solving | ~20% of properties (path coverage, edge case exhaustion, state machine transitions) | 10–60s | $0.50–2.00 | Deterministic (for covered paths) |
| **Tier 4:** SMT | Z3 SMT solver | ~10% of properties (universal quantification, mathematical invariants, security properties) | 30s–5min | $2.00–5.00 | Deterministic (mathematical proof) |

The percentages represent the expected distribution of properties extracted from a typical design document. A design document's scope boundaries ("not multi-tenant in V1") map to Tier 3 symbolic checks that no tenant-crossing paths exist.

### Property Classification: Functional vs. Performance

Not all properties have the same epistemological status. Witness distinguishes two categories that require different verification strategies:

**Functional properties** are environment-independent. "No code path accepts a tenant identifier parameter" is true or false regardless of the hardware, OS, or load. These properties produce deterministic PASS/FAIL verdicts.

**Performance properties** are environment-dependent. "checkRateLimit completes in <1ms" depends on CPU speed, memory bandwidth, GC pressure, and concurrent load. A property test asserting `elapsed < 1ms` passes on an M3 Max and fails on a Raspberry Pi. These are not properties of the *code* — they are properties of the *code-on-hardware pair*.

Witness handles this distinction explicitly:

| Property Type | Verification Strategy | Verdict Format | Confidence Qualifier |
|---|---|---|---|
| **Functional** | Property test, symbolic execution, or SMT | PASS / FAIL (deterministic) | "Verified" |
| **Performance** | Benchmark on reference hardware with specified tolerance | PASS / FAIL *on reference config* | "Verified on [hardware spec] ± [tolerance]" |

Performance properties in the traceability matrix carry an environment tag: the hardware specification, OS, and runtime version under which the benchmark was executed. If the deployment environment differs materially from the reference environment, performance properties are flagged as "requires re-verification" rather than claimed as verified.

This distinction matters because design documents routinely contain both types. Kill criteria are often performance properties ("if latency exceeds 200ms"); scope boundaries are often functional properties ("not distributed in V1"). Conflating them would overstate the confidence level of performance claims.

The cost gradient means that a typical design document with 20 extractable properties would cost approximately:

```
8 properties  × $0.00 (Tier 1)  = $0.00
6 properties  × $0.06 (Tier 2)  = $0.36
4 properties  × $1.25 (Tier 3)  = $5.00
2 properties  × $3.50 (Tier 4)  = $7.00
─────────────────────────────────────────
Total: ~$12.36 for 20 verified properties
```

Compare this to LLM-assisted formal specification with human review: the ATLAS pipeline (Baksys et al., 2025) generated 2,700 verified Dafny programs at scale, implying a per-program pipeline cost on the order of $1–$5 for the automated portion. Adding human review of generated specifications and proofs brings the *estimated* cost to $20–$200 per function for non-trivial programs — this is an estimate derived from ATLAS pipeline costs plus assumed review overhead, not a measured cost from a production workflow. The lower bound ($20) is optimistic and likely applies only to simple functions; the upper bound ($200) is more realistic for functions requiring meaningful specification review. For comparison, the seL4 project (20 person-years for 8,700 lines of verified microkernel code) implies $500+ per function — but seL4 is an extreme case, not a representative baseline. At the ATLAS-derived estimate, Witness achieves roughly one order of magnitude cost reduction by operating at a lower but still rigorous level of formality, while requiring no formal specification expertise from the user.

### System Pipeline

```
┌─────────────────┐
│  Design Document │
│  (Markdown + DDS)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐    Properties: { name, source_clause,
│   Property       │    tier, predicate, threshold }
│   Extractor      │───────────────────────────────────┐
│  (LLM + parser)  │                                   │
└────────┬────────┘                                   │
         │                                             │
         ▼                                             │
┌─────────────────┐                                   │
│   Code           │    Code: content-addressed        │
│   Generator      │    AST graph (BLAKE3)             │
│  (LLM)           │                                   │
└────────┬────────┘                                   │
         │                                             │
         ▼                                             │
┌─────────────────┐                                   │
│   Graduated      │◄──────────────────────────────────┘
│   Verifier       │    Checks code against properties
│  (Tiers 1–4)     │    at appropriate tier
└────────┬────────┘
         │
         ├── PASS ──► Traceability Matrix + Verified Package
         │
         └── FAIL ──► Counterexample + Fix Guidance
                      │
                      ▼
              ┌───────────────┐
              │  Re-generation │
              │  (LLM +        │
              │   counterex.)  │
              └───────┬───────┘
                      │
                      └──► Back to Verifier (max 5 iterations)
```

### Witness Components

| Component | What It Does | Key Technique |
|---|---|---|
| Property Extractor | Extracts verifiable properties from design document sections | LLM-driven parsing + JSON schema validation |
| Code Generator | Generates TypeScript from extracted properties + design context | LLM code generation with property-aware prompting |
| Code Representation | Canonical AST representation; structural validation; semantic mutation | Content-addressed AST graph (BLAKE3 hashing) |
| Graduated Verifier | Four-tier verification with cost gradient | tsc + fast-check + symbolic execution + Z3 |
| Output Materializer | Atomic output of verified package to disk | Transactional filesystem write |
| Traceability Engine | Links every artifact to its originating design document clause | Property → code → test → proof mapping |

All components are built as part of Witness. There are no external project dependencies beyond standard libraries (TypeScript, fast-check, Z3).

### Author's Related Work

The author has built systems in adjacent problem spaces. These inform Witness's architecture but are not dependencies:

| Project | Domain | What it informed |
|---|---|---|
| **Axiom** | Graph-first, content-addressed AST representation | The content-addressed hashing approach (BLAKE3 of AST nodes) for incremental re-verification |
| **Scriblr** | LLM orchestration with multi-provider support | Prompt engineering patterns for structured extraction from natural language |
| **Exemplary** | Pattern library (103 documented software patterns) | Domain-aware code generation strategies |
| **Blood** | Programming language with correctness philosophy | Graduated verification concept (Proposal 7: runtime contracts → compile-time verification → full proofs) |
| **tree2repo** | Specification-to-filesystem generation | Atomic output materialization |

The most critical build item is the Graduated Verifier. If the property-testing tier fails to handle design-document-derived properties, the pipeline stalls. The Week 0 pilot validated that the *properties themselves* are extractable; Weeks 3–4 will validate that they are *verifiable*.

### Production-Grade Example

**Input:** A section from a design document for a rate limiter service.

```markdown
## Kill Criteria

If the rate limiter does not reject requests exceeding the configured
threshold within 50ms of the threshold being reached, the approach is
invalidated. If memory usage per rate-limited key exceeds 1KB, the
sliding window implementation must be replaced with a fixed window.
If the system cannot handle 10,000 rate-limit checks per second on a
single core, the design is not viable for the target deployment.

## Cost Model

| Operation | Budget |
|-----------|--------|
| Check rate limit | <1ms |
| Record request | <500μs |
| Evict expired keys | <10ms per 1,000 keys |

## Scope Boundaries

- Not distributed in V1. Single-node, in-memory only.
- Not persistent. State is lost on restart.
- Not multi-tenant. One rate limit configuration per instance.
```

**Extracted Properties:**

```json
[
  {
    "id": "P1",
    "name": "rejection_latency",
    "source": "Kill Criteria, sentence 1",
    "tier": 2,
    "predicate": "Time from threshold breach to rejection ≤ 50ms",
    "threshold": "50ms",
    "test_type": "property_test"
  },
  {
    "id": "P2",
    "name": "memory_per_key",
    "source": "Kill Criteria, sentence 2",
    "tier": 3,
    "predicate": "sizeof(RateLimitEntry) ≤ 1024 bytes",
    "threshold": "1KB",
    "test_type": "symbolic"
  },
  {
    "id": "P3",
    "name": "throughput",
    "source": "Kill Criteria, sentence 3",
    "tier": 2,
    "predicate": "≥10,000 checks/second on single core",
    "threshold": "10000 ops/s",
    "test_type": "property_test"
  },
  {
    "id": "P4",
    "name": "check_latency",
    "source": "Cost Model, row 1",
    "tier": 2,
    "predicate": "checkRateLimit() completes in <1ms",
    "threshold": "1ms",
    "test_type": "property_test"
  },
  {
    "id": "P5",
    "name": "record_latency",
    "source": "Cost Model, row 2",
    "tier": 2,
    "predicate": "recordRequest() completes in <500μs",
    "threshold": "500μs",
    "test_type": "property_test"
  },
  {
    "id": "P6",
    "name": "eviction_latency",
    "source": "Cost Model, row 3",
    "tier": 2,
    "predicate": "evictExpired(n) completes in <10ms for n≤1000",
    "threshold": "10ms per 1000 keys",
    "test_type": "property_test"
  },
  {
    "id": "P7",
    "name": "no_distributed_state",
    "source": "Scope Boundaries, bullet 1",
    "tier": 1,
    "predicate": "No network imports; no IPC; no shared memory APIs",
    "threshold": "0 occurrences",
    "test_type": "type_check"
  },
  {
    "id": "P8",
    "name": "no_persistence",
    "source": "Scope Boundaries, bullet 2",
    "tier": 1,
    "predicate": "No fs.write, no database connections, no disk I/O",
    "threshold": "0 occurrences",
    "test_type": "type_check"
  },
  {
    "id": "P9",
    "name": "single_tenant",
    "source": "Scope Boundaries, bullet 3",
    "tier": 3,
    "predicate": "No code path accepts a tenant identifier parameter",
    "threshold": "0 paths",
    "test_type": "symbolic"
  }
]
```

**Generated Code** (abbreviated — full output would include complete implementation):

```typescript
// witness:verified P7,P8 (Tier 1: type-level)
// witness:verified P1,P3,P4,P5,P6 (Tier 2: property-tested, 1000 inputs each)
// witness:verified P2,P9 (Tier 3: symbolic execution)
// witness:trace Kill Criteria → P1,P2,P3 | Cost Model → P4,P5,P6 | Scope → P7,P8,P9

interface RateLimitConfig {
  readonly maxRequests: number;
  readonly windowMs: number;
}

interface RateLimitEntry {
  readonly timestamps: number[];  // Sliding window of request timestamps
}

class SlidingWindowRateLimiter {
  private readonly config: RateLimitConfig;
  private readonly entries: Map<string, RateLimitEntry> = new Map();

  constructor(config: RateLimitConfig) {
    this.config = Object.freeze(config);
  }

  /**
   * Check if a key has exceeded its rate limit.
   * @returns true if the request is allowed, false if rate-limited.
   *
   * witness:property P4 — completes in <1ms
   * witness:property P1 — rejects within 50ms of threshold breach
   */
  checkRateLimit(key: string): boolean {
    const now = Date.now();
    const entry = this.entries.get(key);
    if (!entry) return true;

    const windowStart = now - this.config.windowMs;
    const recentRequests = entry.timestamps.filter(t => t > windowStart);
    return recentRequests.length < this.config.maxRequests;
  }

  /**
   * Record a request for a key.
   *
   * witness:property P5 — completes in <500μs
   */
  recordRequest(key: string): void {
    const now = Date.now();
    const entry = this.entries.get(key);
    if (entry) {
      entry.timestamps.push(now);
    } else {
      this.entries.set(key, { timestamps: [now] });
    }
  }

  /**
   * Evict expired entries.
   *
   * witness:property P6 — completes in <10ms per 1000 keys
   */
  evictExpired(): number {
    const now = Date.now();
    const windowStart = now - this.config.windowMs;
    let evicted = 0;

    for (const [key, entry] of this.entries) {
      const recent = entry.timestamps.filter(t => t > windowStart);
      if (recent.length === 0) {
        this.entries.delete(key);
        evicted++;
      } else {
        this.entries.set(key, { timestamps: recent });
      }
    }
    return evicted;
  }
}
```

**Generated Property Tests** (abbreviated):

```typescript
import { fc } from "fast-check";

// P3: Throughput — ≥10,000 checks/second on single core
test("P3: throughput meets kill criteria", () => {
  const limiter = new SlidingWindowRateLimiter({
    maxRequests: 100,
    windowMs: 60_000,
  });

  const start = performance.now();
  for (let i = 0; i < 10_000; i++) {
    limiter.checkRateLimit(`key-${i % 100}`);
  }
  const elapsed = performance.now() - start;

  expect(elapsed).toBeLessThan(1_000); // 10k ops in <1s = >10k ops/s
});

// P1: Rejection latency — within 50ms of threshold breach
test("P1: rejection latency within 50ms of threshold", () => {
  fc.assert(
    fc.property(
      fc.integer({ min: 1, max: 1000 }),  // maxRequests
      fc.integer({ min: 100, max: 60000 }), // windowMs
      (maxRequests, windowMs) => {
        const limiter = new SlidingWindowRateLimiter({ maxRequests, windowMs });
        const key = "test";

        // Fill to threshold
        for (let i = 0; i < maxRequests; i++) {
          limiter.recordRequest(key);
        }

        // Measure rejection time
        const start = performance.now();
        const allowed = limiter.checkRateLimit(key);
        const elapsed = performance.now() - start;

        expect(allowed).toBe(false);
        expect(elapsed).toBeLessThan(50);
      }
    ),
    { numRuns: 1000 }
  );
});

// P4: Check latency — <1ms per call
test("P4: check latency under 1ms", () => {
  fc.assert(
    fc.property(
      fc.string({ minLength: 1, maxLength: 64 }),
      (key) => {
        const limiter = new SlidingWindowRateLimiter({
          maxRequests: 100,
          windowMs: 60_000,
        });

        const start = performance.now();
        limiter.checkRateLimit(key);
        const elapsed = performance.now() - start;

        expect(elapsed).toBeLessThan(1);
      }
    ),
    { numRuns: 1000 }
  );
});
```

**Traceability Matrix:**

| Property | Source Clause | Type | Tier | Verification Method | Result | Hash |
|----------|-------------|------|------|---------------------|--------|------|
| P1: rejection_latency | Kill Criteria §1 | Perf | 2 | Benchmark, 1000 runs | PASS on ref¹ (8ms p99) | `a7c3e1` |
| P2: memory_per_key | Kill Criteria §2 | Func | 3 | Symbolic size analysis | PASS (412 bytes) | `b2d4f0` |
| P3: throughput | Kill Criteria §3 | Perf | 2 | Benchmark, 1000 runs | PASS on ref¹ (14,200 ops/s) | `c9e8a3` |
| P4: check_latency | Cost Model §1 | Perf | 2 | Benchmark, 1000 runs | PASS on ref¹ (0.3ms p99) | `d1f7b2` |
| P5: record_latency | Cost Model §2 | Perf | 2 | Benchmark, 1000 runs | PASS on ref¹ (0.1ms p99) | `e4a6c5` |
| P6: eviction_latency | Cost Model §3 | Perf | 2 | Benchmark, 1000 runs | PASS on ref¹ (3.2ms/1000) | `f8b9d4` |
| P7: no_distributed_state | Scope §1 | Func | 1 | Import analysis | PASS (0 network imports) | `17c2e6` |
| P8: no_persistence | Scope §2 | Func | 1 | Import analysis | PASS (0 fs/db imports) | `28d3f7` |
| P9: single_tenant | Scope §3 | Func | 3 | Symbolic path analysis | PASS (0 tenant params) | `39e4a8` |

> ¹ **Reference environment:** Node.js 22.x, Apple M3 Max, 36GB RAM, macOS 15. Performance properties (P1, P3–P6) are verified on this reference configuration ± 2x tolerance. Deployment on materially different hardware requires re-verification.

The hash column contains content-addressed identifiers (BLAKE3, truncated to 6 hex characters for display; full 256-bit hashes used for verification and collision resistance). Each hash links a verified property to the specific AST node it was verified against. If the code changes, the hash changes, and re-verification is triggered for affected properties.

---

## 4. Detailed Design

**[Principle 7: Production-Grade Examples | Principle 8: Cost Model]**

### 4.1 Property Extractor

The Property Extractor transforms design document sections into a structured list of verifiable properties. It operates in two phases.

**Phase 1: Section Classification.** The extractor identifies which sections of the design document contain verifiable content. The Design Document Standard prescribes specific section types with known structure:

| Section Type | Verifiable Content | Extraction Strategy |
|---|---|---|
| Kill Criteria (P2) | Numeric thresholds with failure conditions | Parse "if [metric] [comparator] [threshold], then [consequence]" patterns |
| Cost Model (P8) | Per-operation budgets | Parse table rows as operation → latency/cost constraints |
| Scope Boundaries (P4) | Negative constraints ("not X in V1") | Parse exclusion statements as forbidden-pattern properties |
| Evaluation Plan (P11) | Success/failure thresholds | Parse experimental design metrics as property targets |
| Architecture tiers (P6) | Per-tier performance envelopes | Parse tier tables as bounded-cost properties |

**Phase 2: Property Synthesis.** For each identified verifiable clause, an LLM call generates the property specification in the structured format shown in the worked example. The LLM prompt includes the original clause, the section type, and a few-shot example set of clause → property mappings. The output is validated against a JSON schema before acceptance.

**Accuracy target:** ≥80% of properties correctly extracted on first pass, as judged by the document's author. Properties the LLM cannot extract are flagged for human review with the source clause highlighted.

**Cost:** One LLM call per design document section (typically 5–9 sections), plus one call per extracted property for tier classification. For a 10-page design document: approximately 15–25 LLM calls at $0.01–0.05 each = **$0.15–$1.25 per document**.

### 4.2 Code Generator

The Code Generator produces TypeScript source code that satisfies the extracted properties, using LLM code generation with property-aware prompting.

**Generation strategy:** The generator receives the full property list, the original design document sections referenced by each property, and pattern guidance for the domain. It generates code as a content-addressed AST graph rather than raw text, ensuring structural validity by construction — syntax errors are impossible because the graph representation enforces well-formedness.

**Re-generation loop:** When the Graduated Verifier reports a property failure, the generator receives:
1. The failing property specification
2. The counterexample (specific input that violated the property)
3. The current code as a content-addressed AST graph
4. Fix guidance from the verifier (which property of which function failed, and how)

The generator then applies a *semantic mutation* (targeted AST edit) rather than regenerating from scratch. This preserves verified properties while targeting the specific failure. Maximum 5 re-generation iterations per property before escalating to human.

**Cost:** 1–3 LLM calls per function (initial generation + 0–2 re-generations). For a typical rate limiter with 5 public functions: **$0.25–$1.50**.

### 4.3 Graduated Verifier

The Graduated Verifier is Witness's core verification engine, with a tier-assignment layer that routes each property to the cheapest tier that can verify it.

**Tier assignment rules:**

| If the property... | Then assign... |
|---|---|
| Can be expressed as a TypeScript type constraint | Tier 1 (type-level) |
| Is a relationship between inputs and outputs | Tier 2 (property-based) |
| Requires path-sensitive reasoning | Tier 3 (symbolic) |
| Requires universal quantification or mathematical proof | Tier 4 (SMT) |

**Tier ambiguity resolution.** Many properties could plausibly be verified at multiple tiers. A property like "no code path accepts a tenant identifier parameter" could be Tier 1 (static analysis of function signatures), Tier 2 (property test that no function accepts a tenantId), or Tier 3 (symbolic path analysis). The assignment rule is: **assign to the cheapest tier that provides sufficient confidence.** When the LLM-suggested tier and the verifier's actual capability disagree, the verifier's tier-escalation mechanism handles it automatically: if Tier 1 verification fails or is inconclusive, the property is promoted to Tier 2, then Tier 3. This "try cheap first, escalate on failure" strategy means tier misassignment costs time (unnecessary verification attempts) but not correctness.

**Verification is incremental.** When code changes (detected by content-addressed hashing of AST nodes), only properties whose referenced nodes have changed hashes are re-verified. This is the key performance optimization: a change to the `evictExpired` function re-verifies P6 but not P1 through P5.

### 4.4 Output Package

The verified package written atomically to disk contains:

```
<project>/
├── src/
│   └── rate-limiter.ts          # Generated source code
├── tests/
│   ├── properties.test.ts       # Generated property tests (Tier 2)
│   └── symbolic.test.ts         # Generated symbolic tests (Tier 3)
├── proofs/
│   └── rate-limiter.smt2        # SMT proofs (Tier 4, if any)
├── witness/
│   ├── properties.json          # Extracted property specifications
│   ├── traceability.json        # Property → source clause → code hash mapping
│   └── verification-report.json # Per-property verification results
└── package.json                 # With test scripts and dependencies
```

The `witness/` directory is the verification audit trail. The traceability matrix in `traceability.json` enables a reviewer to trace any line of generated code back to the design document clause that motivated it, and forward to the verification result that confirmed it.

### 4.5 Aggregate Cost Model

| Phase | Operations | Cost Range |
|---|---|---|
| Property Extraction | 15–25 LLM calls | $0.15–$1.25 |
| Code Generation | 5–15 LLM calls | $0.25–$1.50 |
| Tier 1 Verification | Static analysis (type checking + import/AST analysis) | $0.00† |
| Tier 2 Verification | Property tests (fast-check) | $0.12–$0.60 |
| Tier 3 Verification | Symbolic execution | $2.00–$8.00 |
| Tier 4 Verification | SMT solving | $4.00–$10.00 |
| Output Materialization | Filesystem write | $0.00 |
| **Total (typical document)** | | **$6.52–$21.35** |

> † Tier 1 verification cost is $0.00 for the verification step itself (running `tsc` and AST analysis). The cost of *generating* code that satisfies Tier 1 constraints is included in the Code Generation phase above. Tier 1 is not free — it is free to *check*.

For comparison:
- **Unverified AI generation** (Copilot/Cursor): $0.01–$0.10 per function, but with no verification.
- **LLM-assisted formal specification with human review** (Dafny): $20–$200 per function (ATLAS pipeline costs plus human oversight; upper bound from Kleppmann's seL4 extrapolation for complex code).
- **Witness:** $6.52–$21.35 per design document, covering all functions specified by the document.

Witness is 10–100x more expensive than unverified generation, but roughly one order of magnitude cheaper than LLM-assisted formal specification — and requires no formal specification expertise. The cost advantage is real but modest; the accessibility advantage (no Dafny/Lean knowledge required) is the primary differentiator.

---

## 5. Evaluation Plan

**[Principle 2: Kill Criteria | Principle 11: Evaluation as Experiment]**

### Kill Criteria

The following criteria determine whether Witness is viable. If any kill criterion is triggered, the corresponding component must be redesigned before proceeding.

| ID | Criterion | Threshold | Measurement | Kill Action |
|----|-----------|-----------|-------------|-------------|
| K1 | Property extraction accuracy | <60% correct (author-judged) | 10 design documents, 3 authors | Redesign extraction prompts and parser |
| K2 | Verification success rate | <50% of extracted properties verified | Same 10 documents | Reassess tier assignment or add Tier 2.5 |
| K3 | Re-generation convergence | >5 iterations without convergence on >30% of properties | Same 10 documents | Redesign counterexample-guided re-generation |
| K4 | Cost per document | >$50 for a typical 10-page document | Same 10 documents | Optimize tier assignment; reduce LLM calls |
| K5 | End-to-end time | >30 minutes for a typical document | Same 10 documents | Parallelize verification; reduce SMT scope |
| K6 | Defect rate vs. baseline | <20% improvement over unverified AI generation | Blinded evaluation (see below) | Fundamental approach may be wrong; halt project |

Kill criteria K1 and K6 are the existential gates. If design documents do not contain enough structure for reliable property extraction (K1), or if the extracted-and-verified properties do not translate to fewer defects in practice (K6), the core thesis is falsified.

**Interpreting K6 results:** The thesis targets a 40% weighted defect reduction; the kill criterion floor is 20%. A result in the 20–40% range passes K6 but represents a weaker value proposition than the thesis claims. In this "pass but underwhelming" scenario: proceed to V2 only if the defect reduction concentrates in Tier 3/4 properties (where Witness adds unique value) rather than Tier 1/2 properties (where competent developers rarely err). A 25% overall reduction driven entirely by catching Tier 3 invariant violations is more compelling than a 35% reduction driven by catching Tier 1 type errors.

### Evaluation Design

**Study type:** Blinded multi-arm evaluation.

**Arms:**

| Arm | Description | Code Source |
|-----|-------------|-------------|
| A (Control) | Unverified AI generation | Claude Code generating from the same design document, no verification |
| B (Treatment) | Witness-generated code | Full pipeline: extract → generate → verify → output |
| C (Reference) | Human-written code | Senior engineer implementing from the same design document |

**Procedure:**

1. Select 10 design documents of varying complexity (3 small / 4 medium / 3 large), written by 3 different authors, all conforming to the Design Document Standard.
2. For each document, produce code via all three arms. Strip all identifying markers (Witness annotations, author style indicators, AI generation artifacts).
3. A panel of 5 evaluators (senior engineers not involved in authoring) receives the code from all arms in randomized order. They evaluate each submission against the design document's own kill criteria, cost model, and scope boundaries.
4. Primary metric: **weighted defect count**. Violations of higher-tier properties count more than lower-tier violations, reflecting Witness's actual marginal contribution. Weights: Tier 1 violations = 1 point, Tier 2 = 2 points, Tier 3 = 4 points, Tier 4 = 8 points. Rationale: Tier 1 type constraint violations are errors any competent TypeScript developer would avoid; Tier 3/4 invariant violations are the deep correctness failures where verification adds real value. An unweighted count would overweight trivial properties. Secondary metrics: unweighted defect count (for comparison), code quality (maintainability, readability on 1–5 scale), test coverage of generated tests, and evaluator confidence.

**Blinding:** Evaluators do not know which arm produced which code. The code is presented in a standardized format with consistent style (auto-formatted by Prettier). Witness annotations are stripped. Human-written code is auto-formatted to the same standard.

**Evaluation document authorship:** The 10 evaluation design documents must be written by engineers who are not members of the Witness development team. This eliminates the conflict of interest where builders unconsciously write documents optimized for Witness extraction. The development team may provide the Design Document Standard as guidance, but must not review or influence the evaluation documents before they enter the study.

**Inter-rater reliability:** Before analyzing primary metrics, measure evaluator agreement using Krippendorff's alpha (α). If α < 0.67 (the conventional threshold for tentative conclusions), the defect definition must be tightened and evaluators re-calibrated before results are interpretable. If α < 0.40, the evaluation methodology is unreliable and must be redesigned. Evaluators receive a calibration session with 2 practice documents (not in the study set) to align on what constitutes a "property violation."

**Statistical power:** With 10 documents and 5 evaluators, we have 50 evaluations per arm. Defect counts are likely Poisson-distributed with high variance across documents of varying complexity. For a control arm averaging 5 defects per evaluation with standard deviation 3, detecting a 40% reduction requires approximately 45 observations per arm at α=0.05, β=0.20 — barely within the design's 50. If variance during the pilot (Week 0) suggests the study is underpowered, increase to 15 documents or 7 evaluators before proceeding. The effect size target (40% defect reduction) is aggressive; the kill criterion (K6) uses a lower threshold (20% improvement) as the viability floor, which requires fewer observations.

**Confound: human-written code bias.** Senior engineers writing code from a design document produce code reflecting shared conventions. Evaluators (also senior engineers) may rate human-written code more favorably due to stylistic familiarity rather than correctness. Mitigation: the primary metric is *property violations* (objective, checkable against the design document's stated properties), not *code quality* (subjective). Code quality is a secondary metric reported separately.

**Timeline:** The evaluation runs during Week 6 of the roadmap (Section 6). Results are available by end of Week 6. Kill criteria are evaluated at the start of Week 7.

---

## 6. Implementation Roadmap

**[Principle 13: Scope Discipline | Principle 14: Week-Level Roadmap]**

### V1 Scope

V1 targets TypeScript only. The design document must conform to the Design Document Standard. The verification pipeline supports Tiers 1–3 (static analysis, property-based, symbolic). Tier 4 (SMT) is deferred to V2.

### Week 0: Extraction Pilot (Pre-Commitment Gate)

Before committing to the 6-week build, validate the core thesis with a manual extraction exercise. This is the single most important de-risking step.

**Procedure:**
1. Collect 3–5 existing design documents written independently of the Witness project. These must be real documents from real engineering workflows, not purpose-written examples. **The author's own design documents must not be used** — they reflect internalization of the Design Document Standard and would overstate extractability for the general population. Candidate sources: Oxide Computer's public RFD repository (rfd.shared.oxide.computer), Google's published design doc examples, or design documents from collaborators who have not read the Design Document Standard.
2. For each document, manually extract every verifiable property. Classify each as functional or performance. Assign a tier (1–4).
3. Record: total properties found, percentage with unambiguous numeric thresholds, percentage that could be expressed as a test or constraint, and percentage that are cross-function.

**Go/No-Go criteria:**
- ≥70% of documents yield ≥5 extractable properties → **Go.** Proceed to Week 1.
- 50–70% extractability → **Go with caution.** Proceed but increase K1 threshold to ≥70%.
- <50% extractability → **No-Go.** Revise the thesis. Design documents as-practiced may not contain formal-enough properties. Consider: (a) defining a "Witness-ready" document profile stricter than the Design Document Standard, or (b) adding a document enrichment step before extraction.

**Timeline:** 2–3 days. This is a weekend's work with high information value.

**Pilot Results (2026-02-26).** Extraction was performed on five Oxide Computer RFDs (110, 445, 490, 58, 63) — real engineering documents written independently of the Design Document Standard.

| Document | Candidates | Extractable | Rate | Func / Perf | Tier Distribution (T1/T2/T3/T4) |
|---|---|---|---|---|---|
| RFD 110 (CockroachDB) | 86 | 53 | 61.6% | 41 / 12 | 12 / 35 / 6 / 0 |
| RFD 445 (Backpressure) | 32 | 26 | 81.3% | 22 / 4 | 9 / 10 / 5 / 1 |
| RFD 490 (Packed Extents) | 45 | 31 | 68.9% | 29 / 2 | 14 / 11 / 5 / 0 |
| RFD 58 (Rack Switch) | 115 | 88 | 76.5% | 78 / 10 | 55 / 25 / 7 / 0 |
| RFD 63 (Network Arch) | 75 | 53 | 70.7% | 52 / 1 | 19 / 24 / 10 / 0 |
| **Aggregate** | **353** | **251** | **71.1%** | **222 / 29** | **109 / 105 / 33 / 1** |

**Decision: Go.** 5/5 documents (100%) yield ≥5 extractable properties. Mean 50.2 properties per document; median extractability rate 70.7%. Tier 1–2 accounts for 85.3% of properties, confirming the cost gradient prediction. Functional-to-performance ratio is 7.7:1, meaning most verification is environment-independent. Full per-property extraction records are in `WITNESS-WEEK0-PILOT-RESULTS.md`; extraction protocol is in `WITNESS-WEEK0-EXTRACTION-PROTOCOL.md`.

### Week-by-Week Plan

| Week | Deliverable | Dependencies | Gate |
|------|-------------|-------------|------|
| **Weeks 1–2** | Property Extractor: parse design document sections, extract properties to JSON schema, iterate prompt engineering across multiple document styles, validate extraction accuracy on 5 sample documents. Two weeks because this is the thesis-critical component — it deserves development time proportional to its importance. | Week 0 pilot passes | Extraction accuracy ≥60% on sample docs |
| **Week 3** | Code Generator: generate TypeScript from extracted properties, represent as content-addressed AST graph, render to files | Property Extractor (Weeks 1–2) | Generated code compiles for 3 sample docs |
| **Week 4** | Graduated Verifier: implement Tiers 1–3, tier assignment logic, re-generation loop with counterexample feedback | Code Generator (Week 3) | ≥50% of properties verify on sample docs |
| **Week 5** | End-to-end pipeline: traceability matrix generation, output package, CLI interface (`witness generate <doc.md>`), incremental re-verification on code change | All prior weeks | Full pipeline runs on 5 sample docs |
| **Week 6** | Evaluation: run blinded multi-arm study per Section 5, collect weighted defect counts and secondary metrics | 10 evaluation documents (prepared by external authors during Weeks 1–5), 5 evaluators recruited | Data collected |
| **Week 7** | Decision gate: analyze evaluation results against kill criteria; if K1–K6 pass, plan V2; if any kill criterion triggers, assess whether redesign or halt | Week 6 results | Go / Redesign / Halt |

### Conditional Gates

- **After Week 0 (existential):** If manual extraction pilot shows <50% property extractability on real-world documents, halt. The core thesis requires revision before any build investment. This is the cheapest gate in the entire roadmap — 2–3 days to validate or falsify the foundational assumption.
- **After Week 2:** If extraction accuracy is below 60% after two weeks of development, assess whether the gap is in prompt engineering (iterate further) or in the fundamental expressiveness of design documents (revisit thesis). The two-week allocation already provides iteration time; if 60% is not achieved in this window, the problem is likely structural.
- **After Week 4:** If verification success rate is below 50%, assess whether the gap is in tier assignment (fixable) or in the verifier's capabilities (requires verifier improvements, potentially blocking).
- **After Week 6:** Kill criteria evaluation. This is the hard gate. No V2 planning until K1–K6 are evaluated.

### V2 Scope (Conditional on V1 Success)

- Tier 4 (SMT) verification via Z3
- Support for Rust (with Verus integration potential)
- Design document authoring assistance (property preview during writing)
- CI/CD integration (re-verify on every commit that touches Witness-generated code)

---

## 7. Design Decisions

**[Principle 9: Decisions and Open Questions]**

### Decided

| Decision | Choice | Rationale | Alternatives Considered |
|----------|--------|-----------|------------------------|
| V1 target language | TypeScript | Largest design document authorship base; fast-check is mature for property testing; strong type system for Tier 1 verification | Rust (stronger type system but smaller user base), Python (popular but weak static analysis) |
| Code representation | Content-addressed AST graph | Structural validity by construction; incremental re-verification via hash comparison; semantic mutation for targeted re-generation | Raw text (simpler but no structural guarantees), tree-sitter CST (no content-addressing) |
| LLM orchestration | Model-agnostic with fallback | Multi-provider support for resilience; sandboxed execution for safety | Single-provider (fragile), LangChain (Python-only, heavy abstraction) |
| Verification engine | Graduated pipeline (built into Witness) | Property testing → symbolic → SMT gradient; TypeScript-native; designed for "mortals" not proof experts | Dafny (requires formal spec), Lean (steep learning curve), existing verifier (none fits this use case) |
| Output format | Verified package (code + tests + proofs + traceability) | Audit trail enables trust verification; traceability enables design document updates to trigger targeted re-verification | Code-only (no verification evidence), Formal proof only (not useful to most engineers) |
| Content addressing | BLAKE3 | Fast (3.5 GB/s), cryptographically secure, deterministic; proven approach for AST hashing | SHA256 (slower), xxHash (not cryptographic) |

### Open Questions

| Question | Resolution Path | Decision Deadline |
|----------|----------------|-------------------|
| Which LLM performs best for property extraction? | A/B test Claude Opus vs. GPT-5 vs. Gemini on 10 sample documents during Weeks 1–2 | End of Week 2 |
| Should Tier 2 use fast-check or a custom property testing library? | Evaluate fast-check's expressiveness for design-document-derived properties on 3 sample docs | End of Week 3 |
| How to handle properties that span multiple functions? | Prototype cross-function property tests during Week 4; if >20% of properties are cross-function, design a composition strategy | End of Week 4 |
| What is the minimum design document quality for reliable extraction? | Track extraction accuracy vs. document quality metrics during evaluation; define a "Witness-ready" document checklist | End of Week 6 |
| Should Witness support partial design documents (missing sections)? | User interviews with 5 engineers during Weeks 5–6; assess demand vs. extraction accuracy impact | End of Week 6 |

---

## 8. Risks & Mitigations

**[Principle 10: Three-Tier Risks]**

### Technical Risks

**T1: Property extraction accuracy may be too low for reliable verification.**
Design documents, even those conforming to the standard, contain ambiguity that formal specifications do not. A kill criterion stated as "if latency exceeds 200ms, the approach is invalidated" is clear to a human but requires the extractor to infer: 200ms refers to p99 or p50? Under what load? For which operation?

*Mitigation:* The extraction phase produces a human-reviewable property list before any code is generated. The author can correct misinterpretations before generation begins. Kill criterion K1 (≥60% accuracy on first pass) is set conservatively to account for this; the review step raises effective accuracy to near-100% for engaged users.

*Severity:* High. If extraction is fundamentally unreliable (below 40% even after prompt optimization), the core thesis that design documents contain "formal-enough" properties is wrong.

**T2: Re-generation loop may not converge.**
When a property test fails, the counterexample-guided re-generation may oscillate: fixing one property breaks another. This is the fundamental challenge of multi-property synthesis — satisfying all constraints simultaneously is harder than satisfying them individually.

*Mitigation:* The re-generation loop is capped at 5 iterations. Properties are verified in tier order (cheapest first), so Tier 1 constraints are satisfied before Tier 2 generation begins. Semantic mutation applies targeted AST fixes rather than full regeneration, preserving already-verified properties. Kill criterion K3 (convergence failure on >30% of properties) detects systemic issues.

*Severity:* Medium. Convergence failure on individual properties is expected and handled (escalate to human). Systemic non-convergence would require architectural changes to the generation strategy.

**T3: Symbolic execution (Tier 3) may be too slow for interactive use.**
Symbolic execution scales poorly with code complexity. A function with 10 branch points has 1,024 paths; symbolic exploration of all paths may take minutes.

*Mitigation:* Tier 3 is reserved for ~20% of properties (those requiring path-sensitive reasoning). The symbolic executor uses bounded exploration with configurable depth. Kill criterion K5 (>30 minutes end-to-end) catches cases where Tier 3 dominates runtime.

*Severity:* Low. Tier 3 can be skipped in favor of Tier 2 (property testing) with a reduction in confidence level but not a loss of functionality.

### Product Risks

**P1: Engineers may not write design documents to a sufficient standard.**
Witness requires design documents with kill criteria, cost models, and scope boundaries. Many engineering organizations produce design documents that lack quantitative rigor.

*Mitigation:* Witness explicitly requires documents conforming to the Design Document Standard. This is a feature, not a bug: it creates a positive feedback loop where Witness adoption drives document quality improvement. The evaluation in Section 5 uses documents written to the standard, so the results apply only to conforming documents. V2 may include a document quality checker that identifies sections with insufficient quantitative content.

*Severity:* High for adoption, low for correctness. Witness works correctly on conforming documents; the risk is that too few documents conform.

**P2: The verified package may be too opaque for developer trust.**
Developers may not trust code they did not write, even with verification evidence. The traceability matrix and proof artifacts may be seen as complexity rather than assurance.

*Mitigation:* The output includes human-readable property tests (Tier 2) that developers can run, read, and understand. The traceability matrix links every property to a plain-English source clause in the design document. The verification report uses pass/fail with specific measurements, not formal logic notation. User testing during the evaluation (Week 5) measures trust and comprehension as secondary metrics.

*Severity:* Medium. Trust is a UX challenge, not a technical one. It can be improved iteratively.

### Existential Risks

**E1: Formal specification languages become easy enough to use directly.**
If Dafny, Lean, or Verus become accessible to mainstream engineers (through AI assistance, better tooling, or education), the "bridge" that Witness provides becomes unnecessary. Why extract approximate properties from design documents when you can write exact specifications?

*Assessment:* The vericoding benchmark shows 82% success in Dafny but only 27% in Lean. Dafny is the most accessible formal language, and its success rate suggests the "specification → verified code" pipeline is maturing. However, the specification-writing bottleneck remains: Congdon's "few hundred people" estimate for formal specification expertise is not changing at the pace needed to match AI code generation volume. Even in the scenario where formal specifications become mainstream, Witness retains value as a *triage* tool: it identifies which parts of a system warrant full formal specification by first verifying what can be verified from the design document alone.

*Timeline:* 3–5 years for formal spec languages to reach mainstream accessibility, if Kleppmann's prediction holds. Witness's 6-week development timeline means the investment is recovered long before this risk materializes.

**E2: AI code generation becomes reliable enough that verification is unnecessary.**
If LLMs achieve near-zero defect rates without verification, the demand for Witness disappears.

*Assessment:* This contradicts the observed trend. As models become more capable, they are deployed on harder problems, maintaining a constant error rate on the frontier. The vericoding benchmark found that LLMs "easily generate implementations (consistent with vibe coding success) but struggle more with proofs" — the generation capability outpaces the verification capability, widening the gap Witness addresses.

*Timeline:* No evidence suggests this is near-term. AI-generated code defect rates have decreased but remain significant for non-trivial systems.

---

## 9. Summary

**[Principle 15: Thesis as Bet]**

Witness is a bet on three propositions:

1. **Design documents already contain formal-enough specifications.** Kill criteria with numeric thresholds, cost models with per-operation budgets, and scope boundaries with explicit exclusions are not prose decoration — they are natural-language specifications with quantitative rigor. If this is true, an LLM can extract verifiable properties from them. If this is false, kill criterion K1 will surface it within Week 1.

2. **Graduated verification is the right cost curve.** Not all properties need SMT proofs. Most can be verified by property tests; some by symbolic execution; a few by full SMT. This cost gradient makes verification economically viable at $7–$21 per document, roughly one order of magnitude cheaper than LLM-assisted Dafny specification with human review. The cost advantage is modest; the accessibility advantage (no formal specification expertise required) is the real differentiator. If this cost curve is wrong, kill criterion K4 will surface it within Week 4.

3. **Verified code from design documents has fewer defects than unverified AI code.** This is the claim that matters. If design-document-anchored verification produces measurably better code, then Witness has found the bridge between vibe coding and vericoding — the bridge that meets engineers where they are (writing design documents) rather than where formal methods practitioners are (writing Dafny). If this is false, kill criterion K6 will surface it within Week 7.

**The bet costs a 2-day extraction pilot (Week 0) followed by 7 weeks of development and approximately $500 in LLM API costs for evaluation.** The Week 0 pilot tests proposition 1 before any build investment. The author has built systems in adjacent problem spaces (AST manipulation, LLM orchestration, pattern libraries, content-addressed code, language design with correctness philosophy). Witness draws on that experience but is a standalone system — every component is purpose-built.

**If the bet is wrong,** the kill criteria will identify which proposition failed and why. Proposition 1 is tested in 2 days (Week 0). No remaining proposition requires more than 7 weeks to test. The investment is bounded and the failure modes are informative.

**If the bet is right,** Witness occupies a cell in the verification landscape that no existing tool occupies: verified generation from structured natural-language specifications. The design document — already the standard artifact for proposing new systems — becomes the specification for verified synthesis. The culture change Kleppmann identified is not "learn Dafny" but "write better design documents." That is a change most engineering organizations are already trying to make.

**What happens if this bet is not placed:** AI code generation continues to scale without verification. The gap between code volume and verification capacity widens. Engineers review AI-generated code with decreasing effectiveness as volume increases. The formal verification community builds increasingly powerful tools for a specification format that mainstream engineers do not use. The bridge remains unbuilt.

---

## Sources

- Bursuc, S., Ehrenborg, T., Lin, S., et al. "A benchmark for vericoding: formally verified program synthesis." arXiv:2509.22908, September 2025. POPL 2026.
- Kleppmann, M. "Prediction: AI will make formal verification go mainstream." martin.kleppmann.com, December 8, 2025.
- Congdon, B. "The Coming Need for Formal Specification." benjamincongdon.me, December 12, 2025.
- Baksys, M., et al. "ATLAS: Automated Toolkit for Large-Scale Verified Code Synthesis." arXiv:2512.10173, December 2025. Dafny Workshop, POPL 2026.
- Councilman, A., et al. "Towards Formal Verification of LLM-Generated Code from Natural Language Prompts." arXiv:2507.13290, July 2025.
- Atlas Computing. "A Toolchain for AI-Assisted Code Specification, Synthesis." atlascomputing.org, 2025.
- DeepSeek AI. "DeepSeek-Prover-V2: Advancing Formal Mathematical Reasoning via Reinforcement Learning for Subgoal Decomposition." arXiv:2504.21801, 2025.
- ByteDance Seed. "Seed-Prover: Deep and Broad Reasoning for Automated Theorem Proving." arXiv:2507.23726, 2025.
- JetBrains Research. "Can LLMs Enable Verification in Mainstream Programming?" March 2025.
- Keleş, A. "Test, don't (just) verify." alperenkeles.com, December 2025.
