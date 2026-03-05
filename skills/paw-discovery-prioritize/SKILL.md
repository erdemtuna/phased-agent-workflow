---
name: paw-discovery-prioritize
description: Prioritization activity skill for PAW Discovery workflow. Merges scoping and multi-factor prioritization into a single stage with configurable execution modes to produce MVP roadmap with PAW handoff option.
---

# Discovery Prioritization

> **Execution Context**: This skill runs **directly** in the PAW Discovery session, as scoping and tradeoff discussion require interactive user collaboration.

Apply scoping and multi-factor prioritization to produce a ranked MVP roadmap. This stage is the single decision point for delivery scope — JourneyMap provides the vision, Roadmap determines what ships.

## Configuration

Read DiscoveryContext.md for:
- `Prioritization Mode`: `interactive` | `guided` | `autonomous`

If `Prioritization Mode` is missing or unrecognized, default to `guided`.

## Capabilities

- Read Correlation.md for theme-capability relationships
- Read JourneyMap.md for journey context and pain point severity (if exists)
- Discover scoping options from journey step data (which steps are required vs optional)
- Make scope decisions (journey depth) based on configured Prioritization Mode
- Apply multi-factor prioritization framework (5 base factors + 3 journey factors)
- Categorize items into MVP-Critical / MVP-Nice-to-Have / Post-MVP
- Generate prioritized roadmap with scoping decisions
- Offer PAW workflow handoff for top item
- Generate Roadmap.md artifact

## Prioritization Factors

### 1. Value
**Question**: How much user/business value does this deliver?

| Score | Meaning |
|-------|---------|
| High | Core to user workflow, high demand, competitive differentiator |
| Medium | Useful enhancement, some user demand |
| Low | Nice-to-have, minimal user impact |

### 2. Effort
**Question**: How much work is required to implement?

| Score | Meaning |
|-------|---------|
| Low | Small change, existing patterns, clear path |
| Medium | Moderate work, some complexity |
| High | Large change, new patterns, significant complexity |

Consider correlation type: matches/partials = lower effort; gaps = higher effort

### 3. Dependencies
**Question**: What must be built first? What does this enable?

| Score | Meaning |
|-------|---------|
| Blocker | Nothing else can proceed without this |
| Enabler | Unlocks multiple downstream items |
| Independent | Can be built in isolation |

### 4. Risk
**Question**: What could go wrong? How uncertain is this?

| Score | Meaning |
|-------|---------|
| Low | Well-understood, proven approach |
| Medium | Some unknowns, manageable risk |
| High | Significant unknowns, potential for failure |

### 5. Leverage
**Question**: How much does this leverage existing capabilities?

| Score | Meaning |
|-------|---------|
| High | Extends existing capabilities significantly |
| Medium | Some reuse, some new work |
| Low | Mostly net-new, limited reuse |

From Correlation.md: matches/combinations = high leverage; gaps = low leverage

### 6. Journey Criticality (derived from JourneyMap.md)
**Question**: Is this feature required for a user journey?

| Score | Meaning |
|-------|---------|
| Critical | Required by multiple journeys or by a journey addressing High-severity pain points |
| Supporting | Enhances a journey but not strictly required for any step |
| Unlinked | Not required for any defined journey |

Derive from JourneyMap.md Feature-to-Journey Mapping: check which journeys require this feature and the severity of pain points those journeys address.

### 7. Pain Point Severity (from JourneyMap.md)
**Question**: How severe is the pain point this feature addresses?

| Score | Meaning |
|-------|---------|
| High | Addresses a High-severity pain point |
| Medium | Addresses a Medium-severity pain point |
| Low | Addresses a Low-severity pain point or no specific pain point |

From JourneyMap.md Pain Points section: cross-reference which pain points the feature's journey addresses.

### 8. Scoped Depth (determined during this stage)
**Question**: Is this feature within the scoped MVP depth decided during prioritization?

| Score | Meaning |
|-------|---------|
| In-Scope | Required for the MVP depth selected for its journey |
| Deferred | Only needed for deeper journey depth than selected |
| N/A | Feature not linked to any journey |

