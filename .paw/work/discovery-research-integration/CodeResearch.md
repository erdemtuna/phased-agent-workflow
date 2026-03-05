---
date: 2026-03-05T20:00:00-05:00
git_commit: 75ed20b
branch: feature/discovery-research-integration
repository: erdemtuna/phased-agent-workflow
topic: "Discovery Research Integration — Init, Extraction, Correlation, Review, Delegation Patterns"
tags: [research, discovery, extraction, correlation, review, delegation, subagent]
status: complete
last_updated: 2026-03-05
---

# Research: Discovery Research Integration

## Research Question

How do the Discovery Init, Extraction, Correlation, Review, and Mapping skills currently work? What are their configuration patterns, artifact structures, execution models, delegation mechanisms, and review criteria — with the goal of understanding where and how to integrate opt-in web research at Extraction and Correlation stages?

## Summary

The Discovery pipeline is a five-stage workflow (Extraction → Mapping → Correlation → Journey Grounding → Prioritization), each with a mandatory review step. All skills are pure markdown prompt files in `skills/paw-discovery-*/SKILL.md` — there is no TypeScript implementation logic for Discovery stages. Configuration lives in `DiscoveryContext.md`, created by the init skill with a fixed template. The Mapping stage provides the only existing subagent delegation pattern: it uses the `task()` tool with `agent_type: "general-purpose"` to spawn a `paw-code-research` subagent. Execution modes (interactive/guided/autonomous) are currently only used by the Prioritization stage. Review skills use a binary PASS/REVISE verdict with checklist-based criteria. The Discovery agent orchestrates everything with inline stage-boundary pause logic.

## Documentation System

- **Framework**: mkdocs with Material theme (`mkdocs.yml:1-60`)
- **Docs Directory**: `docs/` with subdirectories `guide/`, `specification/`, `reference/`
- **Navigation Config**: `mkdocs.yml:61-78` — nav structure with Home, User Guide, Specification, Reference
- **Style Conventions**: Material theme with code copy, search suggest, light/dark toggle
- **Build Command**: `mkdocs build --strict` (validates links); `mkdocs serve` for local preview; requires `source .venv/bin/activate` first
- **Standard Files**: `README.md`, `LICENSE`, `DEVELOPING.md` at repo root; `CHANGELOG.md` not observed

## Verification Commands

- **Test Command**: `npm test` (VS Code extension tests); `npm run test:integration:skills` (fast, no LLM); `npm run test:integration:workflows` (slow, requires Copilot auth) — `package.json:136-137`
- **Lint Command**: `npm run lint` (eslint on `src/`); `npm run lint:agent:all` (prompt linting for agents+skills); `npm run lint:skills` (prompt linting for skills only) — `package.json:131-133`
- **Build Command**: `npm run compile` (tsc) — `package.json:126`
- **Type Check**: `tsc -p ./` (same as compile) — `package.json:126`

## Detailed Findings

### 1. Discovery Init Skill (`paw-discovery-init`)

**File**: `skills/paw-discovery-init/SKILL.md`

#### Configuration Parameters

The init skill accepts five parameters with defaults (`SKILL.md:22-28`):

| Parameter | Default | Values |
|-----------|---------|--------|
| `work_title` | auto-derive | text |
| `review_policy` | `every-stage` | `every-stage`, `final-only` |
| `prioritization_mode` | `guided` | `interactive`, `guided`, `autonomous` |
| `final_review` | `enabled` | `enabled`, `disabled` |
| `final_review_mode` | `society-of-thought` | `single-model`, `multi-model`, `society-of-thought` |

**No research-related parameter exists.** This is the primary extension point for FR-001.

#### Initialization Flow

The flow is a 6-step process (`SKILL.md:52-65`):

1. Derive work title from context (priority: user request → branch name → existing docs → ask)
2. Generate work ID (lowercase, hyphens)
3. Apply defaults for missing parameters
4. Present configuration summary for user confirmation
5. If confirmed → create directory structure and DiscoveryContext.md
6. If adjusted → update values and re-confirm

