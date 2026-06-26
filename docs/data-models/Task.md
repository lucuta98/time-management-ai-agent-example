# Task

## 1. Model Purpose

The Task model represents a discrete unit of work in the time management system. Each task captures the user's intention, effort requirements, and progress state. Tasks are the primary input to the PrioritizationEngine, enabling intelligent scheduling and resource allocation. Tasks support recurrence patterns, dependency tracking, and hierarchical composition through self-referential subtask relationships.

## 2. Fields and Types

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique identifier for the task |
| `userId` | UUID | FK to UserProfile; the task owner |
| `title` | String | Task name (1–200 characters) |
| `description` | RichText | Detailed task description |
| `status` | Enum | Current state: `NotStarted`, `InProgress`, `Completed`, `Blocked`, `Cancelled` |
| `priority` | Integer | Calculated score 0–100 (ML + urgency + importance + dependencies) |
| `priorityLevel` | Enum | Human-readable priority: `High`, `Medium`, `Low` |
| `dueDate` | DateTime | Target completion deadline (UTC) |
| `estimatedDuration` | Integer | Predicted effort in minutes |
| `actualDuration` | Integer | Time spent on the task in minutes |
| `tags` | Array\<String\> | User-defined labels for categorisation |
| `category` | String | Organisational category (e.g. "development", "admin") |
| `dependencies` | Array\<UUID\> | IDs of blocking tasks; no circular references allowed |
| `recurrencePattern` | RecurrenceRule | Optional; defines recurring task schedule |
| `assigneeId` | UUID | FK to UserProfile; task owner or delegate |
| `createdAt` | DateTime | Timestamp of task creation (UTC) |
| `updatedAt` | DateTime | Timestamp of last modification (UTC) |
| `completedAt` | DateTime | Timestamp when status transitioned to `Completed`; null otherwise |

### Priority Calculation Formula

```
Priority Score = (mlScore × 0.4) + (urgencyScore × 0.3) + (importanceScore × 0.2) + (dependencyScore × 0.1)
```

- **mlScore** (0–100): ML-derived score based on effort difficulty and user productivity patterns
- **urgencyScore** (0–100): Proximity to `dueDate` relative to `estimatedDuration`
- **importanceScore** (0–100): User-assigned importance or category relevance
- **dependencyScore** (0–100): Number and priority of blocking tasks

### Status Lifecycle

```
NotStarted → InProgress → Completed
                        ↓
                     Blocked → InProgress | Cancelled
                        ↓
                     Cancelled
```

## 3. Constraints

- `title`: 1–200 characters, required
- `priority`: Integer 0–100
- `dueDate`: Must be a valid present or future DateTime
- `estimatedDuration`: Positive integer (minutes)
- `actualDuration`: Non-negative integer (minutes)
- `dependencies`: No circular references; enforced at write time
- `recurrencePattern`: Optional; if present must conform to a valid RecurrenceRule

## 4. Relationships

- **belongs to** UserProfile (`userId`)
- **has many** subtasks (self-referential via `dependencies` array)
- **linked to** ScheduleBlock (one Task → one or many ScheduleBlocks)
- **linked to** AnalyticsRecord (completion and duration events logged)
- **feeds** PrioritizationEngine (status, dependencies, urgency, importance)

## 5. Usage Examples

### Example 1: High-Priority Task with Deadline

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "title": "Deliver Q4 quarterly report",
  "description": "<p>Compile financial metrics, team achievements, and roadmap updates for executive presentation.</p>",
  "status": "InProgress",
  "priority": 92,
  "priorityLevel": "High",
  "dueDate": "2026-06-25T17:00:00Z",
  "estimatedDuration": 480,
  "actualDuration": 120,
  "tags": ["work", "reporting", "urgent"],
  "category": "Executive",
  "dependencies": [],
  "recurrencePattern": null,
  "assigneeId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "createdAt": "2026-06-18T09:30:00Z",
  "updatedAt": "2026-06-20T14:20:00Z",
  "completedAt": null
}
```

### Example 2: Recurring Low-Priority Task with Subtask Dependencies

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440001",
  "userId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "title": "Weekly team sync preparation",
  "description": "<p>Prepare agenda, collect updates from team members, and organise discussion points.</p>",
  "status": "NotStarted",
  "priority": 35,
  "priorityLevel": "Low",
  "dueDate": "2026-06-27T10:00:00Z",
  "estimatedDuration": 60,
  "actualDuration": 0,
  "tags": ["recurring", "team", "admin"],
  "category": "Meetings",
  "dependencies": [
    "550e8400-e29b-41d4-a716-446655440010",
    "550e8400-e29b-41d4-a716-446655440011"
  ],
  "recurrencePattern": {
    "frequency": "WEEKLY",
    "interval": 1,
    "daysOfWeek": ["FRIDAY"],
    "startDate": "2026-06-27T10:00:00Z",
    "endDate": null
  },
  "assigneeId": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "createdAt": "2026-06-01T08:00:00Z",
  "updatedAt": "2026-06-20T11:45:00Z",
  "completedAt": null
}
```

## 6. Notes for Future Extensions

- AI-predicted effort scoring per subtask using historical completion data
- Collaborative ownership (multi-assignee) with per-user roles: owner, collaborator, reviewer
- Task templates with pre-filled fields for recurring workflows
- Dependency graph visualisation (critical path analysis, Gantt chart integration)
- Bidirectional sync with external trackers (Todoist, Asana, Jira)
