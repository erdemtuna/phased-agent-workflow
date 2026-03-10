# Discovery Research Integration

## Overview

Discovery Research Integration adds opt-in web research to the PAW Discovery workflow's Extraction and Correlation stages. When enabled, research enriches extracted themes with external context (industry standards, approaches, prior art) and assesses gap feasibility (existing libraries, patterns, solutions). Research findings are synthesized into the main artifacts while separate provenance files preserve source traceability.

## Architecture and Design

### High-Level Architecture

Research integrates as an optional sub-phase within two existing Discovery stages:

```
Extraction Stage:
  Documents → Theme Identification → [Research Phase] → Q&A → Extraction.md

Correlation Stage:
  Extraction.md + CapabilityMap.md → Match/Gap Analysis → [Research Phase] → Correlation.md
```

Research runs via subagent delegation (same pattern as Mapping → paw-code-research). The subagent uses `web_search` to investigate themes or gaps, produces a provenance artifact, and returns findings for synthesis into the main artifact.

### Design Decisions

**Separate provenance artifacts**: Research findings live in `ExtractionResearch.md` and `CorrelationResearch.md` alongside main artifacts. This preserves source traceability — users can distinguish document-sourced from research-sourced insights. Downstream stages consume only the enriched main artifacts.

**Research-first timing**: Research runs before artifact creation so the main artifact is a synthesis of all sources. This produces richer artifacts than post-hoc annotation.

**Inline subagent prompts**: Research instructions are embedded in the extraction/correlation skills rather than a dedicated `paw-discovery-research` skill. Each stage has distinct research scopes (enrichment vs. feasibility) that don't generalize into a single reusable skill.

**Init-time mode selection**: The research mode (autonomous/guided/disabled) is configured once at init and applies to both research-eligible stages. This keeps the workflow flowing without repeated mode prompts.

### Integration Points

| Component | Change | Purpose |
|-----------|--------|---------|
| `paw-discovery-init` | New `research` parameter | Init-time opt-in configuration |
| `paw-discovery-extraction` | Research Phase section | Theme enrichment via web search |
| `paw-discovery-correlation` | Research Phase section | Gap feasibility assessment via web search |
| `paw-discovery-extraction-review` | Research Integration criteria | Quality check for research-enriched extraction |
| `paw-discovery-correlation-review` | Research Integration criteria | Quality check for research-enriched correlation |
| `paw-discovery-final-review` | 8th review criterion | Cross-artifact research quality check |
| `PAW Discovery.agent.md` | Cascade invalidation update | Include research artifacts in invalidation |
| `paw-discovery-workflow` | Artifact structure, activities table | Reference documentation updates |

## User Guide

### Prerequisites

- Web search tool (`web_search`) available in the agent execution environment
- Standard PAW Discovery setup (documents in `inputs/` folder)

### Basic Usage

**Enable research at init**:
```
Let's start a Discovery workflow for "Q1 Planning"
```
When prompted for configuration, set Research to `autonomous` or `guided`.

- **Autonomous**: Agent researches themes/gaps without asking at each eligible stage
- **Guided**: Agent recommends which themes/gaps to research; you confirm or adjust at each eligible stage

### Configuration

Research is configured via the `Research` field in `DiscoveryContext.md`:

```markdown
## Configuration
- **Research**: autonomous
```

### Research Artifacts

When research is conducted, provenance artifacts appear alongside main artifacts:

```
.paw/discovery/<work-id>/
├── Extraction.md              # Enriched with research context
├── ExtractionResearch.md      # Provenance: what was researched, sources, findings
├── Correlation.md             # Gaps include feasibility context
└── CorrelationResearch.md     # Provenance: gap feasibility research
```

## Testing

### How to Test

1. **Init with research enabled**: Start a Discovery workflow, set Research to `enabled`
2. **Extraction**: Verify the agent prompts for research execution mode; in guided mode, verify theme selection is offered
3. **ExtractionResearch.md**: Verify the provenance artifact is created with per-theme findings and sources
4. **Extraction.md**: Verify themes include research-enriched context with source attribution
5. **Correlation**: After mapping, verify the agent prompts for research mode; in guided mode, verify gap selection
6. **CorrelationResearch.md**: Verify the provenance artifact with per-gap feasibility findings
7. **Correlation.md**: Verify gap entries include feasibility context
8. **Reviews**: Verify extraction and correlation reviews include Research Integration criteria
9. **Disabled path**: Run with Research `disabled` and verify no research prompts or artifacts appear

### Edge Cases

- **Web search unavailable**: Research gracefully degrades; stage proceeds without research; limitation noted in provenance artifact
- **No useful results**: Provenance artifact notes "no significant context found"; main artifact unchanged for that theme/gap
- **Research contradicts documents**: Discrepancy surfaced in both artifacts for user attention
- **Re-extraction**: Research artifacts invalidated alongside main artifacts per cascade

## Limitations and Future Work

- Research uses `web_search` via subagent delegation; a future SDK-level Copilot deep research integration could provide richer results
- Source credibility is assessed at a basic level by the agent; no formal scoring system
- Re-research is full (not incremental) — when inputs change, all research re-runs
- Research is available only at Extraction and Correlation stages; Mapping, Journey Grounding, and Prioritization are not research-eligible