The confirmation prompt template is at `SKILL.md:56-63`:
```
Discovery: [Work Title]
Work ID: [work-id]
Review Policy: [policy]
Final Review: [mode]

Proceed with this configuration? [Yes / Adjust]
```

**Note**: The confirmation template does NOT currently show Prioritization Mode. Only Review Policy and Final Review are displayed. A new research-enabled field would need to be added to this template.

#### DiscoveryContext.md Template

The full template is at `SKILL.md:88-121`. Key sections:

- **Work Identity** (`SKILL.md:92-95`): Work Title, Work ID, Created date, Workflow Version
- **Configuration** (`SKILL.md:97-101`): Review Policy, Prioritization Mode, Final Review, Final Review Mode
- **Stage Progress** (`SKILL.md:103-115`): Table with all 11 stages (5 activities + 5 reviews + Final Review), each with Status and Artifact columns
- **Input Tracking** (`SKILL.md:117-119`): `last_extraction_inputs` array, `stages_requiring_rerun` array

**The Configuration section is where a `Research` field would be added.** The Stage Progress table would not need new rows — research is embedded within existing Extraction and Correlation stages per the spec.

#### Directory Structure Created

`SKILL.md:80-84`:
```
.paw/discovery/<work-id>/
├── inputs/              # Empty, ready for user documents
└── DiscoveryContext.md  # Initialized with config
```

#### Completion Response

After init (`SKILL.md:126-130`): reports artifact path, confirms `inputs/` created, prompts user to add documents.

### 2. Extraction Skill (`paw-discovery-extraction`)

**File**: `skills/paw-discovery-extraction/SKILL.md`

#### Execution Context

Runs **directly** in the PAW Discovery session (not a subagent) to preserve user interactivity for Q&A refinement (`SKILL.md:8`).

#### Document Conversion

Supports five input formats (`SKILL.md:27-33`):
- Markdown (.md) — native read
- Plain text (.txt) — native read
- Word (.docx) — via `docx` skill or `pandoc`
- PDF (.pdf) — via `pdf` skill or `pypdf`
- Images (.png, .jpg, .jpeg) — direct LLM vision

Conversion process (`SKILL.md:88-108`): scan `inputs/`, determine type by extension, convert to `inputs/converted/`, read converted files.

#### Theme Extraction

Five theme categories (`SKILL.md:121-126`):
- Features, User Needs, Constraints, Vision Statements, Open Questions

Every theme requires source attribution (`SKILL.md:129-132`): source document path, relevant quote/paraphrase, confidence level (high/medium/low).

#### Interactive Q&A Phase

Triggers (`SKILL.md:140-145`): ambiguous themes, apparent conflicts, coverage gaps, low-confidence extractions.

Guidelines (`SKILL.md:148-152`): one question at a time, prefer multiple choice, include recommendation, exit after ~5-10 questions.

Conflict detection (`SKILL.md:155-159`): surface conflict explicitly, quote both sources, ask user to clarify, document resolution.

#### Extraction.md Artifact

Saved to `.paw/discovery/<work-id>/Extraction.md` (`SKILL.md:189`).

YAML frontmatter structure (`SKILL.md:194-211`):
```yaml
date: [ISO timestamp]
work_id: [work-id]
source_documents: [array of {path, type, tokens, converted?}]
theme_count: [N]
conflict_count: [resolved conflicts]
status: complete
```

Sections in the artifact template (`SKILL.md:213-276`):
- Summary
- Features (F1, F2, …)
- User Needs (N1, N2, …)
- Constraints (C1, C2, …)
- Vision Statements (V1, V2, …)
- Open Questions
- Clarification Log (Q&A table)
- Conflict Resolutions

**Research integration point**: After theme extraction and before artifact creation. Research findings would enrich theme descriptions. A new "Research Context" section or per-theme research annotations would be added.

#### Re-extraction Flow

`SKILL.md:289-293`: Compare current `inputs/` with last extraction's source list, full re-extraction (not incremental), preserve user decisions from previous conflict resolutions.

#### State Update on Completion

