---
name: paw-discovery-correlation
description: Correlation activity skill for PAW Discovery workflow. Cross-references extracted themes with mapped capabilities to identify matches, gaps, and combination opportunities.
---

# Discovery Correlation

> **Execution Context**: This skill runs **directly** in the PAW Discovery session, as it may require interactive clarification for relevance mismatches.

Cross-reference extracted themes with mapped codebase capabilities to identify relationships, gaps, and opportunities for strategic leverage.

## Capabilities

- Read Extraction.md for themes (features, needs, constraints, vision)
- Read CapabilityMap.md for existing capabilities
- Generate correlation matrix: Theme × Capability relationships
- Identify direct matches (theme maps to existing capability)
- Identify gaps (theme requires new capability)
- Identify combinations (multiple capabilities combine for larger outcome)
- Optional web research to assess gap feasibility (when research enabled)
- Surface relevance mismatches for user confirmation
- Generate Correlation.md artifact

## Correlation Types

### Direct Match
A theme maps directly to an existing capability.
- The capability fully or substantially addresses the theme
- Implementation would extend/modify existing code
- Lower effort than net-new development

### Gap
A theme has no matching capability in the codebase.
- Requires net-new implementation
- No existing code to extend
- Higher effort, but also higher differentiation

### Combination
Multiple capabilities can combine to address a theme.
- Strategic leverage opportunity
- Individual capabilities exist but aren't connected
- Implementation creates emergent value

### Partial Match
A theme partially maps to a capability.
- Capability addresses some aspects of the theme
- Additional work needed to fully address
- Hybrid of extension and new development

## Correlation Analysis Process

### Step 1: Load Artifacts

Read both Extraction.md and CapabilityMap.md:
- Build theme list with IDs (F1, N1, C1, V1, etc.)
- Build capability list with IDs (CAP-1, CAP-2, etc.)
- Note existing coverage from CapabilityMap.md theme coverage table

### Step 2: Analyze Each Theme

For each theme, determine:
1. Which capabilities relate (if any)
2. Correlation type (match, gap, combination, partial)
3. Confidence level (high/medium/low)
4. Rationale for the correlation

### Step 3: Identify Combinations

Look for combination opportunities:
- Multiple capabilities that together address a complex theme
- Capabilities that could be connected for emergent value
- Themes that benefit from coordinated capability enhancement

### Step 4: Surface Relevance Issues

If extracted themes seem unrelated to codebase domain:
1. Flag the mismatch
2. Ask user: Is this intentional (greenfield) or wrong target repo?
3. Document decision in Correlation.md

## Interactive Clarification

### When to Ask

- Relevance mismatch between themes and codebase domain
- Ambiguous correlation (could be match or gap)
- Combination opportunities that need validation
- Theme that seems important but has no correlation

### Question Format

- One question at a time
- Provide context from both artifacts
- Offer recommendation when you have one

## Open Questions Disposition

Before finalizing correlation, review Open Questions from Extraction.md:

For each open question:
1. Does it affect any correlation? (theme-capability relationship)
2. Does it affect effort assessment? (unknowns = higher risk)
3. Should it be resolved now or tracked for later?

Document disposition in Correlation.md:

```markdown
## Open Questions Status

| Question | Disposition | Impact |
|----------|-------------|--------|
| [Q1 from Extraction] | Deferred to Planning | No impact on correlation |
| [Q2 from Extraction] | Resolved via user | Changed F3 from Gap to Partial |
```

## Effort Sanity Check

For each correlation, do a basic feasibility check:

| Check | Question | If Yes |
|-------|----------|--------|
| **Cross-component** | Does this require changes in multiple codebases/languages? | Flag as "Medium-High" effort |
| **Data access layer** | Does this need new database queries or API calls? | Flag as "Medium" effort minimum |
| **New UI** | Does this need significant new user interface? | Flag as "Medium" effort minimum |
| **External dependencies** | Does this need new services or APIs? | Flag as "High" risk |

Add effort notes to correlation entries:
```markdown
### F2 → Gap
- **Why Gap**: Need Query Store queries
- **Effort Note**: Requires Python LSP changes (cross-component)
```

## Research Phase (Optional)

After initial correlation analysis (match/gap identification and effort checks) and before artifact creation, optionally investigate gap feasibility via web research. This phase only runs when `Research` is set to `autonomous` or `guided` in DiscoveryContext.md. When `disabled`, skip this section entirely — no behavioral change.

### Execution Modes

Read the `Research` field from DiscoveryContext.md Configuration section to determine mode:

- **Autonomous**: Agent researches all identified gaps (and optionally partial matches with unclear feasibility)
- **Guided**: Agent presents gaps with a recommendation of which to research (e.g., "These 3 gaps would benefit most from feasibility research: [gap list]"). User confirms, adjusts the selection, or skips.

### Research Delegation

Delegate research to a subagent following the mapping delegation pattern:

```
task(
  agent_type: "general-purpose",
  description: "Correlation research for Discovery",
  prompt: "<research prompt>"
)
```

The research prompt instructs the subagent to:
- Use `web_search` tool to investigate specified gaps
- Focus on **feasibility**: existing libraries, implementation patterns, ecosystem solutions, integration approaches, effort indicators
- NOT enrich theme understanding (that's extraction research's scope)
- Produce `CorrelationResearch.md` at `.paw/discovery/<work-id>/CorrelationResearch.md`

