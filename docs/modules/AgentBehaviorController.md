# AgentBehaviorController

## 1. Purpose

The `AgentBehaviorController` is the top-level orchestrator of all AI subsystems within the Time-Management Agent. It acts as the central coordination layer that ties together the NLU pipeline, `PrioritizationEngine`, `SchedulingEngine`, and `ContextAwareness` module to produce a coherent, ranked set of personalised recommendations.

Its two primary jobs are:

- **Intent routing** — receive the `NLPResult` produced by the NLU pipeline and dispatch each recognised intent to the correct downstream handler (e.g. `TaskManager`, `Calendar Service`, `SchedulingEngine`).
- **Recommendation orchestration** — invoke all specialist sub-engines in sequence, merge their outputs, rank the combined list by confidence and priority, and deliver the final `Recommendation[]` to the API layer.

Every recommendation the controller emits carries a human-readable `rationale` string so that end-users can understand why a suggestion was made (Explainable AI). The controller also respects active A/B experiment variants when selecting which model versions to call, allowing controlled rollouts of new algorithms.

---

## 2. Inputs

### 2.1 NLU Output

The controller receives an `NLPResult` object from the upstream NLU pipeline after a user's natural-language utterance has been parsed:

| Field | Type | Description |
|---|---|---|
| `intent` | `Intent` (enum) | The classified user intent (e.g. `CREATE_TASK`, `RESCHEDULE`) |
| `entities` | `Entity[]` | Extracted entities: title, date, time, duration, priority, person, location |
| `confidence` | `number` | Intent classification confidence score `[0, 1]` |
| `rawText` | `string` | Original user input |

### 2.2 User Context (loaded per request)

The controller loads the following data for the requesting user before generating recommendations:

- **Tasks** — all tasks for the user, including `scheduledTime`, `priority`, `dueDate`, `dependencies`, and `userDefinedPriority`
- **Calendar** — existing events, meetings, and blocked time ranges
- **`UserBehaviorProfile`** — `productiveHours`, `averageTaskDuration`, `completionPatterns`, `breakPreferences`, `priorityPreferences`
- **`UserPreferences`** — `workingHours`, `workingDays`, `notificationSettings`, locale, and display preferences

### 2.3 Experiment Variant Assignment

For any active A/B experiment, the controller receives the assigned `Variant` from the `ExperimentManager` (resolved via consistent hashing on `userId`). The variant may override:

- `modelVersion` — which ML model version to invoke
- `algorithmConfig` — parameter overrides for a given engine

---

## 3. Outputs

### 3.1 Recommendation Array

The primary output is a `Recommendation[]` delivered to the API layer and subsequently pushed to the client. Each object in the array has the following shape:

```typescript
interface Recommendation {
  id: string
  type: RecommendationType
  title: string
  description: string
  rationale: string        // Human-readable explanation (Explainable AI)
  confidence: number       // Composite score used for ranking
  priority: number
  actions: Action[]        // One-click actionable payloads
  expiresAt: Date
}
```

Possible values of `RecommendationType`:

| Value | Meaning |
|---|---|
| `TASK_SCHEDULING` | Suggest when and where to schedule a specific unscheduled task |
| `PRIORITY_ADJUSTMENT` | Suggest raising or lowering a task's priority score |
| `BREAK_SUGGESTION` | Suggest a break based on detected work intensity |
| `FOCUS_TIME` | Suggest blocking calendar white-space for deep work |
| `MEETING_OPTIMIZATION` | Suggest consolidating or rescheduling meetings |
| `WORKLOAD_BALANCE` | Flag overload and suggest deferring low-priority tasks |

### 3.2 Intent Routing Acknowledgement

For intent-driven requests the controller returns the result produced by the downstream handler it routed to (e.g. the newly created task record, the updated schedule, or the confirmation from the Notification Service).

### 3.3 Feedback Events

After a recommendation is accepted or rejected by the user, the controller emits a feedback event to the `AnalyticsEngine` to close the learning loop.

