# Discovery Research Integration Implementation Plan

## Overview

Integrate opt-in web research into PAW Discovery's Extraction and Correlation stages. Research enriches extracted themes with external context (standards, approaches, prior art) and assesses gap feasibility (libraries, patterns, solutions). All changes are to skill markdown files (SKILL.md) and the Discovery agent prompt — no TypeScript code changes.

## Current State Analysis

The Discovery pipeline has five stages, each implemented as a pure markdown skill in `skills/paw-discovery-*/SKILL.md`. Key observations from CodeResearch.md:

- **Init** (`skills/paw-discovery-init/SKILL.md:22-28`): Accepts 5 configuration parameters — no research field exists. DiscoveryContext.md template at lines 88-121 has Configuration section with Review Policy, Prioritization Mode, Final Review fields.
- **Extraction** (`skills/paw-discovery-extraction/SKILL.md`): Runs directly in session. Processes documents → extracts themes → interactive Q&A → produces Extraction.md. No pre-artifact research step exists.
- **Correlation** (`skills/paw-discovery-correlation/SKILL.md`): Runs directly in session. Loads Extraction.md + CapabilityMap.md → analyzes match/gap/combination/partial → produces Correlation.md. No feasibility research exists.
- **Mapping delegation pattern** (`skills/paw-discovery-mapping/SKILL.md:79-108`): Uses `task(agent_type: "general-purpose", prompt: "...")` to spawn subagent for code research. This is the reusable pattern for research subagent invocation.
- **Reviews**: 6-category checklist → PASS/REVISE verdict. No research quality criteria exist.
- **Prioritization modes** (`skills/paw-discovery-prioritize/SKILL.md:12-16`): Reads mode from DiscoveryContext.md Configuration section — pattern to reuse for research config.
- **Cascade invalidation** (`agents/PAW Discovery.agent.md:74-88`): Marks downstream artifacts stale — research artifacts not included.

## Desired End State

- Discovery init offers opt-in research configuration persisted in DiscoveryContext.md
- Extraction stage supports a pre-artifact research phase via subagent with web_search, producing ExtractionResearch.md and synthesizing findings into Extraction.md
- Correlation stage supports a pre-artifact research phase via subagent with web_search, producing CorrelationResearch.md and synthesizing findings into Correlation.md
- Both stages support per-stage execution mode selection (autonomous/guided/skip)
- Review skills evaluate research integration quality when research was conducted
- Discovery agent includes research artifacts in cascade invalidation
- Workflow reference documentation reflects new artifacts and capabilities
- All modified skills pass the prompting linter

## What We're NOT Doing