`SKILL.md:316-318`: Updates `last_extraction_inputs` list and marks Extraction stage as `complete` in DiscoveryContext.md.

### 3. Correlation Skill (`paw-discovery-correlation`)

**File**: `skills/paw-discovery-correlation/SKILL.md`

#### Execution Context

Runs **directly** in the PAW Discovery session (`SKILL.md:8`).

#### Input Artifacts

Reads two artifacts (`SKILL.md:13-14`):
- Extraction.md for themes (features, needs, constraints, vision)
- CapabilityMap.md for existing capabilities

#### Correlation Types

Four types (`SKILL.md:22-47`):
- **Direct Match**: theme maps to existing capability
- **Gap**: theme has no matching capability — requires net-new implementation
- **Combination**: multiple capabilities combine to address theme
- **Partial Match**: theme partially maps to capability

#### Correlation Analysis Process

Four-step process (`SKILL.md:49-77`):
1. **Load Artifacts** (`SKILL.md:51-55`): Build theme list with IDs, build capability list with IDs
2. **Analyze Each Theme** (`SKILL.md:58-63`): Determine which capabilities relate, correlation type, confidence, rationale
3. **Identify Combinations** (`SKILL.md:66-70`): Look for multi-capability opportunities
4. **Surface Relevance Issues** (`SKILL.md:73-77`): Flag domain mismatches, ask user

#### Effort Sanity Check

`SKILL.md:117-132`: Cross-component, data access layer, new UI, external dependencies checks. Adds effort notes to correlation entries.

**Research integration point**: After gap identification and before artifact creation. Research would investigate each gap for feasibility signals (existing libraries, implementation patterns). A new "Feasibility Context" subsection per gap entry.

#### Correlation.md Artifact

Saved to `.paw/discovery/<work-id>/Correlation.md` (`SKILL.md:135`).

YAML frontmatter (`SKILL.md:139-149`):
```yaml
date: [ISO timestamp]
work_id: [work-id]
theme_count: [N]
capability_count: [M]
match_count: [direct matches]
gap_count: [gaps]
combination_count: [combinations]
partial_count: [partial matches]
status: complete
```

Key sections (`SKILL.md:153-217`):
- Summary
- Correlation Matrix (table with Theme, Type, Related Capabilities, Confidence, Notes)
- Direct Matches (detailed per match)
- Gaps (detailed per gap — this is where feasibility context would be added)
- Combinations
- Partial Matches
- Relevance Assessment
- Strategic Insights

### 4. Mapping Skill Delegation Pattern (`paw-discovery-mapping`)

**File**: `skills/paw-discovery-mapping/SKILL.md`

This is the **critical reference pattern** for research subagent delegation.

#### Invocation Pattern

`SKILL.md:79-86`:
```
task(
  agent_type: "general-purpose",
  description: "Code research for Discovery mapping",
  prompt: "[research prompt below]"
)
```

Uses the `task()` tool (subagent spawning mechanism) with `agent_type: "general-purpose"`.

#### Subagent Prompt Structure

`SKILL.md:91-108`: The prompt instructs the subagent to:
1. Load a specific skill (`paw-code-research`)
2. Answer specific research questions
3. Provide context about the Discovery workflow
4. Emphasize focus level (feature-level, not implementation details)
5. Return findings with file:line references

Key template:
```
Load the paw-code-research skill and research the codebase to answer these questions:

1. [Question 1]
2. [Question 2]
...

Context: This research supports a Discovery workflow analyzing [work title].
```

#### Result Handling

`SKILL.md:112-115`: When research completes:
1. Extract capability descriptions from findings
2. Preserve file:line references
3. Map capabilities to relevant extracted themes
4. Identify gaps

**This same pattern would be reused for research subagents**: `task(agent_type: "general-purpose", prompt: "research prompt with web_search instructions")`. The research subagent would use `web_search` tool instead of codebase exploration tools.

### 5. Extraction Review Skill (`paw-discovery-extraction-review`)

**File**: `skills/paw-discovery-extraction-review/SKILL.md`

