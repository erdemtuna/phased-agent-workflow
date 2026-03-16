# Work Shaping: Merge Discovery Scoping into Prioritization

## Problem

The PAW Discovery workflow asks users to make priority-like decisions three times:

1. **Journey Grounding** — validate pain point severity (High/Med/Low)
2. **Journey Scoping** — choose MVP depth per journey (Full/Partial/Minimal)
3. **Prioritization** — review/adjust ranked roadmap

From the user's perspective, all three answer the same meta-question: *"what matters most?"* This creates decision fatigue and a sense of redundancy.

## Core Insight

> **JourneyMap = the vision** (ideal full experience)
> **Roadmap = the cut** (what ships in V1)

JourneyMap should describe the complete picture without making delivery decisions. Roadmap is where scoping and prioritization happen as a single activity.

## Proposed Changes

### 1. Journey Grounding — stays, but becomes pure analysis

- Produces pain points, user journeys, feature-to-journey mapping
- **Removes** MVP depth options (Full/Partial/Minimal) from JourneyMap
- **Removes** any scoping-related fields (MVP Critical column stays TBD or is removed)
- Pain point severity is auto-assigned, not user-validated
- Interactive Q&A remains for journey synthesis quality, not for priority decisions

### 2. Journey Scoping — removed as standalone stage

- Eliminated from the Discovery workflow stage sequence
- Its logic (depth selection, MVP-critical marking) merges into the Roadmap stage
- The `paw-discovery-journey-scoping` skill is deprecated/removed
- The `paw-discovery-journey-grounding-review` no longer checks for scoping readiness

### 3. Prioritization/Roadmap — becomes the single decision point

Merges scoping + prioritization with three execution modes:

| Mode | User involvement | How it works |
|------|-----------------|--------------|
| **Interactive** | High | User walks through journeys, picks depths, validates resulting prioritized roadmap in one pass |
| **Guided** | Medium | User provides high-level direction (e.g., "focus on onboarding, defer admin"), agent applies scoping + generates roadmap |
| **Autonomous** | Low | Agent auto-scopes based on pain severity and correlation data, auto-generates roadmap; user reviews final output only |

The Roadmap stage:
- Reads JourneyMap.md for journey/pain-point understanding (vision context)
- Discovers scoping options itself (what's possible at Full/Partial/Minimal)
- Makes scoping + priority decisions based on selected mode
- Produces Roadmap.md with both scope decisions and priority ranking

### 4. Roadmap Review — expanded scope

- `paw-discovery-prioritize-review` now validates both:
  - Scoping consistency (are depth decisions coherent across journeys?)
  - Roadmap quality (are priorities well-justified, dependencies respected?)
- Replaces the now-removed `paw-discovery-journey-scoping` review

## Affected Skills

| Skill | Action |
|-------|--------|
| `paw-discovery-journey-grounding` | Modify — remove MVP options, scoping fields |
| `paw-discovery-journey-grounding-review` | Modify — remove scoping-readiness checks |
| `paw-discovery-journey-scoping` | **Remove** |
| `paw-discovery-journey-scoping-review` | **Remove** (if exists) |
| `paw-discovery-prioritize` | Modify — add scoping logic + 3 execution modes |
| `paw-discovery-prioritize-review` | Modify — add scoping consistency validation |
| `paw-discovery-workflow` | Modify — update stage sequence, remove scoping stage |
| Discovery agent | Modify — update orchestration to skip scoping stage |

## Affected Artifacts

| Artifact | Change |
|----------|--------|
| JourneyMap.md | Remove MVP Options, MVP Critical column, Scoped field |
| Roadmap.md | Add scoping decisions section (depth per journey + rationale) |
| DiscoveryContext.md | Remove scoping-related config if any |

## Stage Sequence (Before → After)

**Before:** Extraction → Mapping → Correlation → Journey Grounding → Journey Scoping → Prioritization

**After:** Extraction → Mapping → Correlation → Journey Grounding → Prioritization (with scoping)

## Resolved Design Questions

### Mode selection timing
**Decision:** Set at init time in DiscoveryContext.md. The existing `Scoping Style` field is replaced by `Prioritization Mode` with values: `interactive`, `guided`, `autonomous`. Consistent with how `review_policy` is configured today.

### Mode × review_policy interaction
**Decision:** All combinations are allowed freely, no restrictions or warnings. The six combinations form a natural spectrum from fully hands-on (`every-stage` + `interactive`) to fully hands-off (`final-only` + `autonomous`).

## DiscoveryContext.md Configuration Changes

**Remove:**
- `Scoping Style` (per-journey / batch / bulk-guidance)

**Add:**
- `Prioritization Mode` (interactive / guided / autonomous)

**Stage Progress table:** Remove the Journey Scoping and Journey Scoping Review rows (12 → 10 stages).
