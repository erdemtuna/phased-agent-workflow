---
name: paw-discovery-extraction
description: Extraction activity skill for PAW Discovery workflow. Processes input documents (docx/PDF conversion), extracts structured themes, and supports interactive Q&A for refinement.
---

# Discovery Extraction

> **Execution Context**: This skill runs **directly** in the PAW Discovery session (not a subagent), preserving user interactivity for Q&A refinement.

Process input documents and extract structured themes with source attribution. Supports heterogeneous inputs including Word documents and text-based PDFs.

## Capabilities

- Convert .docx files to markdown (mammoth)
- Convert text-based PDF files to text (pdf-parse)
- Read markdown and plain text files natively
- Extract structured themes: features, user needs, constraints, vision statements
- Optional web research to enrich themes with external context (when research enabled)
- Interactive Q&A to validate and refine understanding
- Detect and surface document conflicts
- Warn about scale limits (token overflow)
- Generate Extraction.md with YAML frontmatter

## Document Conversion

### Supported Formats

| Format | Extension | Approach | Output |
|--------|-----------|----------|--------|
| Markdown | .md | Native read via `view` tool | - |
| Plain text | .txt | Native read via `view` tool | - |
| Word | .docx | Load `docx` skill or use `pandoc` | `.md` |
| PDF (text) | .pdf | Load `pdf` skill or use `pypdf` | `.md` |
| Images | .png, .jpg, .jpeg | Pass directly to LLM vision | - |

### Image Handling

PNG and JPEG images are passed directly to the LLM for visual analysis:
- Product mockups, wireframes, diagrams
- Screenshots of requirements or specifications
- Architecture diagrams or flowcharts

The LLM extracts themes from images using vision capabilities. Include image descriptions in theme source attribution.

### Security Note

When processing filenames from `inputs/`:
- Validate filenames contain only safe characters (alphanumeric, hyphens, underscores, dots)
- Reject or sanitize filenames with shell metacharacters (`$`, `;`, `|`, etc.)
- Use quoted paths in shell commands to prevent injection

### Conversion Approaches

**Option 1: Use installed document skills (Recommended)**

If the user has the `docx` and `pdf` skills installed (from Anthropic skills package), load and invoke them:

- For `.docx`: Load `docx` skill, use its `pandoc` command or unpack/read approach
- For `.pdf`: Load `pdf` skill, use its `pypdf`/`pdfplumber` extraction patterns

**Option 2: Direct CLI commands**

If skills aren't available, use standard tools:

**Word documents (.docx)**:
```bash
mkdir -p "inputs/converted"
pandoc "inputs/document.docx" -o "inputs/converted/document.md"
cat "inputs/converted/document.md"
```

**PDF documents (.pdf)**:
```bash
mkdir -p "inputs/converted"
```
```python
from pypdf import PdfReader

reader = PdfReader("inputs/document.pdf")
text = ""
for page in reader.pages:
    text += page.extract_text() + "\n\n"

with open("inputs/converted/document.md", "w") as f:
    f.write(text)
```

### Conversion Process

1. Scan `inputs/` folder for all files
2. For each file, determine type by extension
3. Convert non-native formats and **save to `inputs/converted/`**:
   - `document.docx` → `inputs/converted/document.md`
   - `document.pdf` → `inputs/converted/document.md`
4. Read converted files for theme extraction
5. Pass images directly to LLM for visual analysis

### Converted Output Directory

Create `inputs/converted/` to store text-readable versions:
```
inputs/
├── roadmap.docx          # Original
├── requirements.pdf      # Original
├── mockup.png            # Image (no conversion needed)
└── converted/
    ├── roadmap.md        # Converted from docx
    └── requirements.md   # Converted from pdf
```

This enables:
- Inspection of conversion quality
- Reuse without re-conversion
- Audit trail of what the LLM processed

**Note**: Image-based PDFs are not supported. If pypdf/pdfplumber returns minimal text, warn user that the PDF may be image-based and suggest OCR or alternative format.

## Theme Extraction

### Theme Categories

Extract and categorize content into:

- **Features**: Proposed capabilities, functionality, or behaviors
- **User Needs**: Problems to solve, jobs-to-be-done, pain points
- **Constraints**: Limitations, requirements, boundaries
- **Vision Statements**: Long-term goals, strategic direction
- **Open Questions**: Unresolved items requiring clarification

### Source Attribution

Every extracted theme MUST include:
- Source document path (e.g., `inputs/roadmap.md`)
- Relevant quote or paraphrase
- Theme confidence (high/medium/low based on clarity in source)

## Research Phase (Optional)

After initial theme identification and before Q&A refinement, optionally enrich themes with external context via web research. This phase only runs when `Research` is set to `autonomous` or `guided` in DiscoveryContext.md. When `disabled`, skip this section entirely — no behavioral change.

### Execution Modes

Read the `Research` field from DiscoveryContext.md Configuration section to determine mode:

- **Autonomous**: Agent identifies the most ambiguous or novel themes (those with low confidence, unclear terminology, or implicit assumptions), constructs research questions, and delegates without user input
- **Guided**: Agent presents identified themes with a recommendation of which to research and why (e.g., "These 3 of 8 themes reference standards or technologies that would benefit from external context"). User confirms, adjusts the selection, or skips. For large theme sets (10+), group by category and recommend a focused subset rather than listing all individually.

### Research Delegation

Delegate research to a subagent following the mapping delegation pattern:

```
task(
  agent_type: "general-purpose",
  description: "Extraction research for Discovery",
  prompt: "<research prompt>"
)
```

The research prompt instructs the subagent to:
- Use `web_search` tool to investigate specified themes
- Focus on **enrichment**: industry standards, common approaches, terminology clarification, relevant APIs/libraries, prior art
- NOT consider the codebase (purely external context)
- Produce `ExtractionResearch.md` at `.paw/discovery/<work-id>/ExtractionResearch.md`

### ExtractionResearch.md Format

```markdown
---
date: [ISO timestamp]
work_id: [work-id]
themes_researched: [count]
status: complete
---

# Extraction Research: [Work Title]

## Research Scope
[Which themes were researched and why]

## Findings

### [Theme ID]: [Theme Name]
- **Research Context**: [What was investigated]
- **Findings**: [Standards, approaches, terminology, prior art discovered]
- **Sources**: [URLs and brief descriptions]
- **Relevance**: [How this enriches the theme understanding]

### [Theme ID]: [Theme Name]
...

## Themes Not Researched
[List themes that were skipped and why — already well-defined, out of guided selection, etc.]
```

### Integration into Extraction

After research completes, enrich theme descriptions with research findings:
- Add research-sourced context to theme descriptions, clearly attributing the source (e.g., "Per [source]: ...")
- Mark research-enriched themes with `Research: [brief finding]` alongside existing Source attribution
- If research contradicts a document source, surface the discrepancy for resolution during Q&A

### Graceful Degradation

If `web_search` is unavailable, returns errors, or yields no useful results:
- Note the limitation in ExtractionResearch.md
- Proceed to Q&A without research enrichment
- Do NOT block extraction on research failures

## Interactive Q&A Phase

### When to Ask

- Ambiguous themes that could be interpreted multiple ways
- Apparent conflicts between documents
- Gaps in coverage (expected areas with no themes)
- Low-confidence extractions

### Question Guidelines

- One question at a time
- Prefer multiple choice when options are enumerable
- Include recommendation when you have one
- Exit after ~5-10 questions or when user signals done

### Conflict Detection

When documents contain contradictory information:
1. Surface the conflict explicitly
2. Quote both sources
3. Ask user to clarify which interpretation is correct
4. Document resolution in Extraction.md

## Edge Case Handling

### Scale Limits

When total input content approaches context limits:
1. Calculate approximate token count across all inputs
2. If exceeding practical limits (~50k tokens), warn user
3. Suggest prioritizing most important documents
4. Offer to proceed with subset

### Contradictory Documents

When detecting conflicting information:
1. Extract both versions as separate themes
2. Mark with `[CONFLICT]` tag
3. Surface in Q&A phase for resolution
4. Document final resolution with rationale

