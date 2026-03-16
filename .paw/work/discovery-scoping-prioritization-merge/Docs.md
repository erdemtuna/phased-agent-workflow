# Docs: Discovery Scoping-Prioritization Merge

## Summary

Merged the standalone journey-scoping stage into the prioritization/roadmap stage in the PAW Discovery workflow. JourneyMap is now a pure vision artifact (no delivery decisions); Roadmap is the single decision point for scoping and prioritization with three configurable execution modes.

## Changes

### Removed
- `skills/paw-discovery-journey-scoping/` — entire skill directory deleted

### Modified Skills
- **`paw-discovery-journey-grounding`**: Removed MVP Options, MVP Depth, Scoped, and MVP Critical fields from JourneyMap.md template
- **`paw-discovery-journey-grounding-review`**: Updated references from "journey scoping" to "prioritization"
- **`paw-discovery-prioritize`**: Added Prioritization Mode configuration, three execution modes (interactive/guided/autonomous), agent-derived scoping logic, and scoping decisions section in Roadmap.md template
- **`paw-discovery-prioritize-review`**: Added scoping consistency validation criteria
- **`paw-discovery-workflow`**: Removed journey-scoping from activity table, default flow, and execution model
- **`paw-discovery-init`**: Replaced `Scoping Style` with `Prioritization Mode` configuration, removed Journey Scoping from Stage Progress table

### Modified Agents
- **`PAW Discovery.agent.md`**: Removed journey-scoping transitions, updated stage structure, removed from direct execution list

### Updated Documentation
- **`docs/guide/discovery-workflow.md`**: Removed Journey Scoping checkpoint reference, updated JourneyMap description

## Configuration Change

DiscoveryContext.md configuration field replaced:

| Before | After |
|--------|-------|
| `Scoping Style: per-journey\|batch\|bulk-guidance` | `Prioritization Mode: interactive\|guided\|autonomous` |

Stage Progress table reduced from 12 to 10 rows.

## Prioritization Modes

| Mode | User Involvement | Behavior |
|------|-----------------|----------|
| Interactive | High | Walk through journeys, pick depths, validate roadmap |
| Guided | Medium | Provide high-level direction, agent applies |
| Autonomous | Low | Agent auto-scopes and ranks, user reviews output |

## Verification

```bash
./scripts/lint-prompting.sh skills/paw-discovery-journey-grounding/SKILL.md
./scripts/lint-prompting.sh skills/paw-discovery-journey-grounding-review/SKILL.md
./scripts/lint-prompting.sh skills/paw-discovery-prioritize/SKILL.md
./scripts/lint-prompting.sh skills/paw-discovery-prioritize-review/SKILL.md
./scripts/lint-prompting.sh skills/paw-discovery-workflow/SKILL.md
./scripts/lint-prompting.sh skills/paw-discovery-init/SKILL.md
./scripts/lint-prompting.sh "agents/PAW Discovery.agent.md"
npm run lint
npm run lint:agent:all
source .venv/bin/activate && mkdocs build --strict
```