Scope decisions are made during this stage based on the configured Prioritization Mode (see Execution Modes below). The agent analyzes journey steps to determine which features are needed at different depth levels.

### Depth Derivation

When scoping journeys, classify steps into depth levels based on journey structure:
- **Full**: All journey steps included
- **Partial**: Core steps that deliver the primary user goal, excluding extended/optional steps
- **Minimal**: Only the essential steps required to address the primary pain point

Determine boundaries from step dependencies and feature criticality — steps addressing High-severity pain points form the minimal core.

## JourneyMap.md Fallback Behavior

When JourneyMap.md is absent or incomplete (e.g., skipped Journey Grounding stage, or missing `## User Journeys` / `## Feature-to-Journey Mapping` sections):

- **Factors 6-8**: Mark as N/A in scoring; omit journey-related rationale from Roadmap.md
- **Scoping**: Skip — no journeys to scope
- **Priority calculation**: Use base factors (1-5) only

## Journey Factor Integration

When JourneyMap.md exists, integrate journey factors into prioritization:

**Elevation rules**:
- Features required by journeys addressing High-severity pain points get priority boost
- Features required by multiple journeys get priority boost
- Features within the scoped MVP depth are favored over deferred features

**Demotion rules**:
- Features not required for any journey may be deprioritized (unless high standalone value)
- Features only needed for deeper journey depth than selected scope are deferred

**Rationale inclusion**:
- Roadmap.md item descriptions should reference journey criticality
- "Required for [Journey Name] MVP" or "Deferred: only needed at full depth for [Journey Name]"

## Execution Modes

Read `Prioritization Mode` from DiscoveryContext.md Configuration section.

### Interactive Mode

User makes explicit scoping and prioritization decisions. The agent presents each journey with depth options for the user to choose, scopes features accordingly, then presents the full prioritized roadmap for validation. If the user skips remaining journeys, default unscoped journeys to Full depth. One adjustment pass before finalizing.

### Guided Mode

User provides high-level direction (e.g., "focus on onboarding, defer admin features"). The agent interprets guidance to determine journey depths and feature scoping, then generates the complete roadmap. One adjustment pass before finalizing.

### Autonomous Mode

Agent determines all scoping and prioritization decisions based on pain severity and feature criticality. Higher-severity pain points drive deeper journey scope. Presents final roadmap with one adjustment opportunity.

### Priority Categories (Not Numbers)

Use meaningful categories instead of numeric priority:

| Category | Meaning | Criteria |
|----------|---------|----------|
| **MVP-Critical** | Must ship in MVP | High value + blocker/enabler + manageable effort |
| **MVP-Nice-to-Have** | Should ship if time allows | Medium-high value, no blockers |
| **Post-MVP** | Defer to future releases | Lower value OR high risk OR high effort |

Within each category, order by dependency (blockers first) then value-to-effort ratio.

## Roadmap Generation

### Priority Calculation

Suggested priority formula (can be adjusted):
- High priority: High value + (Blocker or Enabler) + Low/Medium effort + Journey Critical
- Medium priority: Medium value OR High effort with High value OR Supporting journey role
- Low priority: Low value OR High risk without mitigation OR Unlinked to journeys

When JourneyMap.md exists, journey criticality can override base scoring:
- A medium-value feature that is journey-critical may be elevated to MVP-Critical
- A high-value feature not required for any journey may be deprioritized to Nice-to-Have

### Ranking

Order items by:
1. Dependency order (blockers first)
2. Journey criticality (MVP-critical journeys first)
3. Value-to-effort ratio
4. Pain point severity
5. Risk (lower risk preferred for MVP)

## PAW Handoff

### When to Offer

After roadmap generation:
1. Identify the top-priority item
2. Offer to initiate PAW workflow
3. If user accepts, prepare handoff brief

### Handoff Brief

Generate brief for PAW workflow including:
- Work title from roadmap item name
- Description from item rationale
- Reference to Discovery artifacts
- Relevant capabilities from CapabilityMap.md
- Context from Correlation.md

## Roadmap.md Artifact