- Research at Mapping, Journey Grounding, or Prioritization stages
- Creating a separate `paw-discovery-research` skill — research subagent prompts are constructed inline within extraction/correlation skills (following mapping's delegation pattern but without a dedicated target skill, since web research instructions are stage-specific and simpler than code research)
- SDK-level Copilot research mode integration
- Formal source credibility scoring system
- Incremental re-research (full re-research on cascade)
- TypeScript code changes (all changes are prompt/agent markdown)
- Integration test additions (prompt-only changes with no behavioral contract changes in TypeScript)

## Phase Status

- [ ] **Phase 1: Configuration & References** - Add research config to init, update workflow reference and artifact structure
- [ ] **Phase 2: Extraction Research** - Add research phase to extraction skill with subagent delegation
- [ ] **Phase 3: Correlation Research** - Add research phase to correlation skill with subagent delegation
- [ ] **Phase 4: Reviews & Orchestrator** - Extend review criteria and update cascade invalidation
- [ ] **Phase 5: Documentation** - Technical reference and project documentation updates

## Phase Candidates

_(No unresolved phase candidates — all scope decisions finalized in Spec.md)_

---

## Phase 1: Configuration & References

### Changes Required

- **`skills/paw-discovery-init/SKILL.md`**:
  - Add `research` parameter to the parameter table (`SKILL.md:22-28`) with default `disabled`, values `enabled`/`disabled`
  - Update the confirmation prompt template (`SKILL.md:56-63`) to display Research setting
  - Extend DiscoveryContext.md template's Configuration section (`SKILL.md:97-101`) with `Research: [enabled/disabled]`
  - No changes to directory structure creation (research artifacts are created by activity skills, not init)

- **`skills/paw-discovery-workflow/SKILL.md`**:
  - Update artifact directory structure (`SKILL.md:60-73`) to include `ExtractionResearch.md` and `CorrelationResearch.md` (marked as optional/conditional)
  - Update DiscoveryContext.md template in this skill (`SKILL.md:77-117`) to include Research configuration field, matching the init skill's template
  - Add brief mention of research capability in the activities table notes for Extraction and Correlation (`SKILL.md:44-57`)

### Success Criteria

#### Automated Verification
- [ ] Prompt lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-init/SKILL.md`
- [ ] Prompt lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-workflow/SKILL.md`

#### Manual Verification
- [ ] Init parameter table includes research with correct default and values
- [ ] Confirmation template displays Research setting
- [ ] DiscoveryContext.md template has Research field in Configuration section
- [ ] Workflow artifact directory shows optional research artifacts
- [ ] Both templates are consistent with each other regarding the research field

---

## Phase 2: Extraction Research

### Changes Required

- **`skills/paw-discovery-extraction/SKILL.md`**:
  - Add a **Research Phase** section between document conversion/theme identification and interactive Q&A/artifact creation
  - Research phase reads `Research` field from DiscoveryContext.md; if `disabled`, skip entirely (FR-011)
  - When enabled, prompt user for execution mode: autonomous / guided / skip (FR-002)
    - **Autonomous**: Agent identifies ambiguous/novel themes from preliminary extraction, constructs research questions, delegates to subagent
    - **Guided**: Agent presents identified themes with recommendations, user selects subset, then delegates to subagent
    - **Skip**: Proceed without research for this stage
  - Subagent delegation follows mapping pattern (`task(agent_type: "general-purpose", prompt: "...")`). The prompt instructs the subagent to:
    - Use `web_search` tool to investigate specified themes
    - Research context: industry standards, common approaches, terminology, prior art (enrichment, NOT feasibility)
    - Produce `ExtractionResearch.md` at `.paw/discovery/<work-id>/ExtractionResearch.md` with YAML frontmatter, per-theme findings, and source URLs
  - After research completes, integrate findings into theme extraction — research-sourced insights marked with source attribution distinguishing them from document-sourced insights
  - Update Extraction.md YAML frontmatter (`SKILL.md:194-211`) to include `research_conducted: true/false`
  - Graceful degradation: if web_search fails or returns nothing useful, note limitation and proceed (FR-013)
  - Update re-extraction flow (`SKILL.md:289-293`) to invalidate ExtractionResearch.md alongside Extraction.md

### Success Criteria

#### Automated Verification
- [ ] Prompt lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-extraction/SKILL.md`

#### Manual Verification
- [ ] Research phase is clearly positioned before artifact creation
- [ ] All three execution modes (autonomous/guided/skip) are described
- [ ] Subagent delegation prompt covers enrichment scope (standards, approaches, terminology, prior art)
- [ ] ExtractionResearch.md artifact format is defined with YAML frontmatter and per-theme structure
- [ ] Source attribution distinguishes document vs research origins in Extraction.md
- [ ] Graceful degradation path is described
- [ ] Re-extraction includes research artifact invalidation
- [ ] When research is disabled, no behavioral change from current skill

---

## Phase 3: Correlation Research

### Changes Required

- **`skills/paw-discovery-correlation/SKILL.md`**:
  - Add a **Research Phase** section between initial correlation analysis (match/gap identification) and artifact creation
  - Research phase reads `Research` field from DiscoveryContext.md; if `disabled`, skip entirely (FR-011)
  - When enabled, prompt user for execution mode: autonomous / guided / skip (FR-002)
    - **Autonomous**: Agent identifies all gaps, delegates research for each
    - **Guided**: Agent presents gaps with recommendations for which to research, user selects subset
    - **Skip**: Proceed without research for this stage
  - Subagent delegation follows same pattern as extraction research. The prompt instructs the subagent to:
    - Use `web_search` tool to investigate specified gaps
    - Research context: existing libraries, implementation patterns, ecosystem solutions, effort indicators (feasibility, NOT enrichment)
    - Produce `CorrelationResearch.md` at `.paw/discovery/<work-id>/CorrelationResearch.md` with YAML frontmatter, per-gap findings, and source URLs
  - After research completes, integrate feasibility signals into gap entries in Correlation.md — each gap section includes research-sourced feasibility context
  - Update Correlation.md YAML frontmatter (`SKILL.md:139-149`) to include `research_conducted: true/false`
  - Graceful degradation: if web_search fails or returns nothing useful, note limitation and proceed (FR-013)

### Success Criteria

#### Automated Verification
- [ ] Prompt lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-correlation/SKILL.md`

#### Manual Verification
- [ ] Research phase is clearly positioned after initial match/gap analysis but before artifact creation
- [ ] All three execution modes (autonomous/guided/skip) are described
- [ ] Subagent delegation prompt covers feasibility scope (libraries, patterns, solutions, effort)
- [ ] CorrelationResearch.md artifact format is defined with YAML frontmatter and per-gap structure
- [ ] Gap entries in Correlation.md include feasibility context from research
- [ ] Graceful degradation path is described
- [ ] When research is disabled, no behavioral change from current skill

---

## Phase 4: Reviews & Orchestrator

### Changes Required

- **`skills/paw-discovery-extraction-review/SKILL.md`**:
  - Add 7th review criteria category: **Research Integration** (`SKILL.md` after line ~51)
  - Criteria: source credibility assessed, research findings well-integrated into themes, document-sourced vs research-sourced insights distinguishable, research gaps flagged
  - Conditional application: only evaluate when `research_conducted: true` in Extraction.md frontmatter; otherwise skip this category
  - Update quality thresholds table (`SKILL.md:78-85`) with research integration thresholds

- **`skills/paw-discovery-correlation-review/SKILL.md`**:
  - Add 7th review criteria category: **Research Integration** (mirroring extraction review pattern)
  - Criteria: gap feasibility evidence credible, research findings reflected in gap entries, source quality adequate, research limitations noted
  - Conditional application: only evaluate when `research_conducted: true` in Correlation.md frontmatter
  - Update quality thresholds table (`SKILL.md:74-81`)

- **`skills/paw-discovery-final-review/SKILL.md`**:
  - Add 8th review criterion to the criteria list (`SKILL.md:63-76`): Research Integration quality — whether research findings are properly synthesized into main artifacts
  - Conditional: only when research artifacts exist in the discovery directory

- **`agents/PAW Discovery.agent.md`**:
  - Update cascade invalidation logic (`agent.md:74-88`) to include `ExtractionResearch.md` when extraction re-runs and `CorrelationResearch.md` when correlation re-runs
  - Update the execution model documentation (`agent.md:99-113`) to note that extraction and correlation stages may spawn research subagents

### Success Criteria

#### Automated Verification
- [ ] Prompt lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-extraction-review/SKILL.md`
- [ ] Prompt lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-correlation-review/SKILL.md`
- [ ] Prompt lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-final-review/SKILL.md`
- [ ] Prompt lint passes: `./scripts/lint-prompting.sh agents/PAW\ Discovery.agent.md`

#### Manual Verification
- [ ] Extraction review has conditional 7th category with clear criteria
- [ ] Correlation review has conditional 7th category with clear criteria
- [ ] Final review has conditional 8th criterion
- [ ] All review categories gracefully skip when research was not conducted
- [ ] Cascade invalidation includes both research artifacts
- [ ] Execution model docs reflect research subagent possibility

---

## Phase 5: Documentation

### Changes Required

- **`.paw/work/discovery-research-integration/Docs.md`**: Technical reference capturing implementation details (load `paw-docs-guidance` for template)
- **`docs/specification/`**: Update Discovery workflow specification if it documents stage flows — add research as optional sub-phase within Extraction and Correlation
- **`docs/guide/`**: Update Discovery user guide if it documents the init configuration or stage-by-stage flow

### Success Criteria

- [ ] Docs build passes: `source .venv/bin/activate && mkdocs build --strict`
- [ ] Docs.md captures all modified files, research artifact formats, configuration schema, and execution mode behavior
- [ ] Project docs accurately reflect the opt-in research capability

---

## References

- Spec: `.paw/work/discovery-research-integration/Spec.md`
- Research: `.paw/work/discovery-research-integration/CodeResearch.md`
- WorkShaping: `.paw/work/discovery-research-integration/WorkShaping.md`
