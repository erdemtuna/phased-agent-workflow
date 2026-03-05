# Feature Specification: Merge Discovery Scoping into Prioritization

**Branch**: feature/discovery-scoping-prioritization-merge  |  **Created**: 2026-03-05  |  **Status**: Draft
**Input Brief**: Eliminate redundant priority-like decisions in the Discovery workflow by merging journey scoping into the prioritization/roadmap stage.

## Overview

The PAW Discovery workflow currently asks users to make priority-adjacent decisions at three separate stages: pain point severity validation during journey grounding, MVP depth selection during journey scoping, and roadmap ordering during prioritization. While each stage frames its decisions differently (severity vs. scope vs. rank), from the user's perspective they all answer the same meta-question: "what matters most?" This creates decision fatigue and a sense of redundancy.

This feature restructures the Discovery workflow around a clearer mental model: **JourneyMap is the vision** (the ideal full experience) and **Roadmap is the cut** (what ships in V1). Journey grounding becomes pure analysis — producing pain points, journeys, and feature mappings without any delivery decisions. The standalone journey-scoping stage is eliminated entirely, with its logic merging into an enhanced prioritization stage that supports three execution modes: interactive, guided, and autonomous.

The result is a streamlined five-stage analytical pipeline (down from six) where the user makes delivery decisions exactly once — at the roadmap stage — with flexibility to choose their level of involvement.

## Objectives

- Eliminate redundant user decision points by consolidating scoping and prioritization into a single stage
- Establish JourneyMap as a pure analytical artifact with no delivery-decision fields
- Provide flexible execution modes (interactive/guided/autonomous) for the merged prioritization stage
- Reduce the Discovery stage count from 12 to 11 (removing scoping)

## User Scenarios & Testing

### User Story P1 – Streamlined Prioritization with Mode Selection
Narrative: A user initializes a Discovery workflow and selects a prioritization mode at init time. After journey grounding completes with a pure-analysis JourneyMap, the workflow transitions directly to prioritization. The agent handles both scoping and ranking in a single pass according to the selected mode.
Independent Test: A Discovery workflow initialized with `prioritization_mode: guided` skips journey scoping and produces a Roadmap.md that contains both scope decisions and priority rankings.
Acceptance Scenarios:
1. Given a Discovery workflow initialized with `prioritization_mode: interactive`, When the prioritization stage runs, Then the agent walks through journeys with the user, collects depth decisions, and produces a prioritized roadmap — all in one stage.
2. Given a Discovery workflow initialized with `prioritization_mode: guided`, When the prioritization stage runs, Then the agent asks for high-level direction, applies scoping heuristics, and generates the roadmap without per-journey prompting.
3. Given a Discovery workflow initialized with `prioritization_mode: autonomous`, When the prioritization stage runs, Then the agent auto-scopes based on pain severity and correlation data and generates the roadmap without user input.

### User Story P2 – Pure-Analysis JourneyMap
Narrative: A user runs journey grounding and receives a JourneyMap that describes the full vision — pain points, journeys, and feature mappings — without any MVP options, scoping fields, or delivery decisions.
Independent Test: After journey grounding completes, JourneyMap.md contains no `MVP Options`, `MVP Depth`, `Scoped`, or `MVP Critical` fields.
Acceptance Scenarios:
1. Given journey grounding has completed, When the user reads JourneyMap.md, Then it contains pain points with auto-assigned severity, user journeys with steps and feature requirements, and a feature-to-journey mapping — but no MVP depth options, no scoping fields, and no MVP Critical column.
2. Given a JourneyMap.md produced by the updated grounding skill, When the prioritization stage reads it, Then it discovers scoping options itself rather than relying on pre-computed MVP Options.

### User Story P3 – Expanded Roadmap Review
Narrative: A reviewer validates the Roadmap.md and checks both the prioritization logic and the scoping consistency in a single review pass.
Independent Test: The prioritize-review skill validates scoping decisions (journey depth coherence) alongside roadmap quality.
Acceptance Scenarios:
1. Given a Roadmap.md containing scoping decisions, When the prioritize-review runs, Then it checks that depth decisions are coherent across journeys and that the priority ranking respects scoping constraints.

### Edge Cases
- JourneyMap.md is absent entirely — prioritization uses base factors (1-5) only, same as current fallback behavior.

## Requirements

### Functional Requirements