#### Execution Context

Runs in a **subagent** session, delegated by the PAW Discovery orchestrator (`SKILL.md:8`).

#### Review Criteria Categories

Six categories (`SKILL.md:12-55`):
1. **Document Coverage** (`SKILL.md:14-17`): All files processed, source docs match inputs, token counts reasonable
2. **Theme Quality** (`SKILL.md:20-24`): Clear descriptions, source attribution, confidence levels, no placeholders
3. **Category Completeness** (`SKILL.md:27-31`): Features populated, User Needs populated, at least one category has content, categories reflect actual input
4. **Conflict Handling** (`SKILL.md:34-38`): Conflicts detected, surfaced for resolution, outcomes documented, no unresolved tags
5. **Interactive Refinement** (`SKILL.md:41-44`): Q&A conducted, ambiguous items clarified, open questions captured
6. **Artifact Integrity** (`SKILL.md:47-51`): Valid YAML, status complete, theme count matches, well-structured

**Research extension point**: A 7th category "Research Integration" would be added when research was conducted. Criteria would assess source credibility, integration completeness, and source differentiation (document vs. research).

#### Verdict Structure

`SKILL.md:53-55`: Binary verdict — **PASS** or **REVISE**.

#### Feedback Format

`SKILL.md:57-76`: Structured markdown with numbered issues, each containing Category, Evidence, Required Action; followed by Recommended Actions list.

#### Quality Thresholds

`SKILL.md:78-85`: Table with criterion → threshold mappings (100% document coverage, 100% source attribution, ≥1 category with ≥1 theme, 0 unresolved conflicts, valid YAML).

### 6. Correlation Review Skill (`paw-discovery-correlation-review`)

**File**: `skills/paw-discovery-correlation-review/SKILL.md`

#### Execution Context

Runs in a **subagent** session (`SKILL.md:8`).

#### Review Criteria Categories

Six categories (`SKILL.md:12-46`):
1. **Theme Coverage** (`SKILL.md:14-17`): All themes from Extraction.md in matrix, no orphans, IDs match
2. **Correlation Completeness** (`SKILL.md:20-24`): Every theme has type, valid types, confidence levels, rationale
3. **Evidence Quality** (`SKILL.md:27-31`): Valid capability IDs, clear gap explanations, specific capabilities for combinations, partial coverage scope
4. **Logical Consistency** (`SKILL.md:34-38`): No obvious correlations missed, types match descriptions, confidence justified, strategic insights supported
5. **Relevance Assessment** (`SKILL.md:41-44`): Mismatch addressed, user decision documented, domain alignment reasonable
6. **Artifact Integrity** (`SKILL.md:47-51`): Valid YAML, counts match, status complete, sections populated

**Research extension point**: A 7th category "Research Integration" (mirroring extraction review) would be added when research was conducted. Criteria would assess gap feasibility evidence, source quality, and whether research findings are reflected in correlation entries.

#### Verdict Structure

`SKILL.md:48-50`: Binary verdict — **PASS** or **REVISE** (same pattern as extraction review).

#### Feedback Format

`SKILL.md:52-72`: Same structured markdown format as extraction review.

#### Quality Thresholds

`SKILL.md:74-81`: 100% themes correlated, 100% valid type, 100% confidence assigned, rationale present for all, valid YAML.

### 7. Prioritization Execution Modes (`paw-discovery-prioritize`)

**File**: `skills/paw-discovery-prioritize/SKILL.md`

#### Configuration Read

`SKILL.md:12-16`: Reads `Prioritization Mode` from DiscoveryContext.md Configuration section. Values: `interactive`, `guided`, `autonomous`. Default: `guided` if missing or unrecognized.

This is the **exact pattern** to reuse for reading research configuration. The new feature would read a `Research` field from the same Configuration section.

#### Three Execution Modes

**Interactive Mode** (`SKILL.md:153-155`): User makes explicit scoping and prioritization decisions. Agent presents each journey with depth options for user to choose.

**Guided Mode** (`SKILL.md:158-160`): User provides high-level direction. Agent interprets guidance to determine journey depths and feature scoping.

