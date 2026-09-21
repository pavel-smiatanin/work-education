# Azure Durable Functions AI Agents Research — v2 Change Log, Requirements Mapping, Critique, and Next Steps

This note accompanies `output/Azure-Durable-Functions-AI-Agents-v2/index.md`. It records, in one place, the four things requested alongside the v2 document itself: how v1 mapped to the project requirements, what changed and why, an independent critique of v2, and the recommended next steps.

## 1. Requirements-to-v1 mapping (gap analysis)

Source: `input/requirements.md` (Description, Acceptance Criteria, TODO) checked against `output/Azure-Durable-Functions-AI-Agents-v1/index.md` section by section.

**Fully covered in v1** (no change needed, retained as-is in v2): orchestration overview; state management; long-running workflows; scalability; monitoring (all of Acceptance Criterion 1); human-in-the-loop, fan-out/fan-in, multi-agent collaboration patterns (Criterion 2, partial); benefits/limitations/trade-offs vs. Container Apps, standalone Durable Task SDKs, Logic Apps, and agent frameworks (Criterion 3); enterprise use cases with pattern mapping and R&D recommendations (Criterion 4); high-level reference architecture and implementation considerations (Criterion 5); determinism, payload limits, security/governance, cost (TODO items on constraints and platform features).

**Under-addressed in v1** — three specific items, all named explicitly in the requirements but only implicitly touched in v1:

| Requirement source | What v1 had | Gap |
|---|---|---|
| Acceptance Criterion 1: "...event-driven execution..." and Criterion 2: "...event-driven workflows..." | Only the human-in-the-loop external-event pattern and the timer-driven Monitor pattern; no treatment of Blob/Event Grid/Service Bus triggers as a way to *start* an agent workflow from a system event | v1 conflated "waiting on an event mid-workflow" with "being started by an event," and never covered the latter |
| TODO: "Compare Azure Durable Functions against alternative approaches (e.g., container-hosted agents, Azure Container Apps, **Azure Functions without Durable orchestration**, agent frameworks)" | Comparisons against Container Apps, standalone SDKs, Logic Apps, and agent frameworks — but not against plain stateless Azure Functions, the specific alternative named first in the TODO | The most basic "why not just skip Durable Functions" question had no explicit answer |
| TODO: "Identify implementation constraints, operational challenges..." | Determinism constraints (5.1) — but not orchestrator versioning/safe deployment, LLM rate-limit interaction, or disaster recovery, all of which are standard "operational challenges" documented by Microsoft for this exact platform | Three concrete, citable operational risks were absent |

No factual errors were found in v1's retained content; every claim re-checked against current Microsoft Learn documentation during this revision held up. The gaps above are the entire basis for the v2 changes — v2 is not a stylistic rewrite, it is a targeted closure of these three items plus the supporting material (risk register, glossary, testing guidance) needed to make them actionable.

## 2. What changed in v2

- **New Section 3.6 "Event-Driven Execution and Triggers"** — Blob, Event Grid, Service Bus/Queue, and timer triggers as ways to start or signal an orchestration, plus a new Diagram 5 (event-driven document-intake sequence).
- **New pattern entry in Section 4.1** — "Event-driven / trigger-initiated orchestration" added to the foundational pattern list, cross-referencing 3.6.
- **New Section 5.5 "Orchestrator Versioning and Safe Deployment"** — breaking-change identification, the three Microsoft-documented mitigation strategies (orchestration versioning, side-by-side deployment, stop-in-flight), and a new Diagram 24.
- **New Section 5.6 "LLM Rate Limits, Quotas, and Backpressure"** — TPM/RPM mechanics, the retry-layer-compounding risk specific to wrapping an already-retrying SDK in Durable Functions' own retry policy, and a new Diagram 25.
- **New Section 5.7 "Disaster Recovery and Multi-Region Considerations"** — the three Microsoft-documented DR scenarios, with the specific, easy-to-miss caveat that the recommended topology pauses rather than resumes in-flight sessions on failover.
- **New Section 6.1 "Durable Functions vs. Stateless Azure Functions"** — inserted as the first comparison in Section 6, directly closing the named TODO gap, with an explicit decision rule.
- **Section 6.4 decision tree updated** to include the stateless-function branch as its first question.
- **New risk register** appended to the former 6.4 (now 6.5) — reframes every limitation as severity × likelihood × mitigation.
- **New Appendix A: Glossary** and **expanded Sources** (13 new Microsoft Learn citations backing the new material).
- **8.3 Implementation Considerations and 9.2 Production Readiness Checklist** both updated with versioning, rate-limit, and DR gates so the new material is actionable, not just descriptive.
- All 36 original diagrams retained verbatim (content unchanged); 3 new diagrams added (5, 24, 25); all 39 diagrams renumbered sequentially and the Diagram Index regenerated to match exactly.

