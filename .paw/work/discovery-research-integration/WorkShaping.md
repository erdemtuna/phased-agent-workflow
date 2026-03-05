# Work Shaping: Discovery Research Integration

## Problem Statement

PAW Discovery processes input documents through a multi-stage pipeline (Extraction → Mapping → Correlation → Journey Grounding → Prioritization) to produce actionable roadmaps. Currently, the only external knowledge source is codebase research during the Mapping stage. Input documents may be incomplete, assume context, or lack industry grounding. Gaps identified during Correlation have no feasibility signals beyond what the codebase reveals.

**Who benefits**: Users running Discovery workflows get richer, better-grounded artifacts. Extraction themes are informed by external context (standards, common approaches, prior art). Correlation gap assessments include feasibility signals (libraries, patterns, ecosystem knowledge). Downstream stages (Journey Grounding, Prioritization) inherit this enrichment through already-synthesized artifacts.

## Work Breakdown

### Core Functionality

#### 1. Extraction Research — "What does this mean?"

**Purpose**: Enrich theme understanding with external context before the Extraction artifact is created.

**Flow**:
1. Input documents are loaded
2. Initial themes are identified (preliminary pass)
3. **Research phase**: Subagent investigates themes via `web_search` — industry standards, common approaches, terminology, relevant APIs/libraries, prior art
4. `ExtractionResearch.md` is produced (provenance record)
5. `Extraction.md` is created as a synthesis of input documents AND research findings

**Boundary**: Extraction research does NOT consider the codebase. It is purely about enriching understanding of the themes from an external/industry perspective.

#### 2. Correlation Research — "How feasible is this gap?"

**Purpose**: Assess feasibility of identified gaps before the Correlation artifact is created.

**Flow**:
1. Extraction.md (enriched) and CapabilityMap.md are available
2. Initial correlation identifies matches, gaps, partials
3. **Research phase**: Subagent investigates gaps via `web_search` — existing solutions, libraries, implementation patterns, integration approaches, effort indicators
4. `CorrelationResearch.md` is produced (provenance record)
5. `Correlation.md` is created fusing extraction themes, capability mapping, and research feasibility signals

**Boundary**: Correlation research is specifically about bridging gaps between what's envisioned (themes) and what exists (codebase capabilities). It investigates the external solution landscape for identified gaps.

### Supporting Functionality

#### 3. Optionality Configuration

- Research is **opt-in**, configured at Discovery init time in `DiscoveryContext.md`
- Configuration fields:
  ```yaml
  research:
    enabled: true  # opt-in at init
  ```
- When disabled, stages execute exactly as they do today (no behavioral change)

#### 4. Per-Stage Execution Modes

When research is enabled, the user selects an execution mode at each research-eligible stage:

| Mode | Behavior |
|------|----------|
| **Autonomous** | Agent identifies research-worthy themes/gaps and researches them without asking |
| **Guided** | Agent proposes which themes/gaps to research (theme-level selection), user confirms/adjusts |
| **Skip** | Skip research for this stage, proceed with current information |

Mode selection happens at the start of each research-eligible stage (Extraction, Correlation).

#### 5. Research Artifacts (Provenance/Audit Trail)

| Stage | Research Artifact | Purpose |
|-------|-------------------|---------|
| Extraction | `ExtractionResearch.md` | Records what external context was found, sources, and how it informed theme enrichment |
| Correlation | `CorrelationResearch.md` | Records feasibility findings for gaps, sources, solution landscape assessment |

**Downstream access model**: Research artifacts are **not required reading** for subsequent stages. Main artifacts (Extraction.md, Correlation.md) are already synthesized with research findings. Research files serve as provenance/traceability records — showing where enrichment came from.

#### 6. Extended Review Criteria

Existing review skills (`paw-discovery-extraction-review`, `paw-discovery-correlation-review`) are extended with research quality criteria when research was conducted:
- Are research findings well-integrated into the main artifact?
- Are external sources credible and relevant?
- Does the artifact clearly distinguish document-sourced vs research-sourced insights?
- Are research gaps or low-confidence findings appropriately flagged?

Reviews should gracefully handle the case where research was skipped (no additional criteria applied).

## Edge Cases

| Scenario | Expected Handling |
|----------|-------------------|
| Research returns nothing useful for a theme | Note "no significant external context found" in research artifact; proceed with document-sourced understanding |
| Research contradicts input documents | Surface the discrepancy in both research artifact and main artifact; flag for user attention |
| Web search is unavailable or rate-limited | Gracefully degrade to no-research behavior; note the limitation |
| Very large number of themes (10+) | In guided mode, agent groups themes and recommends a subset; in autonomous mode, agent prioritizes by ambiguity/novelty |
| Re-extraction triggered | Research artifacts are invalidated alongside main artifacts (existing cascade behavior) |

## Architecture Sketch