### CorrelationResearch.md Format

```markdown
---
date: [ISO timestamp]
work_id: [work-id]
gaps_researched: [count]
status: complete
---

# Correlation Research: [Work Title]

## Research Scope
[Which gaps were researched and why]

## Findings

### [Theme ID]: [Theme Name] (Gap)
- **Gap Description**: [What capability is missing]
- **Feasibility Assessment**: [Off-the-shelf / requires custom / hybrid]
- **Existing Solutions**: [Libraries, services, patterns found]
- **Effort Indicators**: [Complexity signals from research]
- **Sources**: [URLs and brief descriptions]

### [Theme ID]: [Theme Name] (Gap)
...

## Gaps Not Researched
[List gaps that were skipped and why — small scope, user excluded, etc.]
```

### Integration into Correlation

After research completes, enrich gap entries in Correlation.md with feasibility context:
- Add `Feasibility` field to gap sections with research-sourced assessment
- Add `Research: [brief finding]` to gap entries in the correlation matrix Notes column
- If research reveals a gap actually has off-the-shelf solutions, note this — but do NOT change the correlation type (the codebase still lacks the capability)

### Graceful Degradation

If `web_search` is unavailable, returns errors, or yields no useful results:
- Note the limitation in CorrelationResearch.md
- Proceed to artifact creation without feasibility context
- Do NOT block correlation on research failures

## Correlation.md Artifact

Save to: `.paw/discovery/<work-id>/Correlation.md`

### Template

```markdown
---
date: [ISO timestamp]
work_id: [work-id]
theme_count: [N from Extraction.md]
capability_count: [M from CapabilityMap.md]
match_count: [direct matches]
gap_count: [gaps]
combination_count: [combinations]
partial_count: [partial matches]
research_conducted: true  # only include this field when research was conducted
status: complete
---

# Correlation: [Work Title]

## Summary

[2-3 sentences describing the correlation landscape and key findings]

## Correlation Matrix

| Theme | Type | Related Capabilities | Confidence | Notes |
|-------|------|---------------------|------------|-------|
| F1 | Match | CAP-1 | High | Extends existing auth |
| F2 | Gap | - | High | Net-new capability |
| F3 | Combination | CAP-2, CAP-3 | Medium | Requires integration |
| N1 | Partial | CAP-1 | Medium | Covers 60% |
| C1 | Match | CAP-4 | High | Already enforced |

## Direct Matches

### F1 → CAP-1: [Correlation Name]
- **Theme**: [F1 description from Extraction.md]
- **Capability**: [CAP-1 description from CapabilityMap.md]
- **Rationale**: [Why this is a match]
- **Implementation Impact**: [What extending this means]

...

## Gaps

### F2: [Theme Name]
- **Theme**: [F2 description]
- **Why Gap**: [Why no capability matches]
- **New Work Required**: [What needs to be built]
- **Feasibility**: [If research conducted: off-the-shelf / requires custom / hybrid — with source]

...

## Combinations

### F3 → CAP-2 + CAP-3: [Combination Name]
- **Theme**: [F3 description]
- **Capabilities**: [CAP-2 and CAP-3 descriptions]
- **Synergy**: [How they combine for greater value]
- **Integration Required**: [What connects them]

...

## Partial Matches

### N1 ↔ CAP-1: [Partial Match Name]
- **Theme**: [N1 description]
- **Capability**: [CAP-1 description]
- **Coverage**: [What's covered, what's not]
- **Gap to Fill**: [Additional work needed]

...

## Relevance Assessment

[Any notes about domain alignment between themes and codebase]
[User decisions if relevance mismatch was surfaced]

## Strategic Insights

- [Key insight 1 about the correlation landscape]
- [Key insight 2 about leverage opportunities]
- [Key insight 3 about gap patterns]
```

## Edge Cases

### Relevance Mismatch

If themes describe capabilities far outside codebase domain:
1. Surface explicitly: "Extracted themes appear unrelated to this codebase's domain"
2. Ask user: "Is this intentional (greenfield development) or should we target a different repo?"
3. Document decision in Relevance Assessment section

### No Matches

If all themes are gaps (no matching capabilities):
1. This is valid for greenfield or new domain
2. Document that correlations are primarily gaps
3. Note this affects prioritization (all new work)

## Quality Checklist

- [ ] All themes from Extraction.md are in correlation matrix
- [ ] Correlation types are assigned (match/gap/combination/partial)
- [ ] Confidence levels are assigned
- [ ] Rationale provided for each correlation
- [ ] Relevance issues surfaced (if any)
- [ ] YAML frontmatter counts are accurate
- [ ] Strategic insights section provides value for prioritization
- [ ] If research enabled: CorrelationResearch.md created and feasibility context added to gap entries

## Completion Response

Report to PAW Discovery agent:
- Artifact path: `.paw/discovery/<work-id>/Correlation.md`
- Correlation summary (X matches, Y gaps, Z combinations)
- Relevance assessment (aligned / mismatch resolved)
- Research conducted: yes/no (if yes: `.paw/discovery/<work-id>/CorrelationResearch.md`)
- Ready for correlation review
