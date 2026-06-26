# ContextSnapshot

## 1. Model Purpose

ContextSnapshot is a point-in-time capture of the user's current context as assembled by the ContextAwareness module. It combines real-time signals (current time, workload, calendar state) with learned behavioural patterns (productive hours, detected anomalies) to give the AI layer — specifically the AgentBehaviorController and SchedulingEngine — a unified, up-to-date view of the user's state at the moment a recommendation or scheduling decision is made.

Snapshots are short-lived (TTL-bounded) to prevent stale data from influencing decisions.

## 2. Fields and Types

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique snapshot identifier |
| `userId` | UUID | FK to UserProfile |
| `capturedAt` | DateTime | UTC timestamp when this snapshot was taken |
| `currentTime` | DateTime | Effective "now" for this snapshot |
| `dayOfWeek` | String | e.g. `"Monday"` |
| `hourOfDay` | Integer | Current hour 0–23 |
| `pendingTaskCount` | Integer | Number of open (non-completed) tasks |
| `overdueTaskCount` | Integer | Tasks past their `dueDate` |
| `tasksCompletedToday` | Integer | Tasks completed since midnight (user's timezone) |
| `estimatedRemainingWorkMinutes` | Integer | Sum of `estimatedDuration` for all pending tasks |
| `nextEventStartsAt` | DateTime | Start time of the next upcoming calendar event; nullable |
| `minutesUntilNextEvent` | Integer | Minutes from `capturedAt` to next event; nullable |
| `meetingLoadToday` | Integer | Total meeting minutes already scheduled today |
| `energyLevel` | Float | Predicted energy level 0–100 (from productivity pattern model) |
| `productivityScore` | Float | Current productivity level 0–100 |
| `detectedPatterns` | Array\<String\> | Active pattern labels, e.g. `["deep_work_window", "pre_meeting_buffer"]` |
| `anomalies` | Array\<Anomaly\> | Active anomalies (see Anomaly structure below) |
| `workingHoursActive` | Boolean | `true` if `capturedAt` falls within the user's configured working hours |
| `preferredFocusWindow` | Boolean | `true` if this hour is a known high-productivity period for this user |
| `ttlSeconds` | Integer | Seconds until this snapshot is considered stale (default: 300) |

### Anomaly Structure

```json
{
  "type": "PRODUCTIVITY_DROP",
  "severity": "medium",
  "description": "Productivity score dropped 35 points since morning peak"
}
```

**Anomaly types** (from AnomalyDetection via Isolation Forest):
`UNUSUAL_WORK_HOURS` · `PRODUCTIVITY_DROP` · `EXCESSIVE_WORKLOAD` · `MISSED_DEADLINES` · `UNUSUAL_TASK_DURATION`

**Severity levels**: `low` · `medium` · `high`

## 3. Constraints

- `capturedAt`: Required; must be in UTC
- `energyLevel`: Float 0.0–100.0
- `productivityScore`: Float 0.0–100.0
- `hourOfDay`: Integer 0–23
- `ttlSeconds`: Positive integer; default 300
- A snapshot older than `ttlSeconds` is **stale** and must be regenerated before any scheduling or recommendation decision is made

## 4. Relationships

- **belongs to** UserProfile (`userId`)
- **produced by** ContextAwareness module (PatternRecognition via K-means/DBSCAN + AnomalyDetection via Isolation Forest)
- **consumed by** SchedulingEngine (`energyLevel`, `workingHoursActive`, `preferredFocusWindow`)
- **consumed by** AgentBehaviorController (full snapshot for recommendation ranking and filtering)
- **`energyLevel` copied into** ScheduleBlock at creation time
- **`anomalies`** may trigger proactive recommendations via AgentBehaviorController

## 5. Usage Examples

### Example 1: Healthy Morning Snapshot (High Energy, Deep Work Window, No Anomalies)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "550e8400-e29b-41d4-a716-446655440001",
  "capturedAt": "2026-06-20T09:30:00Z",
  "currentTime": "2026-06-20T09:30:00Z",
  "dayOfWeek": "Saturday",
  "hourOfDay": 9,
  "pendingTaskCount": 5,
  "overdueTaskCount": 0,
  "tasksCompletedToday": 2,
  "estimatedRemainingWorkMinutes": 240,
  "nextEventStartsAt": "2026-06-20T11:00:00Z",
  "minutesUntilNextEvent": 90,
  "meetingLoadToday": 60,
  "energyLevel": 92.5,
  "productivityScore": 88.0,
  "detectedPatterns": ["deep_work_window", "morning_focus_peak"],
  "anomalies": [],
  "workingHoursActive": true,
  "preferredFocusWindow": true,
  "ttlSeconds": 300
}
```

### Example 2: Anomalous Afternoon Snapshot (Productivity Drop + Excessive Workload)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440002",
  "userId": "550e8400-e29b-41d4-a716-446655440001",
  "capturedAt": "2026-06-20T15:45:00Z",
  "currentTime": "2026-06-20T15:45:00Z",
  "dayOfWeek": "Saturday",
  "hourOfDay": 15,
  "pendingTaskCount": 12,
  "overdueTaskCount": 2,
  "tasksCompletedToday": 5,
  "estimatedRemainingWorkMinutes": 480,
  "nextEventStartsAt": "2026-06-20T16:00:00Z",
  "minutesUntilNextEvent": 15,
  "meetingLoadToday": 180,
  "energyLevel": 38.0,
  "productivityScore": 42.5,
  "detectedPatterns": ["afternoon_energy_dip", "context_switching_pattern"],
  "anomalies": [
    {
      "type": "PRODUCTIVITY_DROP",
      "severity": "medium",
      "description": "Productivity score dropped 35 points since morning peak; below user's historical afternoon baseline"
    },
    {
      "type": "EXCESSIVE_WORKLOAD",
      "severity": "high",
      "description": "Estimated remaining work (480 min) significantly exceeds the typical working day; 2 tasks are overdue"
    }
  ],
  "workingHoursActive": true,
  "preferredFocusWindow": false,
  "ttlSeconds": 300
}
```

## 6. Notes for Future Extensions

- Real-time streaming context (sub-minute refresh via WebSocket push)
- Device and location context signals (office vs. home vs. mobile)
- Mood and wellbeing self-report integration
- Multi-day rolling context trends (not just point-in-time snapshots)
- Cross-user context aggregation for team-level coordination
