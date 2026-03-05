# Feature Specification: Discovery Research Integration

**Branch**: feature/discovery-research-integration  |  **Created**: 2026-03-05  |  **Status**: Draft
**Input Brief**: Integrate web research into PAW Discovery Extraction and Correlation stages to enrich themes with external context and assess gap feasibility.

## Overview

PAW Discovery transforms input documents into actionable MVP roadmaps through a five-stage pipeline: Extraction, Mapping, Correlation, Journey Grounding, and Prioritization. Currently, the only external knowledge source is codebase research during the Mapping stage. Input documents — often internal-facing — may assume context, use ambiguous terminology, or lack grounding in industry standards. When Correlation identifies gaps (themes with no matching codebase capability), there are no signals about whether off-the-shelf solutions exist or how feasible the gap is to close.

Discovery Research Integration introduces opt-in web research at two stages — Extraction and Correlation — each with a distinct purpose. At Extraction, research enriches theme understanding with external context: industry standards, common approaches, terminology clarification, and prior art. At Correlation, research assesses the feasibility of identified gaps: existing libraries, implementation patterns, and ecosystem solutions. Research runs before artifact creation so the main artifacts (Extraction.md, Correlation.md) are synthesized with research findings. Separate research provenance files preserve source traceability.

The feature follows established Discovery patterns — init-time configuration (like Prioritization Mode), per-stage execution modes (autonomous/guided/skip), subagent delegation (like Mapping's code research), and extended review criteria. When research is disabled, all stages behave exactly as they do today.

## Objectives

- Enable Discovery users to opt into web research that enriches their extracted themes with external industry context (Rationale: input documents often assume knowledge that research can surface)
- Provide feasibility signals for gaps identified during Correlation (Rationale: knowing a gap has off-the-shelf solutions vs. requires custom development improves prioritization accuracy)
- Maintain source traceability so users can distinguish document-sourced insights from research-sourced insights (Rationale: provenance matters for decision confidence)
- Preserve existing Discovery behavior when research is not enabled (Rationale: research is additive, not disruptive)
- Give users control over research scope and depth through familiar execution modes (Rationale: users may want full autonomy, guided selection, or to skip research entirely at any stage)

## User Scenarios & Testing

### User Story P1 – Research-Enriched Extraction

Narrative: A product lead provides internal planning documents to Discovery. The documents reference "SSO integration" and "real-time collaboration" without defining standards or approaches. With research enabled, the extraction stage first investigates these themes externally — discovering OAuth 2.0/OIDC standards for SSO and CRDTs/OT approaches for collaboration — then produces an Extraction.md that synthesizes both document content and research context.

Independent Test: Run extraction with research enabled on documents containing ambiguous themes; verify Extraction.md includes external context not present in the input documents.

Acceptance Scenarios:
1. Given input documents with ambiguous themes and research enabled in autonomous mode, When extraction completes, Then Extraction.md contains research-enriched theme definitions and ExtractionResearch.md exists as a provenance record
2. Given input documents with clear, well-defined themes, When extraction research runs, Then research findings note "no significant external context needed" for already-clear themes and the artifact is not degraded
3. Given research enabled with guided mode selected, When extraction begins, Then the agent presents a list of identified themes and recommends which to research, and the user can confirm, adjust, or skip

### User Story P2 – Feasibility-Assessed Correlation

Narrative: After extraction and mapping, a correlation reveals three gaps — features envisioned but not present in the codebase. With research enabled, the correlation stage investigates each gap externally — discovering that two have mature library solutions and one requires custom development. Correlation.md reflects these feasibility signals alongside the standard match/gap analysis.

Independent Test: Run correlation with research enabled after a mapping stage identifies gaps; verify Correlation.md includes feasibility context for gap entries.

Acceptance Scenarios:
1. Given Extraction.md and CapabilityMap.md with identified gaps and research enabled, When correlation completes, Then Correlation.md includes feasibility context for each gap and CorrelationResearch.md exists as a provenance record
2. Given correlation research in guided mode with five gaps, When the agent presents gaps for research selection, Then the user can select a subset and research runs only on selected gaps
3. Given a gap where research finds no relevant external solutions, When correlation research completes for that gap, Then the research artifact notes the gap as requiring custom development and the correlation artifact reflects this

### User Story P3 – Research Configuration at Init

Narrative: A user starts a new Discovery workflow. During init, they are asked whether to enable research. They enable it, and this configuration persists throughout the workflow. At each research-eligible stage, they choose how research executes.

Independent Test: Initialize Discovery with research enabled; verify DiscoveryContext.md reflects the setting and research-eligible stages prompt for execution mode.

Acceptance Scenarios:
1. Given a new Discovery init, When the user enables research, Then DiscoveryContext.md includes the research configuration and the setting persists for the workflow
2. Given research enabled in DiscoveryContext.md, When a research-eligible stage begins, Then the user is prompted to select execution mode (autonomous/guided/skip)
3. Given research disabled in DiscoveryContext.md, When extraction and correlation stages run, Then they behave identically to the current non-research flow

### User Story P4 – Research-Aware Reviews

Narrative: After extraction with research, the review stage evaluates the artifact. The review checks not only standard quality criteria but also whether research findings are well-integrated, sources are credible, and document-sourced vs research-sourced insights are distinguishable.

Independent Test: Run extraction review on a research-enriched Extraction.md; verify the review evaluates research integration quality.

Acceptance Scenarios:
1. Given a research-enriched Extraction.md, When extraction review runs, Then the review includes assessment of research integration quality alongside standard criteria
2. Given an Extraction.md produced without research, When extraction review runs, Then only standard criteria are evaluated (no research-specific criteria applied)

### Edge Cases

- Research returns nothing useful for a theme: research artifact notes "no significant external context found"; extraction proceeds with document-sourced understanding only
- Research contradicts input documents: discrepancy surfaced in both research artifact and main artifact; flagged for user attention during interactive Q&A or review
- Web search is unavailable or rate-limited: stage gracefully degrades to no-research behavior; limitation noted in research artifact
- Very large number of themes (10+): guided mode groups and recommends a subset; autonomous mode prioritizes by ambiguity/novelty
- Re-extraction triggered with new inputs: research artifacts are invalidated alongside main artifacts per existing cascade behavior

## Requirements

### Functional Requirements

- FR-001: Discovery init prompts users to opt into research and records the setting in DiscoveryContext.md (Stories: P3)
- FR-002: When research is enabled, research-eligible stages (Extraction, Correlation) prompt for execution mode selection: autonomous, guided, or skip (Stories: P3, P1, P2)
- FR-003: Extraction research identifies themes from input documents, investigates them via web search for external context (standards, approaches, terminology, prior art), and produces ExtractionResearch.md (Stories: P1)
- FR-004: Extraction.md is produced as a synthesis of input document content and research findings when research is conducted (Stories: P1)
- FR-005: Correlation research investigates identified gaps via web search for feasibility signals (existing solutions, libraries, patterns, effort indicators), and produces CorrelationResearch.md (Stories: P2)
- FR-006: Correlation.md incorporates research feasibility context for gap entries when research is conducted (Stories: P2)
- FR-007: In guided mode, the agent presents identified themes (extraction) or gaps (correlation) and recommends which to research; the user confirms, adjusts, or skips (Stories: P1, P2)
- FR-008: Research artifacts (ExtractionResearch.md, CorrelationResearch.md) serve as provenance records with source attribution (Stories: P1, P2)
- FR-009: Downstream stages (Mapping, Journey Grounding, Prioritization) consume only the main artifacts — they do not require access to research artifacts (Stories: P1, P2)
- FR-010: Extraction and correlation review skills evaluate research integration quality when research was conducted (Stories: P4)
- FR-011: When research is disabled or skipped, stages behave identically to existing non-research behavior (Stories: P3)
- FR-012: Research artifacts follow existing cascade invalidation — re-extraction invalidates ExtractionResearch.md; downstream invalidation includes CorrelationResearch.md (Stories: P1, P2)
- FR-013: Research gracefully degrades when web search is unavailable — stages proceed without research and note the limitation (Stories: P1, P2)

### Key Entities

- **Research Configuration**: The init-time setting in DiscoveryContext.md controlling whether research is enabled for the workflow
- **Execution Mode**: The per-stage selection (autonomous/guided/skip) determining how research runs at each eligible stage
- **Research Artifact**: A per-stage provenance file (ExtractionResearch.md, CorrelationResearch.md) recording research findings, sources, and how they informed the main artifact
- **Enriched Artifact**: A main stage artifact (Extraction.md, Correlation.md) that synthesizes both primary sources and research findings

## Success Criteria

- SC-001: A user can enable research at Discovery init and the setting persists in workflow configuration for all subsequent stages (FR-001)
- SC-002: When research is enabled, the user is prompted for execution mode at each research-eligible stage and can choose autonomous, guided, or skip (FR-002)
- SC-003: Extraction with research produces an Extraction.md containing insights not present in input documents, traceable to external sources via ExtractionResearch.md (FR-003, FR-004, FR-008)
- SC-004: Correlation with research produces a Correlation.md with feasibility context for gaps, traceable to external sources via CorrelationResearch.md (FR-005, FR-006, FR-008)
- SC-005: In guided mode, the user can narrow research scope to selected themes or gaps before research executes (FR-007)
- SC-006: Downstream stages (Mapping, Journey Grounding, Prioritization) produce the same quality output regardless of whether they read research artifacts directly — all enrichment is already in the main artifacts (FR-009)
- SC-007: Reviews detect and report on research integration quality (source credibility, integration completeness, source differentiation) when research was conducted (FR-010)
- SC-008: With research disabled, all stages produce output identical to current behavior — no observable difference (FR-011)
- SC-009: Re-extraction with changed inputs invalidates research artifacts alongside main artifacts (FR-012)

## Assumptions

- Web search tool (`web_search`) is available in the agent execution environment and returns relevant results for technology/industry queries (Rationale: this is the standard research mechanism in the Copilot CLI agent environment)
- Research subagents can be invoked using the existing subagent delegation pattern established by the Mapping stage (Rationale: proven pattern already in the codebase)
- Research will add meaningful latency to affected stages; users accept this tradeoff by opting in (Rationale: web search and synthesis take time, but the value proposition is clear)
- The agent can evaluate source credibility at a basic level (official docs, well-known projects vs. obscure sources) without a formal scoring system (Rationale: keeps complexity manageable while still providing value)

## Scope

In Scope:
- Research integration at Extraction and Correlation stages
- Init-time opt-in configuration in DiscoveryContext.md
- Per-stage execution mode selection (autonomous/guided/skip)
- Separate per-stage research provenance artifacts
- Extended review criteria for research-enriched artifacts
- Graceful degradation when web search is unavailable
- Cascade invalidation of research artifacts

Out of Scope:
- Research at Mapping, Journey Grounding, or Prioritization stages
- Programmatic invocation of Copilot deep research mode via SDK (using subagent + web_search instead)
- GitHub code search integration for research (existing code research in Mapping handles this)
- Research artifact structure standardization (theme-by-theme sections, confidence ratings, source bibliography — deferred to planning)
- Formal source credibility scoring system
- Incremental re-research (full re-research on re-extraction)

## Dependencies

- Existing Discovery workflow pipeline (Extraction → Mapping → Correlation → Journey Grounding → Prioritization)
- `web_search` tool availability in the agent execution environment
- Subagent delegation capability (task tool with general-purpose agent type)
- DiscoveryContext.md configuration infrastructure
- Existing review skill framework

## Risks & Mitigations

- **Web search quality varies**: Research findings may be inaccurate or outdated. Mitigation: review skills explicitly evaluate source credibility; research artifacts preserve sources for user verification.
- **Research contradicts input documents**: Could confuse theme understanding. Mitigation: discrepancies are surfaced explicitly in artifacts and flagged for user attention.
- **Autonomous mode over-researches**: Could waste time on well-understood themes. Mitigation: agent prioritizes ambiguous/novel themes; guided and skip modes provide user control.
- **Token/cost increase**: Additional web search calls increase resource usage. Mitigation: opt-in by default; skip mode and guided selection keep usage proportional to value.

## References

- WorkShaping: .paw/work/discovery-research-integration/WorkShaping.md