- FR-001: The `paw-discovery-init` skill replaces the `Scoping Style` configuration field with `Prioritization Mode` accepting values: `interactive`, `guided`, `autonomous` (Stories: P1)
- FR-002: The `paw-discovery-init` skill removes Journey Scoping and Journey Scoping Review rows from the Stage Progress table, reducing from 12 to 10 stages (Stories: P1)
- FR-003: The `paw-discovery-journey-grounding` skill removes MVP Options (Full/Partial/Minimal) from each journey, removes the `MVP Depth` placeholder, removes the `Scoped` placeholder, and removes the `MVP Critical` column from Feature-to-Journey Mapping (Stories: P2)
- FR-004: The `paw-discovery-journey-grounding-review` skill removes any checks related to MVP options completeness or scoping-readiness (Stories: P2)
- FR-005: The `paw-discovery-prioritize` skill adds scoping logic that discovers journey depth options from JourneyMap data and makes scope + priority decisions based on the configured Prioritization Mode (Stories: P1)
- FR-006: The `paw-discovery-prioritize` skill supports three execution modes — interactive (user walks through journeys + validates roadmap), guided (user gives direction, agent applies), autonomous (fully agent-driven) (Stories: P1)
- FR-007: The `paw-discovery-prioritize` skill produces a Roadmap.md that includes a scoping decisions section (depth per journey with rationale) alongside the existing priority categories and items (Stories: P1)
- FR-008: The `paw-discovery-prioritize-review` skill validates scoping consistency (coherent depth decisions across journeys) in addition to existing roadmap quality checks (Stories: P3)
- FR-009: The `paw-discovery-workflow` skill updates its stage sequence and activity table to remove journey-scoping references (Stories: P1, P2)
- FR-010: The Discovery agent removes journey-scoping from its orchestration flow, transitioning directly from journey-grounding-review to prioritization (Stories: P1)
- FR-011: The `paw-discovery-journey-scoping` skill and `paw-discovery-journey-scoping-review` skill (if exists) are removed (Stories: P1)
- FR-012: All six combinations of `review_policy` × `prioritization_mode` are allowed without restriction (Stories: P1)

### Cross-Cutting / Non-Functional
- Token efficiency: The merged prioritization skill should not significantly increase prompt size compared to the combined old scoping + prioritization skills

## Success Criteria

- SC-001: A Discovery workflow runs end-to-end with no journey-scoping stage invocation (FR-002, FR-010, FR-011)
- SC-002: JourneyMap.md produced by journey grounding contains zero MVP/scoping fields (FR-003)
- SC-003: Roadmap.md contains a scoping decisions section with depth per journey and rationale (FR-005, FR-007)
- SC-004: Each prioritization mode (interactive/guided/autonomous) produces a valid Roadmap.md with scoping + priorities (FR-006)
- SC-005: DiscoveryContext.md at init shows `Prioritization Mode` field instead of `Scoping Style`, and the Stage Progress table has 10 rows (FR-001, FR-002)
- SC-006: Prioritize-review validates both scoping consistency and roadmap quality in a single pass (FR-008)

## Assumptions

- The `paw-discovery-journey-scoping-review` skill exists and should be removed alongside the scoping skill (will verify during code research)
- No external consumers depend on the MVP Options / Scoped fields in JourneyMap.md beyond the Discovery workflow itself
- The three prioritization modes (interactive/guided/autonomous) map to the same interaction patterns as existing Discovery features (extraction Q&A, review policy), requiring no new UX primitives

## Scope

In Scope:
- Modifying journey-grounding skill (remove scoping fields from JourneyMap template)
- Modifying journey-grounding-review skill (remove scoping-readiness checks)
- Removing journey-scoping skill
- Removing journey-scoping-review skill (if exists)
- Modifying prioritize skill (add scoping logic + 3 modes)
- Modifying prioritize-review skill (add scoping validation)
- Modifying workflow skill (update stage sequence)
- Modifying Discovery agent (update orchestration)
- Modifying init skill (update DiscoveryContext.md template)

Out of Scope:
- Changes to extraction, mapping, or correlation stages
- Changes to the final-review stage
- Migration tooling for existing DiscoveryContext.md files
- Changes to the PAW implementation workflow (non-Discovery)

## Dependencies

- Existing journey-grounding skill implementation
- Existing prioritize skill implementation
- Discovery agent orchestration logic

## Risks & Mitigations

- **Risk**: Removing MVP Options from JourneyMap reduces context for the prioritization stage, making it harder to discover scope options. **Mitigation**: The prioritization skill derives scope options from journey step data (which steps are required, which are optional) rather than pre-computed options.
- **Risk**: Merging two skills' logic into one increases the prioritize skill's complexity and token cost. **Mitigation**: The scoping logic adds mode-dependent interaction (not a full duplication of the old skill); guided and autonomous modes are shorter than the old interactive scoping flow.

## References

- WorkShaping: .paw/work/discovery-scoping-prioritization-merge/WorkShaping.md
