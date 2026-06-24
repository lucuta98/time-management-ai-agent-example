# Plan Mode Agent Rules

## Automatic Prompt File Generation

### Rule: Requirements Prompt File Creation

**Trigger:** Whenever Plan mode is used to generate a requirements .md file

**Actions:**
1. Automatically create a prompt file in `docs/ai-prompts/` named `01_requirements_prompt.md` (or matching the subject of the requirements file)
2. Automatically create the same prompt file in `docs/requirements/` with the same name

**Purpose:** 
- Maintain consistency between AI prompts and requirements documentation
- Ensure all planning artifacts are properly documented and accessible
- Create a traceable link between the prompts used and the requirements generated

**Implementation Details:**
- The prompt file should contain the exact prompt used to generate the requirements
- File naming should follow the pattern: `{number}_{subject}_prompt.md`
- Both copies (in `docs/ai-prompts/` and `docs/requirements/`) must be identical
- This rule applies to all requirements generation tasks in Plan mode

**Example:**
- When generating requirements for "Time Management System"
- Create: `docs/ai-prompts/01_requirements_prompt.md`
- Create: `docs/requirements/01_requirements_prompt.md`
- Both files contain the same prompt content used to generate the requirements

### Rule Application Scope

This rule triggers automatically for:
- Initial requirements generation
- Requirements updates or revisions
- Any planning file generation that produces requirements documentation

### File Structure Requirements

Each prompt file must include:
- Clear description of what was requested
- Context provided to the AI
- Any specific constraints or guidelines
- Timestamp of when the prompt was used
- Reference to the generated requirements file(s)