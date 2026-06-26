# SchedulingEngine

## 1. Purpose

The `SchedulingEngine` finds optimal time slots for tasks using a **Constraint Satisfaction Problem (CSP)** approach enhanced with ML-based slot scoring. It bridges the gap between a user's raw task list and a fully scheduled calendar by balancing hard scheduling rules against soft personal preferences, then using learned models to rank candidate slots before committing to a final schedule.

---

## 2. Inputs

### Hard Constraints *(must be satisfied — violations discard a candidate slot)*

| Field | Type | Description |
|---|---|---|
| `workingHours` | `TimeRange` | Permitted scheduling window per day |
| `existingEvents` | `Event[]` | Committed calendar entries that cannot be moved |
| `taskDeadlines` | `Date[]` | Latest allowable completion times per task |
| `dependencies` | `TaskDependency[]` | Ordering rules between tasks (A must precede B) |

### Soft Constraints *(preferences — violations reduce a slot's score)*

| Field | Type | Description |
|---|---|---|
| `preferredTimes` | `TimeRange[]` | User-declared preferred working windows |
| `breakPreferences` | `BreakPattern` | Desired break cadence and duration |
| `energyLevels` | `EnergyPattern` | Historical and predicted energy curve across the day |
| `focusTimeNeeded` | `number` | Minimum uninterrupted block length for deep work (minutes) |

### Task List

- Unscheduled `Task[]` objects sourced from **TaskManager**, each carrying:
  - `id`, `title`, `estimatedDuration`
  - `deadline`, `priority`, `complexity`
  - `dependencies` (references to prerequisite task IDs)

---

## 3. Outputs

**`ScheduledTask[]`** — an ordered list of tasks assigned to specific calendar time slots.

Each `ScheduledTask` entry contains:

- `taskId` — reference to the originating task
- `startTime` / `endTime` — the assigned slot boundaries (stored in UTC)
- `confidenceScore` — a `0.0–1.0` value indicating the model's confidence that this slot will lead to successful task completion
- `rationale` — a human-readable string explaining why this slot was chosen (e.g., `"High energy period, no adjacent meetings, matches preferred morning window"`)

These results are returned to the **AgentBehaviorController** as scheduling recommendations, and are subsequently written back to **TaskManager** to populate each task's `scheduledTime` field.

---

## 4. Responsibilities

- **Feasibility filtering** — eliminate all candidate time slots that violate any hard constraint before scoring begins
- **ML-based slot scoring** — rank feasible slots using a trained model that combines temporal features with user-specific signals
- **Energy-aware placement** — predict the user's energy level at each candidate slot and prefer high-energy periods for complex or high-priority tasks
- **CSP resolution** — perform the final assignment of tasks to slots while respecting all inter-task dependencies and mutual exclusion rules
- **Post-processing** — apply time buffers between blocks, merge adjacent tasks where appropriate, and enforce break patterns
- **Explainability** — populate the `rationale` field on every output slot so recommendations remain transparent to the user

---

## 5. Internal Logic

The core logic lives in the [`ScheduleOptimizer`](../../docs/architecture/02_architecture_ai_ml.md) class and runs in four sequential phases:

### Phase 1 — `findFeasibleSlots(tasks, constraints)`

Iterates over all candidate time slots within the scheduling horizon and discards any slot that:

- falls outside `workingHours`
- overlaps with an entry in `existingEvents`
- would push a task past its `deadline`
- violates an ordering rule in `dependencies`

Only slots that pass **all** hard constraints are forwarded to Phase 2.

### Phase 2 — `scoreTimeSlots(feasibleSlots)`

Each feasible slot is scored by the ML scheduling model using the following feature vector:

| Feature | Source |
|---|---|
| `timeOfDay` | `slot.startTime.getHours()` |
| `dayOfWeek` | `slot.startTime.getDay()` |
| `userEnergyLevel` | Result of `predictEnergyLevel()` (see below) |
| `taskComplexity` | `task.complexity` field |
| `historicalSuccessRate` | Lookup against the user's past task completion data |