---

## 4. Responsibilities

1. **Receive and validate `NLPResult`** — confirm that the intent classification confidence meets the minimum threshold before dispatching.
2. **Route intents to handlers** — map each `Intent` enum value to the correct downstream service or engine (see §5 for the full routing table).
3. **Load user context** — aggregate tasks, calendar events, `UserBehaviorProfile`, and `UserPreferences` into a single `UserContext` object before calling any engine.
4. **Orchestrate the recommendation pipeline** — invoke each generator in order (scheduling → priority → break → focus-time), collect partial results, and merge them.
5. **Rank and deduplicate recommendations** — sort the merged list by `confidence × priority`, remove duplicates or conflicting suggestions, and cap the result at a configured maximum `N`.
6. **Apply Explainable AI rationale** — ensure every recommendation carries a populated `rationale` string generated by `explainSchedulingDecision()` or `explainPriorityScore()`.
7. **Respect A/B experiment variants** — query `ExperimentManager.assignVariant()` at the start of each request and select the appropriate model version accordingly.
8. **Publish results to the API layer** — emit the final `Recommendation[]` so the API layer can deliver it to the client.
9. **Record feedback** — log user acceptance or rejection of each recommendation back to `AnalyticsEngine` to enable continuous model improvement.

---

## 5. Internal Logic

### 5.1 Intent Routing

After receiving an `NLPResult`, the controller consults the following dispatch table:

| Intent | Handler |
|---|---|
| `CREATE_TASK` | `TaskManager` |
| `CREATE_EVENT` | `Calendar Service` |
| `UPDATE_TASK` | `TaskManager` |
| `DELETE_TASK` | `TaskManager` |
| `QUERY_SCHEDULE` | `SchedulingEngine` |
| `SET_REMINDER` | `Notification Service` |
| `RESCHEDULE` | `SchedulingEngine` + `TaskManager` |

Requests with a confidence score below the configured threshold are rejected with an appropriate user-facing clarification prompt rather than being silently dispatched.

### 5.2 Recommendation Generation Algorithm

The controller runs the following ordered pipeline for every `generateRecommendations(userId)` call:

1. **`getUserContext(userId)`** — load the user's tasks, calendar, `UserBehaviorProfile`, and `UserPreferences` into a unified `UserContext`.

2. **`generateSchedulingRecommendations(context)`** — call `SchedulingEngine.findOptimalSlot()` for each unscheduled task. Include a `TASK_SCHEDULING` recommendation only when `optimalSlot.score > 0.7` (high-confidence threshold).

3. **`generatePriorityRecommendations(context)`** — call `PrioritizationEngine` for tasks whose priority score is stale or has low confidence. Emit a `PRIORITY_ADJUSTMENT` recommendation for each affected task.

4. **`generateBreakRecommendations(context)`** — query `ContextAwareness` for current work-intensity patterns and the user's `breakPreferences`. Emit a `BREAK_SUGGESTION` recommendation when sustained high-intensity work is detected.

5. **`generateFocusTimeRecommendations(context)`** — identify calendar white-space that falls within the user's productive hours (from `UserBehaviorProfile.productiveHours`). Emit `FOCUS_TIME` recommendations for suitable gaps.

6. **`rankRecommendations(all)`** — merge all partial arrays, sort descending by `confidence × priority`, deduplicate overlapping suggestions (same task / same time window), and cap the list at `N` (configurable).

7. **Return `Recommendation[]`** — each item contains `id`, `type`, `title`, `description`, `rationale`, `confidence`, `priority`, `actions[]`, and `expiresAt`.

### 5.3 Explainable AI — Rationale Generation

Each recommendation's `rationale` field is populated by one of two explanation helpers before the result is returned:

**`explainSchedulingDecision(slot: TimeSlot): string`**

Evaluates slot features and concatenates applicable reasons:

