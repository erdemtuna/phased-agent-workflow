# Discovery Scoping-Prioritization Merge Implementation Plan

## Overview

Merge the standalone journey-scoping stage into the prioritization/roadmap stage across the PAW Discovery workflow. JourneyMap becomes a pure vision artifact (no delivery decisions), Roadmap becomes the single decision point with three execution modes (interactive/guided/autonomous). This touches 8 files: 6 skill modifications, 1 skill removal, and 1 agent update.

## Current State Analysis

The Discovery workflow has 12 stages including a dedicated Journey Scoping checkpoint between Journey Grounding Review and Prioritization. Journey grounding produces JourneyMap.md with MVP Options (Full/Partial/Minimal), MVP Depth placeholders, Scoped placeholders, and MVP Critical columns — all deferred to the scoping stage. The scoping skill reads DiscoveryContext.md for a `Scoping Style` config field. The prioritize skill reads scoped JourneyMap fields (MVP Critical, Scoped) as factors 6-8. The Discovery agent orchestrates the scoping checkpoint as a direct-execution interactive stage.

## Desired End State

- JourneyMap.md is a pure analysis artifact: pain points, journeys, steps, feature mapping — zero delivery-decision fields
- No journey-scoping stage exists in the workflow (12 → 10 stages)
- Prioritization stage handles both scoping and ranking with `Prioritization Mode` config (interactive/guided/autonomous)
- Roadmap.md includes a scoping decisions section alongside priority categories
- Prioritize-review validates scoping consistency in addition to roadmap quality
- All prompt files pass `lint-prompting.sh`

## What We're NOT Doing

- Changing extraction, mapping, or correlation stages
- Changing the final-review stage
- Building migration tooling for existing DiscoveryContext.md files
- Changing the PAW implementation workflow (non-Discovery)
- Adding integration tests (prompt-only changes)

## Phase Status
- [ ] **Phase 1: Purify JourneyMap** - Remove scoping fields from grounding, remove scoping skill
- [ ] **Phase 2: Enhance Prioritization** - Add scoping logic and execution modes to prioritize skill
- [ ] **Phase 3: Update Workflow Infrastructure** - Update init, workflow skill, and Discovery agent
- [ ] **Phase 4: Documentation** - Docs.md and project documentation updates

## Phase Candidates
<!-- None currently -->

---

## Phase 1: Purify JourneyMap

Remove all scoping/delivery-decision fields from journey grounding and eliminate the journey-scoping skill entirely.

### Changes Required:

- **`skills/paw-discovery-journey-grounding/SKILL.md`**:
  - Remove `- **MVP Depth**: [To be set by scoping]` from journey template (~line 193)
  - Remove `#### MVP Options` section including Full/Partial/Minimal breakdowns and `**Scoped**` field (~lines 205-211)
  - Remove `MVP Critical` column from Feature-to-Journey Mapping table (~line 218)
  - Update any surrounding prose that references scoping handoff

- **`skills/paw-discovery-journey-grounding-review/SKILL.md`**:
  - Change description from "before the journey scoping checkpoint proceeds" to "before prioritization proceeds" (~line 10)
  - Change PASS verdict from "proceed to journey scoping" to "proceed to prioritization" (~line 60)
  - Change completion response from "Ready for journey scoping checkpoint" to "Ready for prioritization" (~line 103)
  - Remove any review criteria checking MVP options completeness

- **`skills/paw-discovery-journey-scoping/`**: Delete entire directory

### Success Criteria:

#### Automated Verification:
- [ ] Lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-journey-grounding/SKILL.md`
- [ ] Lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-journey-grounding-review/SKILL.md`
- [ ] Directory removed: `! test -d skills/paw-discovery-journey-scoping`
- [ ] Full lint: `npm run lint`

#### Manual Verification:
- [ ] JourneyMap template contains no MVP Options, MVP Depth, Scoped, or MVP Critical fields
- [ ] Grounding review makes no reference to scoping
- [ ] No broken cross-references remain in modified files

---

## Phase 2: Enhance Prioritization

Add scoping logic and three execution modes to the prioritize skill. Update the prioritize-review skill to validate scoping consistency.

### Changes Required:

