# Code Research: Discovery Scoping-Prioritization Merge

## Affected Files Summary

| File | Action | Key Lines |
|------|--------|-----------|
| `skills/paw-discovery-journey-grounding/SKILL.md` | Modify | 193, 205-211, 218 |
| `skills/paw-discovery-journey-grounding-review/SKILL.md` | Modify | 10, 60, 103 |
| `skills/paw-discovery-journey-scoping/SKILL.md` | Remove | entire file |
| `skills/paw-discovery-prioritize/SKILL.md` | Modify | 83, 96, 105, 122, 126-128, 138-144 |
| `skills/paw-discovery-prioritize-review/SKILL.md` | Modify | 14-54 |
| `skills/paw-discovery-workflow/SKILL.md` | Modify | 54, 140, 222 |
| `skills/paw-discovery-init/SKILL.md` | Modify | 35, 99, 114 |
| `agents/PAW Discovery.agent.md` | Modify | 26-27, 43, 46, 106-107 |

## Detailed Findings

### Journey Grounding — Scoping Fields to Remove

- [skills/paw-discovery-journey-grounding/SKILL.md:193](skills/paw-discovery-journey-grounding/SKILL.md#L193): `- **MVP Depth**: [To be set by scoping]` — journey-level placeholder
- [skills/paw-discovery-journey-grounding/SKILL.md:205-211](skills/paw-discovery-journey-grounding/SKILL.md#L205-L211): `#### MVP Options` section with Full/Partial/Minimal breakdowns and `**Scoped**: [To be set by scoping checkpoint]`
- [skills/paw-discovery-journey-grounding/SKILL.md:218](skills/paw-discovery-journey-grounding/SKILL.md#L218): `| MVP Critical |` column in Feature-to-Journey Mapping table

### Journey Grounding Review — Scoping References

- [skills/paw-discovery-journey-grounding-review/SKILL.md:10](skills/paw-discovery-journey-grounding-review/SKILL.md#L10): Description mentions "before the journey scoping checkpoint proceeds"
- [skills/paw-discovery-journey-grounding-review/SKILL.md:60](skills/paw-discovery-journey-grounding-review/SKILL.md#L60): PASS verdict says "proceed to journey scoping"
- [skills/paw-discovery-journey-grounding-review/SKILL.md:103](skills/paw-discovery-journey-grounding-review/SKILL.md#L103): Completion response mentions "Ready for journey scoping checkpoint"

### Prioritize — JourneyMap Field Consumption

- [skills/paw-discovery-prioritize/SKILL.md:83](skills/paw-discovery-prioritize/SKILL.md#L83): Factor 6 reads "MVP Critical" column from JourneyMap
- [skills/paw-discovery-prioritize/SKILL.md:96](skills/paw-discovery-prioritize/SKILL.md#L96): Factor 8 "MVP Scope" section header
- [skills/paw-discovery-prioritize/SKILL.md:105](skills/paw-discovery-prioritize/SKILL.md#L105): Factor 8 reads "Scoped" field from JourneyMap
- [skills/paw-discovery-prioritize/SKILL.md:122](skills/paw-discovery-prioritize/SKILL.md#L122): Elevation rule for "MVP Critical" features
- [skills/paw-discovery-prioritize/SKILL.md:126-128](skills/paw-discovery-prioritize/SKILL.md#L126-L128): Demotion rules for unlinked/deferred features

### Workflow Skill — Stage Sequence

- [skills/paw-discovery-workflow/SKILL.md:54](skills/paw-discovery-workflow/SKILL.md#L54): Journey Scoping row in activity table
- [skills/paw-discovery-workflow/SKILL.md:140](skills/paw-discovery-workflow/SKILL.md#L140): Journey Scoping in default flow
- [skills/paw-discovery-workflow/SKILL.md:222](skills/paw-discovery-workflow/SKILL.md#L222): Journey Scoping in execution model list

### Init Skill — DiscoveryContext Template

- [skills/paw-discovery-init/SKILL.md:35](skills/paw-discovery-init/SKILL.md#L35): `### Scoping Style Options` section
- [skills/paw-discovery-init/SKILL.md:99](skills/paw-discovery-init/SKILL.md#L99): `- **Scoping Style**: [per-journey|batch|bulk-guidance]`
- [skills/paw-discovery-init/SKILL.md:114](skills/paw-discovery-init/SKILL.md#L114): `| Journey Scoping | pending | - |` in Stage Progress table

### Discovery Agent — Orchestration

- [agents/PAW Discovery.agent.md:26](agents/PAW%20Discovery.agent.md#L26): Transition `journey-grounding-review → journey-scoping`
- [agents/PAW Discovery.agent.md:27](agents/PAW%20Discovery.agent.md#L27): Transition `journey-scoping → prioritize`
- [agents/PAW Discovery.agent.md:43](agents/PAW%20Discovery.agent.md#L43): Stage structure showing scoping as part of Journey Grounding stage
- [agents/PAW Discovery.agent.md:46](agents/PAW%20Discovery.agent.md#L46): Note about scoping being interactive checkpoint
- [agents/PAW Discovery.agent.md:106-107](agents/PAW%20Discovery.agent.md#L106-L107): Journey Scoping in direct execution list

### Journey Scoping Skill — To Be Removed

- `skills/paw-discovery-journey-scoping/SKILL.md` — entire skill file
- `skills/paw-discovery-journey-scoping/` — entire directory
- No `paw-discovery-journey-scoping-review/` directory exists (confirmed)

## Documentation Infrastructure

- Skills are defined in `skills/<skill-name>/SKILL.md`
- Agents are defined in `agents/<name>.agent.md`
- Prompt linting: `./scripts/lint-prompting.sh` for all agent/skill files
- Project docs in `docs/` — specification docs may reference Discovery stages