**Autonomous Mode** (`SKILL.md:163-165`): Agent determines all decisions based on pain severity and feature criticality.

All modes offer one adjustment pass before finalizing.

**This tri-modal pattern is the template for research execution modes**: autonomous (research all themes/gaps), guided (agent recommends, user confirms subset), skip (no research). The spec defines these same three modes for research.

#### Per-Stage Mode Selection Pattern

The prioritization skill reads its mode from DiscoveryContext.md at the start of execution. For research integration, each research-eligible stage (Extraction, Correlation) would prompt for mode selection at execution time rather than reading a pre-configured mode — per FR-002, the mode is selected per-stage at runtime.

### 8. Discovery Agent / Orchestrator (`PAW Discovery.agent.md`)

**File**: `agents/PAW Discovery.agent.md`

#### Initialization

`agent.md:9-10`: On first request, identify work context from environment (branch, `.paw/discovery/` directories). If no DiscoveryContext.md exists, load `paw-discovery-init`. If resuming, derive stage from completed artifacts. Load `paw-discovery-workflow` for reference.

#### Mandatory Transition Table

`agent.md:17-29`: Complete stage-to-stage transition table. All transitions are non-skippable (Skippable = NO) except init → extraction (per user, needs inputs first). Activity → Review transitions are ALWAYS immediate.

#### Stage Boundary Handling

`agent.md:35-53`: Five stages, each with activity + review. Stage boundaries occur between stages:

| Boundary | every-stage | final-only |
|----------|-------------|------------|
| Extraction → Mapping | PAUSE | continue |
| Mapping → Correlation | PAUSE | continue |
| Correlation → Journey Grounding | PAUSE | continue |
| Journey Grounding → Prioritization | PAUSE | continue |
| Prioritization → Done | PAUSE | PAUSE |

Review Policy read from DiscoveryContext.md. Activity → Review = always immediate. Only stage boundaries check policy.

**Research does NOT add new stage boundaries.** Research runs within existing Extraction and Correlation stages before artifact creation, per the spec.

#### Execution Model

`agent.md:99-113`:

**Direct execution** (interactive — runs in the Discovery session):
- paw-discovery-extraction
- paw-discovery-mapping
- paw-discovery-correlation
- paw-discovery-journey-grounding
- paw-discovery-prioritize

**Subagent delegation** (context isolation):
- paw-discovery-extraction-review
- paw-discovery-mapping-review
- paw-discovery-correlation-review
- paw-discovery-journey-grounding-review
- paw-discovery-prioritize-review
- paw-discovery-final-review (SoT-capable)
- paw-code-research (invoked by mapping skill)

#### Cascade Invalidation

`agent.md:74-88`: When extraction re-runs:
1. Mark downstream artifacts as stale (CapabilityMap.md, Correlation.md, Roadmap.md)
2. Update DiscoveryContext.md `stages_requiring_rerun` field
3. Notify user which stages will re-run

**Research artifacts (ExtractionResearch.md, CorrelationResearch.md) would be included in cascade invalidation per FR-012.** When extraction re-runs, ExtractionResearch.md is invalidated. When correlation re-runs, CorrelationResearch.md is invalidated.

#### DiscoveryContext.md State Tracking

`agent.md:186-202`: Updates after each stage with Stage Progress table and Re-invocation State (Last Extraction Inputs, Stages Requiring Rerun).

#### Error Handling

`agent.md:206-223`: Review failure → re-invoke activity with feedback → re-run review → repeat until PASS. Missing inputs → ask user. Conversion failures → report specific error, ask for alternative format.

### 9. Discovery Workflow Skill (`paw-discovery-workflow`)

**File**: `skills/paw-discovery-workflow/SKILL.md`

This is **reference documentation**, not executable logic. Orchestration is owned by the PAW Discovery agent (`SKILL.md:8`).

#### Activities Table

`SKILL.md:44-57`: Lists all 11 skills with capabilities and primary artifacts. Utility skills: `paw-code-research` (invoked by mapping), `paw-sot` (invoked by final-review).

