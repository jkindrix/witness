# Witness

Design-document-anchored verified code synthesis. Takes a design document conforming to the Design Document Standard and produces a verified package: source code, tests, property proofs, and a traceability matrix.

Witness is a standalone system. Every component is purpose-built.

## Status

**Week 0 complete. Go decision confirmed. Ready for Week 1.**

- Design document scored 9.1/10 on independent review (3 rounds)
- Extraction pilot: 251 properties from 5 Oxide Computer RFDs (71.1% extractability, 85.3% Tier 1–2)
- All 5 documents exceeded ≥5 property threshold (100% pass rate)

## Key Files

- `WITNESS-DESIGN-DOCUMENT.md` — The complete design document. Read this first. Contains thesis, architecture, cost model, evaluation plan, 7-week roadmap, and kill criteria.
- `WITNESS-WEEK0-EXTRACTION-PROTOCOL.md` — Extraction methodology, rubric, and tier assignment rules.
- `WITNESS-WEEK0-PILOT-RESULTS.md` — All 251 extracted properties across 5 RFDs with per-document metrics and hard extraction decisions.

## External References

- **Design Document Standard:** `~/references/document-generation/the-design-document-standard.md` — The 15-principle standard that Witness input documents must conform to.
- **Blood Proposal 7:** `~/blood/docs/proposals/EXTRAORDINARY_FEATURES.md` — Gradual Verification proposal that independently converges on the same graduated verification concept. Conceptual influence, not a dependency.
- **Formal Verification Taxonomy:** `~/software-quality-taxonomy/16-formal-verification/` — Contains `formal-verification.md` and `advanced-formal-verification.md` with a `witnesses/` directory blueprint that predates this project.

## Next Steps (Week 1)

Per the design document roadmap:

1. **Weeks 1–2: Property Extractor.** Parse design document sections, extract properties to JSON schema, iterate prompt engineering, validate on 5 sample documents. Gate: extraction accuracy ≥60%.
2. The 251 pilot properties in `WITNESS-WEEK0-PILOT-RESULTS.md` are the ground truth for evaluating extractor accuracy.
3. The 5 hardest extraction decisions (documented in the results file) are the edge-case test suite.

## Architecture Summary

```
Design Document (Markdown)
    ↓
Property Extractor (LLM) → JSON property schema
    ↓
Code Generator (LLM) → content-addressed AST graph (BLAKE3)
    ↓
Graduated Verifier
  Tier 1: Static analysis (tsc + AST) — ~40% of properties, $0.00/fn
  Tier 2: Property-based testing (fast-check) — ~30%, $0.02–0.10/fn
  Tier 3: Symbolic execution — ~20%, $0.50–2.00/fn
  Tier 4: SMT (Z3) — ~10%, $2.00–5.00/fn
    ↓
Re-generation loop (counterexample → LLM → re-verify)
    ↓
Output Package
  ├── src/          (generated code)
  ├── tests/        (generated tests)
  ├── proofs/       (verification artifacts)
  └── TRACEABILITY.md (property → code → test → proof)
```

## Kill Criteria

- **K1:** Property extraction accuracy <60% after Week 2
- **K2:** Verification success rate <50% after Week 4
- **K3:** End-to-end pipeline fails on >2 of 5 sample documents
- **K4:** Cost per verified function >10x the design document estimate
- **K5:** Blinded evaluators cannot distinguish Witness output from unverified baseline
- **K6:** Defect rate reduction <40% vs unverified AI-generated code
