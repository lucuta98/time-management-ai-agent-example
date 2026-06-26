# Plan Mode Agent Rules

## Automatic Prompt File Generation

### Rule 1: Save AI Prompt on Every Plan Mode Execution

**Trigger:** Whenever Plan mode is used to generate any documentation, architecture, module, or requirements output

**Actions:**
1. Automatically create a prompt file in `docs/ai-prompts/` immediately after the output files are generated
2. The filename must follow the pattern: `{number}_{subject}_prompt.md`
   - `{number}` — next available sequential number (inspect `docs/ai-prompts/` to determine it)
   - `{subject}` — single lowercase word describing the output (e.g. `requirements`, `architecture`, `modules`, `deployment`)
3. For requirements generation only, also create an identical copy in `docs/requirements/`

**Purpose:**
- Maintain a complete audit trail of every AI-assisted generation task
- Ensure all planning artifacts are traceable back to the prompt that produced them
- Create a consistent, browsable history of how the project was built

**Implementation Details:**
- The prompt file must contain the **exact original prompt** as written by the user (verbatim, inside a code block)
- File naming: `{number}_{subject}_prompt.md` — one word for subject, no extra words (e.g. `03_modules_prompt.md` not `03_module_docs_prompt.md`)
- Sequential number must be determined by inspecting existing files in `docs/ai-prompts/` at the time of writing
- This rule applies to **all** generation tasks executed in Plan mode, not just requirements

**Prompt File Structure (required sections):**

```
# Original Prompt for <Subject>

## Prompt Used
<verbatim prompt inside a code block>

## Context
- Date, Mode, Input Files used

## Output Files Generated
Numbered list of every file produced

## Key Instructions
Checklist of constraints that were applied

## Current Location
Where the output files live

## Notes
Any relevant observations about the generation
```

**Examples:**
| Task | Prompt File Created |
|---|---|
| Generate requirements | `docs/ai-prompts/01_requirements_prompt.md` |
| Generate architecture | `docs/ai-prompts/02_architecture_prompt.md` |
| Generate module docs | `docs/ai-prompts/03_modules_prompt.md` |
| Generate deployment docs | `docs/ai-prompts/04_deployment_prompt.md` |

---

### Rule 2: Requirements Prompt — Dual Copy

**Trigger:** Specifically when Plan mode generates requirements documentation

**Actions:**
1. Apply Rule 1 (save to `docs/ai-prompts/`)
2. Additionally save an identical copy to `docs/requirements/` with the same filename

**Purpose:**
- Keep requirements documentation self-contained — the prompt that produced the requirements lives alongside them

---

### Rule Application Scope

Both rules trigger automatically for:
- Initial generation of any document type (requirements, architecture, modules, deployment, security, etc.)
- Revisions or updates to existing documents
- Any Plan mode task that produces one or more output `.md` files

### File Structure Requirements

Each prompt file must include:
- The verbatim original prompt (inside a fenced code block)
- Context: date, mode, input files
- List of all output files generated
- Key constraints or instructions applied
- Location of output files
- Any relevant notes