#### Artifact Directory Structure

`SKILL.md:60-73`:
```
.paw/discovery/<work-id>/
├── inputs/
├── DiscoveryContext.md
├── Extraction.md
├── CapabilityMap.md
├── Correlation.md
├── JourneyMap.md
└── Roadmap.md
```

**New research artifacts would be added here**: `ExtractionResearch.md` and `CorrelationResearch.md`.

#### DiscoveryContext.md Template (Workflow Version)

`SKILL.md:77-117`: Slightly different from init skill template — includes Input Documents table with File, Type, Added, Status columns. Configuration section includes Review Policy and Prioritization Mode but not Final Review fields.

**Note**: There is a minor inconsistency between the init skill template (`SKILL.md:88-121` in init) and the workflow skill template (`SKILL.md:77-117` in workflow). The init template includes `Final Review` and `Final Review Mode` fields; the workflow template omits them. The init template uses `last_extraction_inputs` and `stages_requiring_rerun` in Input Tracking; the workflow template uses a more structured Input Documents table plus Re-invocation State section.

#### Execution Model Summary

`SKILL.md:219-222`: Direct execution for 5 activity skills, subagent delegation for 5 review skills + code-research.

#### Re-invocation Support

`SKILL.md:169-184`: Input change detection, cascade invalidation, selective re-run. Trigger commands include natural language phrases.

### 10. Subagent Delegation Patterns

#### Mapping → paw-code-research (Primary Pattern)

**File**: `skills/paw-discovery-mapping/SKILL.md:79-108`

This is the **only existing subagent delegation pattern within Discovery activity skills**.

Mechanism:
```
task(
  agent_type: "general-purpose",
  description: "Code research for Discovery mapping",
  prompt: "<skill loading instruction + research questions + context>"
)
```

The prompt:
1. Instructs subagent to load a specific skill (`paw-code-research`)
2. Provides numbered research questions
3. Gives workflow context
4. Sets focus constraints (feature-level, not implementation details)
5. Requests structured findings with file:line references

Result handling: Extract, preserve references, map to themes, identify gaps.

#### Review Skill Delegation (Agent → Review Subagent)

**File**: `agents/PAW Discovery.agent.md:107-112`

The Discovery agent delegates review skills as subagents. The agent description says these run in subagent sessions, but the delegation mechanism is not explicit in the agent prompt — it relies on the agent's understanding of the execution model and the `task()` tool.

Each review skill header states: "This skill runs in a **subagent** session, delegated by the PAW Discovery orchestrator."

#### paw-code-research Skill (Subagent Target)

**File**: `skills/paw-code-research/SKILL.md`

Execution context: runs in a subagent session (`SKILL.md:8`). Documents implementation details with file:line references. Creates CodeResearch.md at `.paw/work/<work-id>/CodeResearch.md`.

Key characteristics relevant to research pattern:
- Accepts a research topic and questions
- Produces a structured markdown artifact
- Has a strict documentarian mindset (no suggestions)
- Uses YAML frontmatter

**A new research skill (or inline research instructions) would follow a similar pattern**: accept research questions about themes/gaps, use `web_search` tool, produce a structured provenance artifact.

### 11. Discovery Final Review (`paw-discovery-final-review`)

**File**: `skills/paw-discovery-final-review/SKILL.md`

Runs **directly** in Discovery session (not subagent) (`SKILL.md:8`).

Three review modes (`SKILL.md:81-94`):
- **single-model**: Direct review, saves `REVIEW.md`
- **multi-model**: Parallel subagents per model, saves `REVIEW-{MODEL}.md` + `REVIEW-SYNTHESIS.md`
- **society-of-thought**: Loads `paw-sot` skill with `type: discovery-artifacts`

Configuration read from DiscoveryContext.md (`SKILL.md:24-33`): `Final Review Mode`, `Final Review Interactive`, optionally `Final Review Specialists` and `Final Review Interaction Mode`.