- **`skills/paw-discovery-prioritize/SKILL.md`**:
  - Add `Prioritization Mode` configuration reading from DiscoveryContext.md (interactive/guided/autonomous)
  - Replace factors 6-8 (which currently read pre-computed scoping fields) with agent-driven scoping logic:
    - Factor 6 (Journey Criticality): Derive from journey steps' required features instead of MVP Critical column
    - Factor 7 (Pain Point Severity): Unchanged — reads pain points directly
    - Factor 8 (MVP Scope): Replaced by agent's own scoping decisions based on mode
  - Add three execution mode sections:
    - **Interactive**: Walk through journeys with user, collect depth decisions per journey, then rank
    - **Guided**: Ask user for high-level direction, agent applies scoping heuristics + ranks
    - **Autonomous**: Agent auto-scopes based on pain severity and feature criticality + ranks
  - Add scoping decisions section to Roadmap.md template (depth per journey + rationale)
  - Update elevation/demotion rules to reference agent-derived scope decisions instead of JourneyMap fields

- **`skills/paw-discovery-prioritize-review/SKILL.md`**:
  - Add scoping consistency validation criteria:
    - Depth decisions coherent across journeys (no conflicting feature assignments)
    - Scoping rationale documented per journey
    - Features required by in-scope journeys not categorized as Post-MVP without justification
  - Update existing criteria descriptions to reference the merged scoping+prioritization flow

### Success Criteria:

#### Automated Verification:
- [ ] Lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-prioritize/SKILL.md`
- [ ] Lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-prioritize-review/SKILL.md`
- [ ] Full lint: `npm run lint`

#### Manual Verification:
- [ ] Prioritize skill describes all three execution modes clearly
- [ ] Roadmap.md template includes scoping decisions section
- [ ] Prioritize-review checks both scoping consistency and roadmap quality
- [ ] Backward compatibility: skill describes fallback when JourneyMap lacks journey data

---

## Phase 3: Update Workflow Infrastructure

Update the Discovery init skill, workflow skill, and agent to reflect the reduced stage count and new configuration.

### Changes Required:

- **`skills/paw-discovery-init/SKILL.md`**:
  - Replace `Scoping Style` config section (~line 35) with `Prioritization Mode` section describing interactive/guided/autonomous
  - Replace `- **Scoping Style**: [per-journey|batch|bulk-guidance]` (~line 99) with `- **Prioritization Mode**: [interactive|guided|autonomous]`
  - Remove `| Journey Scoping | pending | - |` row from Stage Progress table (~line 114)
  - Update any references to stage count (12 → 10)

- **`skills/paw-discovery-workflow/SKILL.md`**:
  - Remove `paw-discovery-journey-scoping` row from activity table (~line 54)
  - Remove journey-scoping from default flow section (~line 140)
  - Remove journey-scoping from execution model list (~line 222)
  - Update stage descriptions to show direct grounding-review → prioritize flow

- **`agents/PAW Discovery.agent.md`**:
  - Remove transition rows: `journey-grounding-review → journey-scoping` and `journey-scoping → prioritize` (~lines 26-27)
  - Add direct transition: `journey-grounding-review → prioritize`
  - Update stage structure to remove scoping from Journey Grounding stage (~line 43)
  - Remove footnote about scoping being interactive checkpoint (~line 46)
  - Remove `paw-discovery-journey-scoping` from direct execution list (~lines 106-107)
  - Add `Prioritization Mode` to any configuration references

### Success Criteria:

#### Automated Verification:
- [ ] Lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-init/SKILL.md`
- [ ] Lint passes: `./scripts/lint-prompting.sh skills/paw-discovery-workflow/SKILL.md`
- [ ] Lint passes: `./scripts/lint-prompting.sh agents/PAW\ Discovery.agent.md`
- [ ] Full lint: `npm run lint`
- [ ] Agent lint: `npm run lint:agent:all`

#### Manual Verification:
- [ ] DiscoveryContext.md template shows Prioritization Mode instead of Scoping Style
- [ ] Stage Progress table has 10 rows (no Journey Scoping)
- [ ] Workflow activity table has no journey-scoping entry
- [ ] Discovery agent transitions directly from grounding-review to prioritize

---

## Phase 4: Documentation

Create Docs.md and update project documentation to reflect the workflow change.

### Changes Required:

- **`.paw/work/discovery-scoping-prioritization-merge/Docs.md`**: Technical reference documenting the merge (load `paw-docs-guidance`)
- **Project docs**: Check `docs/` for Discovery workflow references that mention journey scoping or the old stage count — update as needed
- **Specification docs**: Check if `paw-specification.md` or `paw-review-specification.md` reference Discovery scoping stages

### Success Criteria:
- [ ] Docs build: `source .venv/bin/activate && mkdocs build --strict`
- [ ] Content accurate, style consistent
- [ ] No stale references to journey scoping in project docs

---

## References
- Spec: `.paw/work/discovery-scoping-prioritization-merge/Spec.md`
- Research: `.paw/work/discovery-scoping-prioritization-merge/CodeResearch.md`
- WorkShaping: `.paw/work/discovery-scoping-prioritization-merge/WorkShaping.md`