Net effect: 1,158 lines / 36 diagrams (v1) → 1,396 lines / 39 diagrams (v2), with every addition traceable to a specific requirements gap rather than general padding.

## 3. Critique of v2 (independent QA pass)

This is an adversarial self-review performed after v2 was drafted, looking specifically for the kind of errors that would make v2 *not* a strict improvement on v1.

**Checks performed and results:**

- **Diagram numbering integrity** — programmatically verified all 39 in-body diagram captions and all 39 Diagram Index rows are sequential 1–39 with no gaps or duplicates, and that caption order matches index order exactly. Passed after one fix (see below).
- **Cross-reference integrity** — every "(Section N.N)" and "(Diagram N)" reference in the body was checked against the actual section/diagram numbering. **One error was found and fixed**: the new event-driven diagram (Diagram 5) originally cited "(Diagram 24)" for the document-intake pipeline — a leftover from before renumbering, since the document-intake diagram is Diagram 27 in v2, not 24 (that number now belongs to the orchestrator-versioning diagram). Corrected in both `index.md` and the extracted `.mmd` source, and the SVG was re-rendered.
- **Markdown table integrity** — every table in the document was checked programmatically for consistent column (pipe) counts row-to-row; zero issues found.
- **TOC anchor integrity** — every Table of Contents link was checked against actual heading text using GitHub's anchor-generation rules; all 44 resolve correctly.
- **Diagram rendering** — all 39 Mermaid sources were rendered to SVG with `mmdc` (Mermaid CLI) and checked for rendering errors; zero failures, zero "syntax error" artifacts in any output SVG. One diagram (the versioning flowchart) was visually inspected in full to confirm its three deployment-phase clusters and their edges render as intended.
- **Factual grounding of new content** — the four new technical areas (orchestrator versioning/deployment behavior, Event Grid/Blob/Service Bus triggers, Azure OpenAI TPM/RPM rate-limit mechanics and retry-header behavior, and the Durable Task Scheduler's disaster-recovery scenarios) were each verified against current Microsoft Learn documentation via live lookup before being written, rather than drafted from memory; source URLs are listed in Section 13.
- **No content removed or weakened** — every v1 section, table, and diagram is present in v2 verbatim or with additive (not subtractive) edits; the comparison-table structure in Section 6 and the use-case table in Section 7 are unchanged apart from added cross-references.

**Residual limitations of v2 (disclosed, not hidden):**

- The document is still deliberately architecture/pattern-level, not a build guide — code samples remain out of scope, consistent with the original v1 scope decision and restated explicitly in the new "Assumptions and scope boundaries" in Section 2.1.
- Cost, quota, and SKU figures are point-in-time and flagged as such; they will drift and should be re-verified before use in a funding request.
- The new DR section stops at describing the trade-off (pause vs. resume on failover) rather than prescribing a specific RTO/RPO target, because that target is a business decision this document cannot make on the R&D team's behalf — this is called out explicitly as a Phase 4 activity rather than left implicit.

**Conclusion of the critique:** v2 is a strict superset of v1's content, closes all three requirements gaps identified in Section 1 above with material grounded in current Microsoft documentation, and the one numbering defect found during QA was caught and corrected before delivery. No other errors were found.

## 4. Proposed next steps

1. **Circulate v2 for a technical read** by one or two senior engineers on the R&D team, specifically asking them to react to Section 6.1 (stateless vs. Durable Functions) and Section 6.5 (risk register) — these are the sections most likely to surface a disagreement with the document's framing before it informs a go/no-go conversation.
2. **Stand up Phase 1 of the PoC roadmap (Section 9.1)** using the local development path described in 8.3 — the Durable Task Scheduler emulator plus Azurite — so the team can validate checkpointing behavior and the developer experience before any Azure spend is committed.
3. **Decide the orchestration-versioning strategy up front**, not after Phase 1: Section 5.5 recommends built-in orchestration versioning from the first non-throwaway PoC specifically so the team doesn't learn this lesson from a stuck production instance.
4. **Size a target Azure OpenAI deployment's TPM/RPM quota against the Phase 3 multi-agent fan-out scenario** before building it, using the guidance in Section 5.6, so the first fan-out test isn't also the first time the team hits a 429 storm.
5. **Bring a business stakeholder into the Section 5.7 trade-off conversation** (pause-vs-resume on regional failover) before Phase 4, so the production-readiness sign-off in Section 9.2 has an actual owner for that decision rather than defaulting silently to the cheapest DR topology.
6. **Re-verify all cost, quota, and SKU figures** in the document against current Azure pricing and the Foundry quota portal immediately before they are used in any funding or budget request, per the caveat in Section 2.1.
7. **Revisit this document at the end of Phase 1** (per the roadmap timeline, mid-to-late October 2026) and fold PoC findings back into a v3, particularly around whether the determinism and versioning disciplines proved as friction-heavy in practice as Sections 5.1 and 5.5 anticipate.
