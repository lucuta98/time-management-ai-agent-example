# Original Prompt for Data Models

## Prompt Used

```
Generate documentation for all data models using parallel sub‑agents.
For each data model, create a separate file inside docs/data-models.
Use one parallel sub‑agent per data model.
Each sub‑agent must generate exactly one file.
File format: <model_name>.md.

Each file must include:

model purpose

fields and types

constraints

relationships

usage examples

notes for future extensions

Data models to generate:

Task

Event

UserProfile

Preferences

ScheduleBlock

ContextSnapshot

AnalyticsRecord

Ensure all files follow the same structure.
Ensure all sub‑agents run in parallel.
Ensure all files are saved in docs/data-models.
```

## Context

- Date: 2026-06-19
- Mode: Plan
- Input Files:
  - `docs/architecture/02_architecture_modules.md`
  - `docs/architecture/02_architecture_ai_ml.md`
  - `docs/architecture/02_architecture_components.md`
  - `docs/architecture/02_architecture_dataflows.md`

## Output Files Generated

1. `docs/data-models/Task.md`
2. `docs/data-models/Event.md`
3. `docs/data-models/UserProfile.md`
4. `docs/data-models/Preferences.md`
5. `docs/data-models/ScheduleBlock.md`
6. `docs/data-models/ContextSnapshot.md`
7. `docs/data-models/AnalyticsRecord.md`

## Key Instructions

- One file per data model — 7 files total
- Each file follows a fixed 6-section structure: Purpose / Fields and Types / Constraints / Relationships / Usage Examples / Notes for Future Extensions
- All content sourced exclusively from the existing architecture documents
- All 7 sub-agents spawned in a single parallel batch
- Each file saved to `docs/data-models/`

## Current Location

All output files live in `docs/data-models/`.

## Notes

- `AnalyticsRecord` is a discriminated union model (3 sub-types: TimeEntry, ProductivityMetric, Goal) driven by a `recordType` enum field
- `ContextSnapshot` has a TTL constraint (default 300 s) — stale snapshots must be regenerated before scheduling or recommendation decisions
- `ScheduleBlock` carries a `constraintsSatisfied` audit object recording which hard/soft scheduling constraints were met at creation time
- `UserProfile` and `Preferences` are 1:1; all other models are 1:N per user
- Field types are expressed in plain English (not a specific language) to remain implementation-agnostic
