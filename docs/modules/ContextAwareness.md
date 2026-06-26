# ContextAwareness

## 1. Purpose

Build and maintain a live model of the user's behavioural context — their patterns, energy levels, and anomalies — so other modules can make context-aware decisions.

The module is composed of two complementary sub-systems:

- **Pattern Recognition** — continuously analyses historical activity to surface recurring behavioural patterns (productive hours, task habits, break rhythms, procrastination tendencies).
- **Anomaly Detection** — monitors the same activity stream in real time and raises alerts when observed behaviour deviates significantly from the established baseline.

Together they keep a `UserBehaviorProfile` up to date that the rest of the agent consults when scheduling tasks, prioritising work, and triggering proactive recommendations.

---

## 2. Inputs

| Source | Data | Description |
|---|---|---|
| `AnalyticsEngine` | Raw event / activity stream | Time-stamped user activity events consumed as a continuous stream |
| `TaskManager` | Task completion history | Historical records of task start times, durations, categories, deadlines, and outcomes |

**Feature vector extracted for Anomaly Detection:**

```python
features = [
    'work_hours_per_day',
    'tasks_completed',
    'avg_task_duration',
    'missed_deadlines',
    'productivity_score'
]
```

**Feature vector extracted for Pattern Recognition:**

Each entry in the time-series carries at minimum:
- `entry.hour` — hour of day (0–23)
- `entry.productivity_score` — numeric score reflecting output quality/quantity at that hour

---

## 3. Outputs

### 3.1 `UserPatterns` (Pattern Recognition output)

```typescript
interface UserPatterns {
  // Temporal patterns
  productiveHours: TimeRange[]
  preferredWorkDays: number[]
  breakPatterns: BreakPattern[]

  // Task patterns
  taskCompletionPatterns: {
    category: string
    avgDuration: number
    preferredTime: TimeRange
    successRate: number
  }[]

  // Behavioural patterns
  procrastinationTendency: number       // 0–1 scalar
  multitaskingFrequency: number         // average context switches per hour
  planningHorizon: number               // days ahead the user typically plans

  // Embedded anomaly signals
  unusualWorkHours: Date[]
  productivityDrops: Date[]
}
```

### 3.2 `Anomaly` events (Anomaly Detection output)

```typescript
enum AnomalyType {
  UNUSUAL_WORK_HOURS      = 'unusual_work_hours',
  PRODUCTIVITY_DROP       = 'productivity_drop',
  EXCESSIVE_WORKLOAD      = 'excessive_workload',
  MISSED_DEADLINES        = 'missed_deadlines',
  UNUSUAL_TASK_DURATION   = 'unusual_task_duration'
}

interface Anomaly {
  type:           AnomalyType
  severity:       'low' | 'medium' | 'high'
  timestamp:      Date
  description:    string
  recommendation: string
}
```

---

## 4. Responsibilities

- **Collect** time-series activity data from upstream event streams on a rolling basis.
- **Cluster** hourly activity data to identify periods of high and low productivity.
- **Classify** recurring behaviour into named pattern categories (productive hours, break patterns, meeting preferences, procrastination tendency, etc.).
- **Score** each detected pattern with a confidence value so consumers can weight decisions accordingly.
- **Maintain** a continuously updated `UserBehaviorProfile` that consolidates all recognised patterns.
- **Train and serve** an Isolation Forest model to flag statistically unusual behaviour windows.
- **Emit** typed `Anomaly` events with severity ratings and human-readable recommendations whenever an anomaly is detected.
- **Expose** `planningHorizon` and `procrastinationTendency` as first-class signals for proactive agent behaviour.

---

## 5. Internal Logic

### 5.1 Pattern Recognition Pipeline

The pipeline executes in five sequential steps:

1. **Collect** — aggregate time-series entries from the activity stream into a sliding window of historical data.
2. **Apply clustering** — run DBSCAN (and K-means where appropriate) over the feature matrix.
3. **Identify recurring patterns** — group cluster members by pattern category (productive hours, break times, meeting slots, etc.).
4. **Calculate confidence scores** — weight each pattern by cluster density and recency.
5. **Update `UserBehaviorProfile`** — write the latest `UserPatterns` object so downstream modules always read a fresh snapshot.