The model returns a scalar score per slot. All slots are returned as `ScoredSlot[]` with the original fields plus `score`.

#### Energy Level Sub-model — `predictEnergyLevel(time, userId)`

A separate regression model predicts the user's energy on a **0–100** scale at any given moment. Its feature vector is:

```typescript
{
  hour:               number,   // 0–23
  dayOfWeek:          number,   // 0–6
  recentSleepQuality: number,   // from UserBehaviorProfile
  recentProductivity: number    // from UserBehaviorProfile
}
```

- Values are sourced from the user's `UserBehaviorProfile` maintained by **ContextAwareness**
- Predicted energy directly feeds the `userEnergyLevel` feature in Phase 2
- **High-energy periods are preferred for tasks with high `complexity` or high `priority`**

### Phase 3 — `solveCSP(scoredSlots, constraints)`

Runs an optimization pass over the scored slots to produce a conflict-free, dependency-respecting assignment:

- Tasks are assigned greedily to their highest-scored feasible slot
- The solver backtracks if a downstream dependency becomes infeasible after an assignment
- Soft constraints (preferred times, break patterns) act as tie-breakers between equal-scoring slots

### Phase 4 — Post-processing

After the CSP solver produces a raw schedule, the following transformations are applied:

- **Buffer insertion** — adds transition time between consecutive blocks (informed by `ConflictDetector`'s soft-conflict rules for back-to-back meetings)
- **Block merging** — adjacent same-task or same-category blocks are consolidated where doing so respects `focusTimeNeeded`
- **Break enforcement** — ensures `breakPreferences` are honoured, inserting mandatory breaks if the solver packed tasks too tightly
- **Time zone normalisation** — all stored times are in UTC; display conversion is handled by `TimeZoneConverter`

---

## 6. Interactions with Other Modules

```
TaskManager  ──────────────────────►  SchedulingEngine
UserPreferencesManager  ────────────►  SchedulingEngine
ContextAwareness  ──────────────────►  SchedulingEngine
Calendar Service  ──────────────────►  SchedulingEngine
                                             │
                    ┌────────────────────────┴──────────────────────┐
                    ▼                                               ▼
        AgentBehaviorController                             TaskManager
        (scheduling recommendations)                (update scheduledTime field)
```

### Reads

| Module | Data Consumed |
|---|---|
| **TaskManager** | Unscheduled tasks and their priorities, deadlines, durations, and dependency graph |
| **UserPreferencesManager** | Working hours, break preferences, focus time requirements |
| **ContextAwareness** (`UserBehaviorProfile`) | Historical energy patterns, recorded productive hours, recent sleep and productivity signals |
| **Calendar Service** — `AvailabilityCalculator` | Free/busy slots derived from merged participant calendars, pre-filtered by working hours and ranked by time-of-day preference |
| **Calendar Service** — `ConflictDetector` | Hard conflicts (overlap), soft conflicts (back-to-back), travel conflicts, and attendee conflicts for the candidate slots under consideration |

### Writes / Returns

| Module | Data Provided |
|---|---|
| **AgentBehaviorController** | `ScheduledTask[]` delivered as scheduling recommendations for the agent to present or act upon |
| **TaskManager** | Scheduled slot assignments written back to update each task's `scheduledTime` field |

### Notes on Calendar Service Integration

- `AvailabilityCalculator` merges participant calendars, identifies busy periods, finds free slots of the required duration, filters by working hours, and ranks by preference before handing candidate slots to the `SchedulingEngine`
- `ConflictDetector` evaluates four conflict severities — **hard** (complete overlap), **soft** (back-to-back without buffer), **travel** (insufficient travel time between locations), and **attendee** (attendee has a conflicting commitment) — which directly influence Phase 1 feasibility filtering and Phase 4 buffer insertion
- `TimeZoneConverter` ensures all scheduling arithmetic is performed in UTC; user-facing output is converted to the user's local time zone, including correct handling of DST transitions
