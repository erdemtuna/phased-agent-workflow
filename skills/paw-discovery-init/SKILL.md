---
name: paw-discovery-init
description: Bootstrap skill for PAW Discovery workflow initialization. Creates DiscoveryContext.md, directory structure, and inputs folder.
---

# Discovery Initialization

> **Execution Context**: This skill runs **directly** in the PAW Discovery session (not a subagent), as bootstrap requires deriving context and confirming configuration.

Bootstrap skill that initializes the Discovery workflow directory structure. This runs **before** the workflow skill is loaded—DiscoveryContext.md must exist for the workflow to function.

## Capabilities

- Derive Work Title from user request, branch name, or folder context
- Generate Work ID from Work Title (normalized, unique)
- Create `.paw/discovery/<work-id>/` directory structure
- Generate DiscoveryContext.md with configuration fields
- Create `inputs/` folder for user documents

## Input Parameters

| Parameter | Required | Default | Values |
|-----------|----------|---------|--------|
| `work_title` | No | auto-derive | text |
| `review_policy` | No | `every-stage` | `every-stage`, `final-only` |
| `prioritization_mode` | No | `guided` | `interactive`, `guided`, `autonomous` |
| `research` | No | `disabled` | `disabled`, `autonomous`, `guided` |
| `final_review` | No | `enabled` | `enabled`, `disabled` |
| `final_review_mode` | No | `society-of-thought` | `single-model`, `multi-model`, `society-of-thought` |

### Review Policy Options

- **`every-stage`**: Pause at each stage boundary (after Extraction, Mapping, Correlation, Journey Grounding, Prioritization) for user review
- **`final-only`**: Run all stages autonomously, only pause when Roadmap.md is complete

### Prioritization Mode Options

- **`interactive`**: User walks through journeys, picks depths, validates resulting prioritized roadmap in one pass
- **`guided`** (default): User provides high-level direction, agent applies scoping and generates roadmap
- **`autonomous`**: Agent auto-scopes based on pain severity and correlation data, generates roadmap without user input

### Research Options

- **`disabled`** (default): No web research. Extraction and Correlation stages behave as standard.
- **`autonomous`**: Web research enabled. Agent identifies research-worthy themes/gaps and researches them without user input at each eligible stage.
- **`guided`**: Web research enabled. Agent proposes which themes/gaps to research; user confirms or adjusts the selection at each eligible stage.

## Work Title Derivation

When `work_title` is not provided, derive from (priority order):

1. **User's request**: Extract topic from what user asked (e.g., "analyze my product roadmap" → "Product Roadmap Analysis")
2. **Branch name**: If on discovery-related branch (e.g., `discovery/q1-planning` → "Q1 Planning")
3. **Existing documents**: If documents already exist, derive from folder or file names
4. **Ask as last resort**: Only if no context available

## Initialization Flow

1. **Derive work title** from context (see priority order above)
2. **Generate work ID** from title (lowercase, hyphens)
3. **Apply defaults** for missing parameters
4. **Present configuration summary**:
   ```
   Discovery: [Work Title]
   Work ID: [work-id]
   Review Policy: [policy]
   Research: [disabled/autonomous/guided]
   Final Review: [mode]
   
   Proceed with this configuration? [Yes / Adjust]
   ```
5. **If user confirms**: Create directory structure and DiscoveryContext.md
6. **If user adjusts**: Update values and re-confirm

## Desired End States

### Work Title
- A 2-5 word human-readable name exists
- Derived from context, not asked as first question
- Capitalized appropriately (e.g., `Q1 Product Roadmap`)

### Work ID
- Unique within `.paw/discovery/`
- Format: lowercase letters, numbers, hyphens only; 1-100 chars (no path traversal characters)
- If conflict: append `-2`, `-3`, etc.

### Directory Structure
```
.paw/discovery/<work-id>/
├── inputs/              # Empty, ready for user documents
└── DiscoveryContext.md  # Initialized with config
```

## DiscoveryContext.md Template

```markdown
# Discovery Context

## Work Identity
- **Work Title**: [Title]
- **Work ID**: `[work-id]`
- **Created**: [ISO date]
- **Workflow Version**: 2.0

## Configuration
- **Review Policy**: [every-stage|final-only]
- **Prioritization Mode**: [interactive|guided|autonomous]
- **Research**: [disabled|autonomous|guided]
- **Final Review**: [enabled|disabled]
- **Final Review Mode**: [single-model|multi-model|society-of-thought]

## Stage Progress
| Stage | Status | Artifact |
|-------|--------|----------|
| Extraction | pending | - |
| Extraction Review | pending | - |
| Mapping | pending | - |
| Mapping Review | pending | - |
| Correlation | pending | - |
| Correlation Review | pending | - |
| Journey Grounding | pending | - |
| Journey Grounding Review | pending | - |
| Prioritization | pending | - |
| Prioritization Review | pending | - |
| Final Review | pending | - |

## Input Tracking
- **last_extraction_inputs**: []
- **stages_requiring_rerun**: []
```

## Completion Response

After initialization:
- Report artifact path: `.paw/discovery/<work-id>/DiscoveryContext.md`
- Confirm `inputs/` folder created
- Prompt user to add documents to `inputs/` folder
- Ready for extraction stage when documents added