**Productive-hours detection (reference implementation):**

```python
from sklearn.cluster import DBSCAN
import numpy as np

def detect_productive_hours(time_entries):
    # Build [hour, productivity_score] matrix
    X = np.array([
        [entry.hour, entry.productivity_score]
        for entry in time_entries
    ])

    # DBSCAN: eps=2 hours, require at least 5 samples to form a cluster
    clustering = DBSCAN(eps=2, min_samples=5)
    labels = clustering.fit_predict(X)

    productive_clusters = []
    for label in set(labels):
        if label == -1:          # skip noise points
            continue

        cluster_points  = X[labels == label]
        avg_productivity = cluster_points[:, 1].mean()

        if avg_productivity > 70:   # high-productivity threshold
            hour_range = (
                cluster_points[:, 0].min(),
                cluster_points[:, 0].max()
            )
            productive_clusters.append(hour_range)

    return productive_clusters
```

**DBSCAN configuration:**

| Parameter | Value | Rationale |
|---|---|---|
| `eps` | `2` | Two-hour neighbourhood radius |
| `min_samples` | `5` | Minimum data points to form a dense cluster |
| Productivity threshold | `> 70` | Clusters above this average are labelled *productive* |

**Patterns detected:**

| Pattern | Method | Description |
|---|---|---|
| Productive Hours | DBSCAN on `(hour, productivity_score)` | Windows when the user consistently completes the most work |
| Task Duration | Time-series analysis per category | Typical elapsed time per task type |
| Procrastination | Association rule mining on postponement events | Tasks the user repeatedly defers |
| Meeting Patterns | K-means on `(hour, day_of_week)` | Preferred meeting times and days |
| Break Patterns | Clustering on idle periods | Natural rest intervals |
| Context Switching | Frequency analysis on task-switch events | How often the user changes active task |

### 5.2 Anomaly Detection Pipeline

An **Isolation Forest** model is trained offline on the user's historical feature matrix and served for online inference:

```python
from sklearn.ensemble import IsolationForest

detector = IsolationForest(
    contamination=0.1,   # expected proportion of outlier windows
    random_state=42
)

X = extract_features(user_activity, features)
detector.fit(X)

# Inference on new windows
predictions = detector.predict(X_new)
anomalies   = X_new[predictions == -1]
```

- `contamination=0.1` assumes roughly 10 % of historical windows are genuine outliers.
- Any window scored `-1` by the model triggers construction of a typed `Anomaly` object, which is annotated with a `severity` level (`low` / `medium` / `high`) and a natural-language `recommendation` string before being emitted downstream.

---

## 6. Interactions with Other Modules

```
AnalyticsEngine  ──(event stream)──────────────────► ContextAwareness
TaskManager      ──(completion history)────────────► ContextAwareness

ContextAwareness ──(UserBehaviorProfile)────────────► PrioritizationEngine
                    (workload, completion-rate features)

ContextAwareness ──(productive hours / energy)──────► SchedulingEngine
                    (slot scoring weights)

ContextAwareness ──(Anomaly events)─────────────────► AgentBehaviorController
                    (triggers proactive recommendations)

ContextAwareness ──(planningHorizon,                ► AgentBehaviorController
                    procrastinationTendency)
```

| Consumer | Data provided | How it is used |
|---|---|---|
| `PrioritizationEngine` | `UserBehaviorProfile` (`workload`, `completionRate` features) | Adjusts task priority scores based on current user capacity |
| `SchedulingEngine` | `productiveHours`, energy level patterns | Scores candidate time slots; prefers high-energy windows for demanding tasks |
| `AgentBehaviorController` | `Anomaly` events (`type`, `severity`, `recommendation`) | Decides whether and how to surface a proactive intervention to the user |
| `AgentBehaviorController` | `planningHorizon`, `procrastinationTendency` | Tunes reminder lead times and nudge aggressiveness |