### Empty or Minimal Inputs

If `inputs/` folder is empty or contains only trivial content:
1. Report to user
2. Ask for additional documents or clarification
3. Do not generate Extraction.md until meaningful input exists

## Extraction.md Artifact

Save to: `.paw/discovery/<work-id>/Extraction.md`

### Template

```markdown
---
date: [ISO timestamp]
work_id: [work-id]
source_documents:
  - path: inputs/doc1.md
    type: markdown
    tokens: [approximate]
  - path: inputs/doc2.docx
    type: docx
    converted: inputs/converted/doc2.md
    tokens: [approximate]
  - path: inputs/mockup.png
    type: image
    tokens: [approximate vision tokens]
theme_count: [N]
conflict_count: [resolved conflicts]
research_conducted: true  # only include this field when research was conducted
status: complete
---

# Extraction: [Work Title]

## Summary

[2-3 sentence overview of what was extracted and key themes]

## Features

### F1: [Feature Name]
- **Description**: [What this feature does]
- **Source**: `inputs/roadmap.md` - "[relevant quote]"
- **Confidence**: high

### F2: [Feature Name]
...

## User Needs

### N1: [Need Name]
- **Description**: [The problem or job-to-be-done]
- **Source**: `inputs/research.pdf` - "[relevant quote]"
- **Confidence**: medium

...

## Constraints

### C1: [Constraint Name]
- **Description**: [The limitation or boundary]
- **Source**: `inputs/requirements.docx` - "[relevant quote]"
- **Confidence**: high

...

## Vision Statements

### V1: [Vision Statement]
- **Description**: [Long-term goal or direction]
- **Source**: `inputs/strategy.md` - "[relevant quote]"
- **Confidence**: high

...

## Open Questions

- [Question 1] - surfaced during extraction, not resolved in Q&A
- [Question 2]

## Clarification Log

Track all Q&A interactions during extraction for auditability:

| # | Question Asked | User Answer | Impact on Extraction |
|---|----------------|-------------|---------------------|
| 1 | [Question] | [Answer] | Added constraint C3 |
| 2 | [Question] | [Answer] | Scoped F5 to exclude X |

## Conflict Resolutions

### CR1: [Conflict Description]
- **Source A**: `inputs/doc1.md` - "[quote]"
- **Source B**: `inputs/doc2.md` - "[quote]"
- **Resolution**: [User decision and rationale]
```

## Execution

### Desired End States

- All input documents processed (converted if needed)
- Themes extracted with source attribution
- Research conducted if enabled (ExtractionResearch.md created, findings synthesized into themes)
- Conflicts detected and resolved via Q&A
- Scale warnings surfaced if applicable
- Extraction.md saved with valid YAML frontmatter

### Re-extraction Flow

When re-invoked after inputs change:
1. Compare current `inputs/` with last extraction's source list
2. Identify new/modified/removed files
3. Full re-extraction (not incremental merge)
4. Invalidate ExtractionResearch.md if it exists (research re-runs with new themes)
5. Preserve user decisions from previous conflict resolutions if applicable

## Quality Checklist

- [ ] All input documents processed
- [ ] Themes have source attribution with document path
- [ ] Conflicts detected and surfaced for resolution
- [ ] Q&A phase completed or user signaled done
- [ ] Scale warnings surfaced if inputs are large
- [ ] Extraction.md has valid YAML frontmatter
- [ ] Theme categories are populated (features, needs, constraints, vision)
- [ ] DiscoveryContext.md updated with current inputs list
- [ ] If research enabled: ExtractionResearch.md created and findings synthesized into themes

## Completion Response

Report to PAW Discovery agent:
- Artifact path: `.paw/discovery/<work-id>/Extraction.md`
- Theme counts by category
- Conflict resolutions (if any)
- Open questions (if any)
- Research conducted: yes/no (if yes: `.paw/discovery/<work-id>/ExtractionResearch.md`)
- Ready for extraction review

**State Update**: Update DiscoveryContext.md with:
- `last_extraction_inputs`: List of files processed (enables re-invocation detection)
- Stage Progress table: Mark Extraction as `complete`
