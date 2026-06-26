# AnalyticsRecord

## 1. Model Purpose

AnalyticsRecord is the unified data model for all behavioural and performance events captured by the AnalyticsEngine. It covers raw time-tracking entries, derived daily productivity metrics, and user-defined goals. These records feed the ContextAwareness module (pattern detection and anomaly detection), the PrioritizationEngine (completion rates, current workload), and all user-facing reports and dashboards. Long-term records are stored in the data warehouse (Elasticsearch) for trend analysis.

## 2. Fields and Types

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique record identifier |
| `userId` | UUID | FK to UserProfile |
| `recordType` | Enum | `TimeEntry`, `ProductivityMetric`, or `Goal` — determines which field group is populated |

#### TimeEntry fields (when `recordType = TimeEntry`)

| Field | Type | Description |
|---|---|---|
| `taskId` | UUID | Optional FK to Task; null for non-task activities |
| `eventId` | UUID | Optional FK to Event; set for meeting time entries |
| `category` | String | Activity category, e.g. `"development"`, `"meetings"`, `"admin"` |
| `startTime` | DateTime | Session start (UTC) |
| `endTime` | DateTime | Session end (UTC) |
| `durationMinutes` | Integer | Must equal `endTime − startTime` in minutes |
| `isBillable` | Boolean | Whether this time is billable |
| `notes` | String | Optional free-text annotation |

#### ProductivityMetric fields (when `recordType = ProductivityMetric`)

| Field | Type | Description |
|---|---|---|
| `date` | Date | The calendar day this metric covers |
| `tasksCompleted` | Integer | Tasks completed during this day |
| `focusTimeMinutes` | Integer | Total uninterrupted focus work minutes |
| `meetingTimeMinutes` | Integer | Total time in meetings |
| `breakTimeMinutes` | Integer | Total break time |
| `productivityScore` | Float | Composite score 0–100 |
| `topCategories` | Array\<Object\> | `[{ category: String, minutes: Integer }]` sorted by time |

#### Goal fields (when `recordType = Goal`)

| Field | Type | Description |
|---|---|---|
| `goalTitle` | String | Goal description |
| `goalType` | Enum | `TimeAllocation`, `TaskCompletion`, `Habit` |
| `target` | Object | e.g. `{ targetMinutes: 120, period: "daily" }` |
| `currentProgress` | Object | e.g. `{ achievedMinutes: 90 }` |
| `startDate` | Date | Goal start date |
| `endDate` | Date | Goal end date |
| `goalStatus` | Enum | `Active`, `Completed`, `Abandoned` |

#### Common audit fields

| Field | Type | Description |
|---|---|---|
| `createdAt` | DateTime | Record creation timestamp (UTC) |
| `updatedAt` | DateTime | Last modification timestamp (UTC) |

### Metrics Tracked

- Task completion rates and time distribution by category
- Meeting time vs. focus time ratio
- Productivity trends over time
- Feature adoption rates and engagement signals

## 3. Constraints

- `userId`: Required, FK to UserProfile
- `recordType`: Required; determines which field group is populated
- For `TimeEntry`: `startTime` must be before `endTime`; `durationMinutes` must equal the interval
- For `ProductivityMetric`: one record per `userId` per `date`
- `productivityScore`: Float 0.0–100.0
- For `Goal`: `endDate` must be after `startDate`

## 4. Relationships

- **belongs to** UserProfile (`userId`)
- **references** Task (`taskId`) for TimeEntry records
- **references** Event (`eventId`) for meeting time entries
- **feeds** ContextAwareness (pattern detection via DBSCAN/K-means; anomaly detection via Isolation Forest)
- **feeds** PrioritizationEngine (`recentCompletionRate`, `currentWorkload` features)
- **queried by** AnalyticsEngine to generate reports, dashboards, and goal progress views
- **stored in** data warehouse (Elasticsearch) for long-term trend analysis

## 5. Usage Examples

### Example 1: TimeEntry Record (Completed Focus Work Session)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440001",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "recordType": "TimeEntry",
  "taskId": "550e8400-e29b-41d4-a716-446655440002",
  "eventId": null,
  "category": "development",
  "startTime": "2026-06-20T09:00:00Z",
  "endTime": "2026-06-20T11:30:00Z",
  "durationMinutes": 150,
  "isBillable": true,
  "notes": "Implemented authentication module for API",
  "createdAt": "2026-06-20T11:30:05Z",
  "updatedAt": "2026-06-20T11:30:05Z"
}
```

### Example 2: ProductivityMetric Record (Full Working Day)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440003",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "recordType": "ProductivityMetric",
  "date": "2026-06-20",
  "tasksCompleted": 5,
  "focusTimeMinutes": 360,
  "meetingTimeMinutes": 90,
  "breakTimeMinutes": 50,
  "productivityScore": 82.5,
  "topCategories": [
    { "category": "development", "minutes": 300 },
    { "category": "meetings",    "minutes": 90  },
    { "category": "admin",       "minutes": 60  }
  ],
  "createdAt": "2026-06-21T00:15:00Z",
  "updatedAt": "2026-06-21T00:15:00Z"
}
```

## 6. Notes for Future Extensions

- Real-time streaming ingestion (Kafka / event pipeline for sub-second latency)
- Cross-team aggregated productivity benchmarks
- Burnout risk scoring derived from multi-day productivity trend analysis
- Integration with external time trackers (Toggl, Harvest) for import/export
- Exportable reports in CSV and PDF formats for billing or performance reviews