Review criteria (`SKILL.md:63-76`): 7 criteria including Extraction Completeness, Source Attribution, Capability Coverage, Correlation Accuracy, Prioritization Rationale, Actionability, Artifact Consistency.

**Research would add an 8th criterion**: Research Integration quality — assessing whether research findings are properly synthesized into main artifacts.

### 12. Discovery Prioritize Review (`paw-discovery-prioritize-review`)

**File**: `skills/paw-discovery-prioritize-review/SKILL.md`

Runs in subagent session (`SKILL.md:8`).

Seven review criteria categories (`SKILL.md:12-63`):
- Multi-Factor Rationale (5 base + 3 journey factors)
- Scoping Consistency (when JourneyMap.md exists)
- Priority Logic
- Item Completeness
- PAW Handoff Quality
- Coverage
- Artifact Integrity

Binary PASS/REVISE verdict (`SKILL.md:65-67`).

### 13. Existing Integration Tests

**File**: `tests/integration/tests/workflows/discovery-workflow.test.ts`

Test exists for Discovery workflow stages. It tests extraction (lines 28-100+), mapping, correlation, journey grounding, and prioritization.

Key test patterns observed (`discovery-workflow.test.ts:28-58`):
- Uses `loadSkill()` to load skill content
- Uses `RuleBasedAnswerer` for interactive Q&A responses
- Uses `createTestContext()` with fixture, skill/agent name, system prompt, answerer
- Seeds input documents and prerequisite artifacts with `writeFile`
- Asserts tool calls and artifact creation

Runtime: ~60-120 seconds per test.

## Code References

- `skills/paw-discovery-init/SKILL.md:22-28` — Init parameter table
- `skills/paw-discovery-init/SKILL.md:52-65` — Initialization flow
- `skills/paw-discovery-init/SKILL.md:56-63` — Confirmation prompt template
- `skills/paw-discovery-init/SKILL.md:88-121` — DiscoveryContext.md template
- `skills/paw-discovery-extraction/SKILL.md:8` — Direct execution context
- `skills/paw-discovery-extraction/SKILL.md:27-33` — Supported input formats
- `skills/paw-discovery-extraction/SKILL.md:121-126` — Theme categories
- `skills/paw-discovery-extraction/SKILL.md:129-132` — Source attribution requirements
- `skills/paw-discovery-extraction/SKILL.md:138-159` — Interactive Q&A phase
- `skills/paw-discovery-extraction/SKILL.md:189` — Artifact save path
- `skills/paw-discovery-extraction/SKILL.md:194-211` — YAML frontmatter structure
- `skills/paw-discovery-extraction/SKILL.md:213-276` — Artifact template sections
- `skills/paw-discovery-extraction/SKILL.md:289-293` — Re-extraction flow
- `skills/paw-discovery-extraction/SKILL.md:316-318` — State update on completion
- `skills/paw-discovery-correlation/SKILL.md:8` — Direct execution context
- `skills/paw-discovery-correlation/SKILL.md:13-14` — Input artifacts
- `skills/paw-discovery-correlation/SKILL.md:22-47` — Correlation types
- `skills/paw-discovery-correlation/SKILL.md:49-77` — Analysis process
- `skills/paw-discovery-correlation/SKILL.md:117-132` — Effort sanity check
- `skills/paw-discovery-correlation/SKILL.md:135` — Artifact save path
- `skills/paw-discovery-correlation/SKILL.md:139-149` — YAML frontmatter
- `skills/paw-discovery-correlation/SKILL.md:153-217` — Artifact template sections
- `skills/paw-discovery-mapping/SKILL.md:79-86` — Task tool invocation pattern
- `skills/paw-discovery-mapping/SKILL.md:91-108` — Subagent prompt structure
- `skills/paw-discovery-mapping/SKILL.md:112-115` — Result handling
- `skills/paw-discovery-extraction-review/SKILL.md:12-55` — Review criteria (6 categories)
- `skills/paw-discovery-extraction-review/SKILL.md:53-55` — Binary verdict
- `skills/paw-discovery-extraction-review/SKILL.md:57-76` — Feedback format
- `skills/paw-discovery-extraction-review/SKILL.md:78-85` — Quality thresholds
- `skills/paw-discovery-correlation-review/SKILL.md:12-46` — Review criteria (6 categories)
- `skills/paw-discovery-correlation-review/SKILL.md:48-50` — Binary verdict
- `skills/paw-discovery-correlation-review/SKILL.md:52-72` — Feedback format
- `skills/paw-discovery-correlation-review/SKILL.md:74-81` — Quality thresholds
- `skills/paw-discovery-prioritize/SKILL.md:12-16` — Configuration read pattern
- `skills/paw-discovery-prioritize/SKILL.md:149-167` — Execution modes
- `skills/paw-discovery-final-review/SKILL.md:24-33` — Final review config read
- `skills/paw-discovery-final-review/SKILL.md:63-76` — Review criteria
- `skills/paw-discovery-final-review/SKILL.md:81-94` — Review mode execution
- `agents/PAW Discovery.agent.md:9-10` — Initialization logic
- `agents/PAW Discovery.agent.md:17-29` — Mandatory transitions table
- `agents/PAW Discovery.agent.md:35-53` — Stage boundary handling
- `agents/PAW Discovery.agent.md:74-88` — Cascade invalidation
- `agents/PAW Discovery.agent.md:99-113` — Execution model (direct vs subagent)
- `agents/PAW Discovery.agent.md:186-202` — State tracking
- `skills/paw-discovery-workflow/SKILL.md:44-57` — Activities table
- `skills/paw-discovery-workflow/SKILL.md:60-73` — Artifact directory structure
- `skills/paw-discovery-workflow/SKILL.md:77-117` — DiscoveryContext.md template
- `skills/paw-discovery-workflow/SKILL.md:169-184` — Re-invocation support
- `skills/paw-code-research/SKILL.md:8` — Subagent execution context
- `skills/paw-code-research/SKILL.md:86-154` — CodeResearch.md template
- `package.json:126-137` — Build, test, lint commands
- `mkdocs.yml:1-78` — Documentation system config
- `tests/integration/tests/workflows/discovery-workflow.test.ts:28-58` — Extraction test pattern