```
┌─────────────────── Discovery Pipeline ───────────────────┐
│                                                           │
│  Input Docs ──┐                                          │
│               ▼                                          │
│  ┌──────────────────────┐                                │
│  │ EXTRACTION STAGE     │                                │
│  │                      │                                │
│  │ 1. Preliminary theme │                                │
│  │    identification    │                                │
│  │         │            │    ┌─────────────────────┐     │
│  │         ▼            │    │ Research Subagent    │     │
│  │ 2. Research phase ───┼───▶│ (web_search tool)   │     │
│  │         │            │    │                     │     │
│  │         ◀────────────┼────│ → ExtractionResearch│     │
│  │         │            │    │   .md               │     │
│  │         ▼            │    └─────────────────────┘     │
│  │ 3. Synthesize into   │                                │
│  │    Extraction.md     │                                │
│  └──────────┬───────────┘                                │
│             ▼                                            │
│  ┌──────────────────────┐                                │
│  │ MAPPING STAGE        │ (unchanged — uses              │
│  │ → CapabilityMap.md   │  paw-code-research)            │
│  └──────────┬───────────┘                                │
│             ▼                                            │
│  ┌──────────────────────┐                                │
│  │ CORRELATION STAGE    │                                │
│  │                      │                                │
│  │ 1. Initial match/gap │                                │
│  │    analysis          │                                │
│  │         │            │    ┌─────────────────────┐     │
│  │         ▼            │    │ Research Subagent    │     │
│  │ 2. Research phase ───┼───▶│ (web_search tool)   │     │
│  │         │            │    │                     │     │
│  │         ◀────────────┼────│ → CorrelationResearch│    │
│  │         │            │    │   .md               │     │
│  │         ▼            │    └─────────────────────┘     │
│  │ 3. Synthesize into   │                                │
│  │    Correlation.md    │                                │
│  └──────────┬───────────┘                                │
│             ▼                                            │
│  Journey Grounding → Prioritization → Roadmap            │
│  (consume enriched artifacts; no direct research access)  │
└───────────────────────────────────────────────────────────┘
```

## Critical Analysis

### Value Assessment

- **High value in Extraction**: Input documents are often internal-facing and assume context. External research can fill terminology gaps, identify industry standards the documents reference implicitly, and surface approaches the document authors may not have considered.
- **Moderate-to-high value in Correlation**: Knowing that a gap has off-the-shelf solutions (vs. requires custom development) significantly improves prioritization accuracy. Libraries, services, and established patterns reduce estimated effort.
- **Compounding benefit**: Since research is synthesized into main artifacts, every downstream stage benefits without added complexity.

### Build vs Modify Tradeoffs

- **Extraction stage**: Needs modification to support a pre-artifact research phase. Current flow is: load documents → extract → produce artifact. New flow adds a research step between extraction and artifact production.
- **Correlation stage**: Similar modification — add research between initial analysis and artifact production.
- **DiscoveryContext.md**: Needs new `research` configuration section.
- **Review skills**: Need extended criteria (additive change, not restructuring).
- **New components**: Research subagent invocation pattern (similar to existing paw-code-research delegation in Mapping).

## Codebase Fit

### Existing Patterns to Reuse

- **Subagent delegation**: Mapping stage already delegates to `paw-code-research` via `task(agent_type: "general-purpose")`. Research delegation follows the same pattern.
- **Execution modes**: Prioritization stage already implements interactive/guided/autonomous modes. Same pattern applies to research mode selection.
- **DiscoveryContext.md**: Already tracks stage configuration and progress. Adding research config is a natural extension.
- **Review extensions**: Review skills already have structured criteria. Adding research-specific criteria follows the existing pattern.
- **Cascade invalidation**: Re-extraction already triggers downstream invalidation. Research artifacts follow the same cascade.

### Integration Points

- `paw-discovery-extraction` skill: Primary modification target for extraction research
- `paw-discovery-correlation` skill: Primary modification target for correlation research
- `paw-discovery-init` skill: Needs to support research configuration in DiscoveryContext.md
- `paw-discovery-extraction-review` skill: Extended review criteria
- `paw-discovery-correlation-review` skill: Extended review criteria

## Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|------------|
| Web search quality varies | Medium | Agent should evaluate source credibility; review criteria catch poor integration |
| Research adds latency to Discovery | Low | Opt-in by default; skip mode available per-stage |
| Research findings may mislead if sources are outdated | Medium | Research artifact timestamps and source URLs enable verification |
| Scope creep in autonomous mode (researching everything) | Low | Theme-level selection in guided mode; autonomous mode should prioritize ambiguous/novel themes |
| Token/cost increase from web_search calls | Low | Skip mode and guided selection keep costs controllable |

## Open Questions for Downstream Stages

1. **Research artifact structure**: What specific sections should `ExtractionResearch.md` and `CorrelationResearch.md` contain? (Theme-by-theme findings? Source bibliography? Confidence ratings?)
2. **Source credibility signals**: Should the research subagent evaluate and score source reliability, or leave that to the review stage?
3. **Research prompt templates**: What specific research questions yield the best results for enrichment vs. feasibility contexts?
4. **Guided mode UX**: Exact interaction pattern for theme selection (list with checkboxes? ranked recommendation with confirm?)
5. **DiscoveryContext.md schema**: Exact field names and validation for research configuration

## Session Notes

### Key Decisions
- **Two-stage research** with clear boundary: enrichment (extraction) vs. feasibility (correlation)
- **Research-first timing**: Research happens before artifact creation, so the artifact is the synthesis
- **Separate provenance files**: Research findings are separate artifacts for traceability, but main artifacts are the downstream interface
- **Init-time opt-in**: Research is configured once at Discovery init, not decided ad-hoc per run
- **Per-stage mode selection**: Even when enabled, user can skip research at any individual stage
- **Theme-level guided scope**: In guided mode, the agent proposes themes to research (not individual questions)

### Rejected Alternatives
- **Inline research only** (no separate files): Rejected because source differentiation matters — knowing what came from documents vs. research has value for traceability
- **Single Research.md**: Rejected in favor of per-stage files to maintain clear 1:1 relationship with stage artifacts
- **Research during artifact creation** (interleaved): Rejected in favor of research-first → synthesized artifact, which produces richer artifacts
- **Downstream stages reading research files**: Rejected in favor of synthesis-only model — keeps stage interfaces clean
- **SDK-level Copilot research mode**: Deferred in favor of practical subagent + web_search approach available today
