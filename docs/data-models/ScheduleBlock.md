# ScheduleBlock

## 1. Model Purpose

ScheduleBlock represents a reserved time slot assigned to a specific Task by the SchedulingEngine. It is the concrete output of schedule optimisation — a time window on the calendar during which the user is expected to work on a task. ScheduleBlock bridges the Task domain and the Calendar domain: it carries a predicted energy score for the slot and an audit record of which scheduling constraints were satisfied.

## 2. Fields and Types

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique identifier for the block |
| `userId` | UUID | FK to UserProfile |
| `taskId` | UUID | FK to Task; the task assigned to this time slot |
| `eventId` | UUID | Optional FK to Event; set if a calendar Event was created for this block |
| `startTime` | DateTime | Block start time (UTC) |
| `endTime` | DateTime | Block end time (UTC) |
| `durationMinutes` | Integer | `endTime − startTime` in minutes |
| `energyScore` | Float | Predicted user energy level during this slot (0–100) |
| `status` | Enum | `Proposed`, `Accepted`, `Rejected`, `Completed` |
| `source` | Enum | How the block was created: `AIGenerated` or `UserManual` |
| `constraintsSatisfied` | Object | Audit record of constraint evaluation at creation time (see below) |
| `createdAt` | DateTime | Creation timestamp (UTC) |
| `updatedAt` | DateTime | Last modification timestamp (UTC) |

### `constraintsSatisfied` Structure

```json
{
  "withinWorkingHours": true,
  "noConflict": true,
  "beforeDeadline": true,
  "preferredTime": false,
  "energyLevelAdequate": true
}
```

Hard constraints (`withinWorkingHours`, `noConflict`, `beforeDeadline`, dependency order) must all be `true` for a block to be proposed. Soft constraints (`preferredTime`, `energyLevelAdequate`) are scored but may be `false`.

### Scheduling Algorithm

The SchedulingEngine uses a Constraint Satisfaction Problem (CSP) approach enhanced with ML-based slot scoring:

1. Filter feasible slots (hard constraints)
2. Score each slot using the energy prediction model
3. Solve CSP optimisation
4. Emit ScheduleBlocks with status `Proposed`

## 3. Constraints

- `startTime` must be before `endTime`
- `durationMinutes` must equal `(endTime − startTime)` in minutes
- `taskId` must reference an existing non-`Completed` Task
- No two `Accepted` ScheduleBlocks for the same user may overlap in time
- `energyScore`: Float 0.0–100.0
- `status` defaults to `Proposed` for AI-generated blocks

## 4. Relationships

- **belongs to** UserProfile (`userId`)
- **belongs to** Task (`taskId` — many ScheduleBlocks → one Task)
- **optionally linked to** Event (`eventId` — when the block is also a calendar Event)
- **produced by** SchedulingEngine
- **energyScore sourced from** ContextSnapshot (user energy at time of scheduling)
- **status transitions driven by** AgentBehaviorController (accept / reject recommendations)

## 5. Usage Examples

### Example 1: AI-Generated Proposed Block (High-Priority Task, Morning Productive Slot)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "user-001",
  "taskId": "task-high-priority-001",
  "eventId": null,
  "startTime": "2026-06-20T09:00:00Z",
  "endTime": "2026-06-20T10:30:00Z",
  "durationMinutes": 90,
  "energyScore": 92.5,
  "status": "Proposed",
  "source": "AIGenerated",
  "constraintsSatisfied": {
    "withinWorkingHours": true,
    "noConflict": true,
    "beforeDeadline": true,
    "preferredTime": true,
    "energyLevelAdequate": true
  },
  "createdAt": "2026-06-19T18:30:00Z",
  "updatedAt": "2026-06-19T18:30:00Z"
}
```

### Example 2: User-Manually-Created Block (Low-Priority Task, Afternoon Slot)

```json
{
  "id": "660e8400-e29b-41d4-a716-446655440001",
  "userId": "user-001",
  "taskId": "task-low-priority-002",
  "eventId": "event-calendar-002",
  "startTime": "2026-06-20T14:00:00Z",
  "endTime": "2026-06-20T15:00:00Z",
  "durationMinutes": 60,
  "energyScore": 65.0,
  "status": "Accepted",
  "source": "UserManual",
  "constraintsSatisfied": {
    "withinWorkingHours": true,
    "noConflict": true,
    "beforeDeadline": true,
    "preferredTime": false,
    "energyLevelAdequate": true
  },
  "createdAt": "2026-06-20T08:00:00Z",
  "updatedAt": "2026-06-20T08:30:00Z"
}
```

## 6. Notes for Future Extensions

- Automatic rescheduling suggestions when a block is rejected
- Buffer blocks: automatic break insertion between consecutive blocks
- Drag-and-drop UI integration for manual block override on calendar view
- Cross-user block coordination for collaborative tasks requiring synchronised schedules
- Block effectiveness scoring after task completion (predicted vs. actual duration and energy)