- `slot.features.userEnergyLevel > 80` → `"You're typically most productive at this time"`
- `slot.features.historicalSuccessRate > 0.8` → `"You've successfully completed similar tasks at this time"`
- `slot.features.noMeetings` → `"No meetings scheduled, allowing for focused work"`
- `slot.features.beforeDeadline` → `"Leaves buffer time before the deadline"`

**`explainPriorityScore(task, score): string`**

Evaluates task attributes and concatenates applicable factors:

- `urgency > 0.7` (derived from `task.dueDate`) → `"High urgency (due <relative time>)"`
- `task.dependencies.length > 0` → `"Blocks N other tasks"`
- `task.userDefinedPriority === 'high'` → `"Marked as high priority"`

The final string is formatted as: `Priority score: <score>/100. <factors joined by '. '>`.

### 5.4 A/B Experiment Variant Selection

At the start of each request the controller calls:

```typescript
const variant = await experimentManager.assignVariant(userId, experimentId)
```

`ExperimentManager` uses consistent hashing (`hashUserId(userId, experimentId)`) to produce a stable, reproducible bucket assignment. The returned `Variant` object may carry a `modelVersion` or `algorithmConfig` override that the controller passes to the relevant engine. Per-variant metrics tracked include:

- `task_completion_rate`
- `user_satisfaction`
- `time_to_complete`

These are recorded via `ExperimentManager.trackMetric()` at the end of the request.

---

## 6. Interactions with Other Modules

```
                       ┌─────────────────────────────┐
                       │   AgentBehaviorController    │
                       └──────────────┬──────────────┘
          ┌────────────────┬──────────┼──────────┬────────────────┐
          ▼                ▼          ▼          ▼                ▼
       NLU             Prioritiz-  Scheduling  Context-       User
    Pipeline           ationEngine  Engine     Awareness   Preferences
   (NLPResult in)   (priority scores) (slots) (profiles,    Manager
                                              anomalies)  (filter/personalise)
          │                                                        │
          │ intent dispatch                                        │
          ▼                                                        ▼
  ┌───────────────────────────────────────────────────────────────────┐
  │  TaskManager  │  Calendar Service  │  Notification Service         │
  └───────────────────────────────────────────────────────────────────┘
          │
          ▼
  AnalyticsEngine   ◄──── feedback loop (acceptance / rejection events)
          │
          ▼
      API Layer  ──► Client (Recommendation[] delivery)
```

| Counterpart | Direction | Data Exchanged |
|---|---|---|
| **NLU Pipeline** | Inbound | `NLPResult` (intent + entities + confidence) |
| **PrioritizationEngine** | Outbound (call) | Up-to-date priority scores for tasks with stale or low-confidence scores |
| **SchedulingEngine** | Outbound (call) | Optimal time-slot suggestions; also receives `QUERY_SCHEDULE` and `RESCHEDULE` intents |
| **ContextAwareness** | Outbound (read) | `UserBehaviorProfile`, real-time work-intensity data, and anomaly events |
| **UserPreferencesManager** | Outbound (read) | `UserPreferences` used to personalise and filter the final recommendation list |
| **AnalyticsEngine** | Bidirectional | Reads productivity-metrics context before generation; writes recommendation acceptance/rejection events for the feedback loop |
| **TaskManager** | Outbound (dispatch) | Handles `CREATE_TASK`, `UPDATE_TASK`, `DELETE_TASK` intents; also executes `schedule_task` actions from recommendations |
| **Calendar Service** | Outbound (dispatch) | Handles `CREATE_EVENT` intent and `FOCUS_TIME` / `MEETING_OPTIMIZATION` recommendation actions |
| **Notification Service** | Outbound (dispatch) | Handles `SET_REMINDER` intent |
| **ExperimentManager** | Outbound (call) | Resolves active experiment variant for the requesting user; receives metric tracking calls after each request |
| **API Layer** | Outbound (publish) | Receives final `Recommendation[]` for delivery to the client |
