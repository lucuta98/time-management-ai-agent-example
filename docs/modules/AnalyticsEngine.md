# AnalyticsEngine

## 1. Purpose

The AnalyticsEngine is responsible for collecting, processing, and aggregating behavioural and operational events emitted by all services in the system. It transforms raw event streams into structured metrics, usage statistics, and per-user behaviour profiles that directly power AI features — including contextual recommendations, priority scoring, and ML model retraining cycles.

It acts as the system's observability and intelligence backbone: every action a user takes eventually flows through the AnalyticsEngine, where it is distilled into durable, queryable knowledge about how the system and its users behave over time.

---

## 2. Inputs

### Domain Events (async, via Message Queue)

The AnalyticsEngine subscribes to domain events published by all core services:

| Source Service       | Events Consumed                                                  |
|----------------------|------------------------------------------------------------------|
| **Task Service**     | `TaskCreated`, `TaskCompleted`, `TaskUpdated`                    |
| **Calendar Service** | `EventCreated`, `CalendarSynced`                                 |
| **User Service**     | `Login`, `PreferencesChanged`                                    |
| **Time Tracking Service** | `TimerStarted`, `TimerStopped`                              |
| **AI/ML Service**    | `RecommendationGenerated`, `RecommendationAccepted`              |

Each event payload typically carries a `userId`, a `timestamp`, and event-specific fields (e.g., `taskId`, `duration`, `priority`).

### Direct API Requests

- `POST /analytics/time-entries` — manually logged time entries from clients
- `GET /analytics/productivity` — requests for computed productivity metrics
- `GET /analytics/time-distribution` — requests for category-level time breakdowns
- `GET /analytics/reports/{type}` — on-demand report generation (daily, weekly, monthly)

### Scheduled Batch Triggers

- Hourly, daily, and weekly cron jobs that initiate aggregation and rollup computations against the stored event log.

---

## 3. Outputs

### Per-User Productivity Metrics

Computed and stored after each aggregation cycle:

- **Task completion rate** — completed tasks ÷ total tasks created in period
- **On-time completion rate** — tasks completed before deadline ÷ total completed
- **Average task duration vs. estimate** — actual `duration` vs. original `estimatedDuration`
- **Productivity score by hour/day** — composite score derived from focus time, completion rate, and work pattern alignment
- **Streak data** — consecutive days meeting a productivity threshold

### System-Level Metrics

- Active user counts (real-time)
- Event throughput (events per second / minute)
- Service latency percentiles
- Error rates per service

### User Behaviour Profiles (`UserBehaviorProfile`)

Structured profiles built from aggregated per-user data, including:
- Typical productive hours
- Task category distribution
- Meeting-time vs. focus-time ratio
- Historical goal progress
- Recommendation acceptance rate

### ML Training Dataset Exports

Snapshots of anonymised, labelled event data formatted for retraining ML models (e.g., priority classifiers, scheduling optimisers).

### Report Artefacts

Generated documents/JSON payloads surfacing aggregated metrics over a requested time window, delivered through the Analytics REST API.

---

## 4. Responsibilities