Save to: `.paw/discovery/<work-id>/Roadmap.md`

### Template

```markdown
---
date: [ISO timestamp]
work_id: [work-id]
mvp_critical_count: [N]
mvp_nice_to_have_count: [N]
post_mvp_count: [N]
top_item: [item-name]
paw_handoff_ready: true
status: complete
---

# Roadmap: [Work Title]

## Summary

[2-3 sentences describing the prioritization outcome and recommended first steps]

## Scoping Decisions

| Journey | Depth | Rationale |
|---------|-------|-----------|
| J-1: [name] | [Full/Partial/Minimal] | [Why this depth was selected] |
| J-2: [name] | [Full/Partial/Minimal] | [Why this depth was selected] |

## Priority Categories

| Category | Count | Description |
|----------|-------|-------------|
| MVP-Critical | N | Must ship in MVP |
| MVP-Nice-to-Have | N | Should ship if time allows |
| Post-MVP | N | Deferred to future releases |

## Prioritized Items

### MVP-Critical

Each MVP-Critical item includes a **self-contained handoff brief** so any item can start a PAW workflow without cross-referencing other Discovery artifacts.

#### [Item Name]
- **Theme**: [Theme ID]: [Full description from Extraction.md — not just ID]
- **What exists**: [Relevant capability summary from CapabilityMap.md]
- **Integration path**: [From Correlation.md — Match/Extend/Gap + what's needed]
- **Why critical**: [Rationale - value, dependency role, blockers]
- **Constraints**: [Relevant C1-Cn constraints that apply]
- **Success criteria**: [What "done" looks like for this item]
- **Suggested approach**: [Implementation guidance if correlation provided insights]

#### [Item Name]
...

### MVP-Nice-to-Have

Nice-to-Have items use condensed format (full details available in Extraction/Correlation):

#### [Item Name]
- **Theme**: [Theme ID and brief description]
- **Correlation**: [Match/Gap/Combination]
- **Why nice-to-have**: [Rationale]

### Post-MVP

#### [Item Name]
- **Theme**: [Theme ID and description]
- **Why deferred**: [Rationale - high effort, risk, lower priority, etc.]

## How to Use This Roadmap

### For Humans
- **Quick review**: Read Summary + MVP-Critical items
- **Deep dive**: Each MVP-Critical item is self-contained with context, constraints, and success criteria
- **Cross-reference**: Nice-to-Have and Post-MVP items reference theme IDs — see Extraction.md for full details

### For PAW Handoff
- **Any MVP-Critical item** can start a PAW workflow directly
- Copy the item's content as the initial brief
- Discovery artifacts path provided for additional context during Spec/Planning

## PAW Handoff Brief

**Ready for implementation**: [Top item name]

(Note: Any MVP-Critical item above can serve as a PAW handoff brief. This section highlights the recommended first item.)

### Work Title
[Derived from top priority item]

### Description
[Brief for PAW workflow initialization]

### Context
- Discovery artifacts: `.paw/discovery/[work-id]/`
- Related capabilities: [CAP IDs from CapabilityMap.md]
- Correlation type: [Match/Gap/Combination]

### Suggested Approach
[Any implementation guidance from correlation analysis]
```

## Quality Checklist

- [ ] All correlated themes considered for roadmap
- [ ] Base factors (1-5) scored for each item
- [ ] Journey factors (6-8) scored when JourneyMap.md exists
- [ ] Scoping decisions documented with rationale per journey (when JourneyMap.md exists)
- [ ] Items categorized as MVP-Critical / MVP-Nice-to-Have / Post-MVP
- [ ] Dependency order is logical within categories
- [ ] Scoping decisions are coherent (no conflicting feature assignments across journeys)
- [ ] Top MVP-Critical item identified for PAW handoff
- [ ] PAW handoff brief is actionable
- [ ] YAML frontmatter is complete

## Completion Response

Report to PAW Discovery agent:
- Artifact path: `.paw/discovery/<work-id>/Roadmap.md`
- Item count and priority distribution
- Top priority item identified
- PAW handoff ready (yes/no)
- Ready for prioritization review