## Architecture Documentation

### Skill Architecture

All Discovery skills are pure markdown files (`SKILL.md`) within `skills/paw-discovery-*/` directories. There is no TypeScript implementation code for any Discovery stage logic — the skills are agent prompts consumed via the `paw_get_skill` tool.

### Execution Model Pattern

Two execution modes exist across Discovery:
1. **Direct execution**: Interactive skills (extraction, mapping, correlation, journey-grounding, prioritize) run in the main Discovery agent session. The agent loads the skill content and follows its instructions.
2. **Subagent delegation**: Review skills and code-research run in isolated subagent sessions via the `task()` tool. This provides context isolation.

### Configuration Pattern

All stage configuration is read from DiscoveryContext.md at the start of skill execution. The pattern is: read markdown file → parse specific field from Configuration section → apply default if missing/unrecognized.

### Review Pattern

All stage reviews follow identical structure:
1. Checklist-based criteria organized in categories
2. Binary PASS/REVISE verdict
3. Structured feedback format with Issues Found + Recommended Actions
4. Quality thresholds table

### Research Artifact Convention

Research provenance artifacts are separate files alongside main artifacts in the work directory. The spec defines `ExtractionResearch.md` and `CorrelationResearch.md` following the same directory convention as existing artifacts.

## Open Questions

1. **DiscoveryContext.md template inconsistency**: The init skill template and workflow skill template differ slightly (init includes Final Review fields; workflow includes Input Documents table). Which is canonical for the research configuration addition?
2. **Research subagent skill**: Should a new `paw-discovery-research` skill be created for the research subagent, or should research instructions be inlined in the extraction/correlation skill prompts? The mapping skill uses a separate skill (`paw-code-research`) for its delegation, suggesting a separate skill is the established pattern.
3. **Integration test coverage**: The existing `discovery-workflow.test.ts` tests the current flow. New tests will be needed for research-enabled extraction and correlation.