- **Event collection** — consume and durably store all domain events from the message queue without loss.
- **Real-time aggregation** — maintain live counters and sliding-window metrics (active users, current workload, today's task count) via a stream processor (e.g., Kafka Streams).
- **Batch aggregation** — run scheduled jobs (hourly / daily / weekly) to compute historical rollups, productivity trends, and time-distribution breakdowns.
- **Behaviour profile generation** — build and refresh `UserBehaviorProfile` objects from accumulated event history; these profiles feed the `ContextAwareness` module and `PrioritizationEngine`.
- **Productivity metric computation** — calculate composite indicators such as completion rate, on-time rate, duration accuracy, and productivity score per user per period.
- **Report generation** — expose pre-built and on-demand report endpoints for frontend dashboards.
- **ML training data export** — produce labelled, anonymised datasets on a scheduled cadence for model retraining pipelines.
- **Data retention enforcement** — apply configurable retention policies; age-out or archive raw events beyond the retention window.
- **Goal progress tracking** — monitor user-defined goals (time allocation, task completion, habit formation) and emit progress updates.

---

## 5. Internal Logic

### 5.1 Aggregation Pipeline

Raw events flow through a multi-stage pipeline before they are stored as metrics:

```
Raw Events → Filter → Transform → Aggregate → Store → Query
```

1. **Filter** — drop malformed, duplicate, or out-of-retention-window events.
2. **Transform** — normalise event fields into a canonical schema; enrich with user timezone and category metadata.
3. **Aggregate** — apply real-time or batch aggregation logic (see §5.2 and §5.3).
4. **Store** — write aggregates to the appropriate storage backend (see §5.4).
5. **Query** — serve results through the Analytics API.

### 5.2 Real-Time Processing Path

Incoming events are consumed from the message queue by a stream processor. The stream processor maintains in-memory state for sliding-window computations:

- **Active user count** — users who have emitted any event in the last 15 minutes.
- **Current workload** — open tasks per user, updated on each `TaskCreated` / `TaskCompleted` event.
- **Active timer sessions** — currently running `TimerStarted` events without a matching `TimerStopped`.

Results are pushed to live dashboards with sub-second latency.

### 5.3 Batch Processing Path

Scheduled jobs run on a configurable cadence and operate on the full historical event log:

| Cadence  | Computations                                                                 |
|----------|------------------------------------------------------------------------------|
| Hourly   | Rolling productivity score; real-time metric reconciliation                  |
| Daily    | Completion rate, on-time rate, duration accuracy, streak update, goal progress |
| Weekly   | Time-distribution breakdown, behaviour profile refresh, ML dataset snapshot  |

**Example daily rollup query (PostgreSQL)**:

```sql
SELECT
  user_id,
  DATE(completed_at)    AS date,
  COUNT(*)              AS tasks_completed,
  SUM(actual_duration)  AS total_time_minutes,
  AVG(priority_score)   AS avg_priority
FROM tasks
WHERE status = 'Completed'
GROUP BY user_id, DATE(completed_at);
```

### 5.4 Storage Layer

| Store              | Technology       | Contents                                                      |
|--------------------|------------------|---------------------------------------------------------------|
| Event store        | Elasticsearch    | Raw and transformed domain events; full-text + range queries  |
| Analytics indexes  | Elasticsearch    | Pre-aggregated metric documents; dashboard query targets      |
| Aggregated metrics | PostgreSQL       | Daily/weekly rollup rows; `ProductivityMetrics`, `TimeEntry`, `Goal` tables |

**Key data models**:

```
TimeEntry {
  id: UUID, userId: UUID, taskId: UUID, eventId: UUID,
  category: String, startTime: DateTime, endTime: DateTime,
  duration: Integer (minutes), isBillable: Boolean, notes: String
}

ProductivityMetrics {
  userId: UUID, date: Date,
  tasksCompleted: Integer, focusTimeMinutes: Integer,
  meetingTimeMinutes: Integer, breakTimeMinutes: Integer,
  productivityScore: Float, topCategories: Array<Object>
}

Goal {
  id: UUID, userId: UUID, title: String,
  type: Enum(TimeAllocation | TaskCompletion | Habit),
  target: Object, currentProgress: Object,
  startDate: Date, endDate: Date,
  status: Enum(Active | Completed | Abandoned)
}
```

### 5.5 Behaviour Profile Construction

After each weekly batch cycle the AnalyticsEngine rebuilds `UserBehaviorProfile` records by joining productivity metrics, time-distribution data, and recommendation-acceptance history. The profile captures:

- Peak productive hours (hour-of-day histogram derived from high-score sessions)
- Category affinity (proportion of time per task category)
- Estimation accuracy (mean ratio of `actual_duration` to `estimatedDuration`)
- Recommendation receptivity (proportion of `RecommendationAccepted` events to `RecommendationGenerated`)

These profiles are the primary output consumed downstream by `ContextAwareness` and `PrioritizationEngine`.

---

## 6. Interactions with Other Modules

```
┌──────────────────────────────────────────────────────────────────┐
│                         Event Sources                            │
│  TaskManager · Calendar Service · User Service                   │
│  Time Tracking Service · AI/ML Service                           │
└───────────────────────────┬──────────────────────────────────────┘
                            │  Domain Events (async, Message Queue)
                            ▼
               ┌────────────────────────┐
               │     AnalyticsEngine    │
               │  (Event Processor +    │
               │   Batch Jobs)          │
               └──────┬─────────┬───────┘
                      │         │
         ┌────────────┘         └─────────────────────┐
         │                                            │
         ▼                                            ▼
┌─────────────────────┐                  ┌────────────────────────┐
│  ContextAwareness   │                  │     AI/ML Service      │
│  (UserBehavior-     │                  │  (ML model retraining  │
│   Profile input)    │                  │   dataset export)      │
└─────────────────────┘                  └────────────────────────┘
         │
         ▼
┌─────────────────────┐        ┌──────────────────────────────────┐
│ PrioritizationEngine│        │  Frontend / Dashboard Clients    │
│ (productivity       │        │  (REST API — productivity,       │
│  metrics context)   │        │   time-distribution, reports)    │
└─────────────────────┘        └──────────────────────────────────┘
```

| Counterpart                | Direction              | Purpose                                                                                       |
|----------------------------|------------------------|-----------------------------------------------------------------------------------------------|
| **TaskManager**            | Inbound (events)       | Receives `TaskCreated`, `TaskCompleted`, `TaskUpdated` to track task lifecycle metrics.        |
| **Calendar Service**       | Inbound (events)       | Receives `EventCreated`, `CalendarSynced` to measure meeting time and calendar activity.       |
| **User Service**           | Inbound (events)       | Receives `Login`, `PreferencesChanged` to track engagement and preference-shift patterns.      |
| **Time Tracking Service**  | Inbound (events)       | Receives `TimerStarted`, `TimerStopped` to compute actual task durations.                      |
| **AI/ML Service**          | Inbound + Outbound     | Inbound: `RecommendationGenerated`, `RecommendationAccepted` for acceptance-rate tracking. Outbound: exports labelled training datasets for model retraining pipelines. |
| **ContextAwareness**       | Outbound               | Publishes `UserBehaviorProfile` objects used for pattern detection and anomaly analysis.       |
| **PrioritizationEngine**   | Outbound               | Provides aggregated productivity metrics used as contextual features in priority scoring.      |
| **AgentBehaviorController**| Outbound               | Supplies productivity metrics that inform recommendation context and ranking decisions.         |
| **Frontend / Dashboards**  | Outbound (REST API)    | Serves `/analytics/productivity`, `/analytics/time-distribution`, and `/analytics/reports/{type}` endpoints. |
| **Sync Service**           | Inbound (optional)     | May propagate analytics-related preference changes via `ChangeTracker` to keep profiles current. |
