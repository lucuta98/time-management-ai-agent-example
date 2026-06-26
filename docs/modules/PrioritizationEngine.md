# PrioritizationEngine

## 1. Purpose

The PrioritizationEngine calculates an intelligent, normalized priority score (0–100) for every task in the system. It exists to answer the question _"how urgently should this task be acted upon right now?"_ in a way that is richer than a simple user-supplied label.

It achieves this by combining two complementary signals:

- **ML prediction** — a Gradient Boosting Classifier (`XGBoost`) trained on historical task completion records, which captures learned behavioural patterns that are difficult to encode as hand-written rules.
- **Business rules** — deterministic components for urgency, importance, and blocking-dependency pressure, which provide interpretable, always-correct baselines even when model confidence is low.

The engine is implemented as the `PriorityCalculator` service inside the **Task Service Application Layer**. The ML model itself (`PriorityClassifier`) lives in the **AI/ML Service** under `ml/models/PriorityClassifier` and is called remotely.

---

## 2. Inputs

### Task Data (from TaskManager)

The engine receives a `Task` object on every `CreateTask` and `UpdateTask` call. The following fields are consumed directly:

| Field | Type | Usage |
|---|---|---|
| `dueDate` | `DateTime` | Derives `daysUntilDue`, `hoursUntilDue`, `hasDeadline` |
| `estimatedDuration` | `Duration` | Feature for the ML model |
| `taskCategory` | `string` | Feature for the ML model |
| `hasSubtasks` | `boolean` | Feature for the ML model |
| `dependencies` | `UUID[]` | Derives `dependencyCount` and `dependencyScore` |
| `isRecurring` | `boolean` | Feature for the ML model |
| `isBlocking` | `boolean` | Feature for the ML model |
| `userDefinedPriority` | `number` | Used directly as `importanceScore` base |

### User Context (from UserPreferencesManager)

- User-defined priority weights (overrides for the importance component of the formula).

### Behavioural Context (from ContextAwareness — `UserBehaviorProfile`)

| Field | Usage |
|---|---|
| `currentWorkload` | ML feature — how busy the user currently is |
| `recentCompletionRate` | ML feature — the user's recent task-finishing velocity |

### Historical Patterns (derived from stored task records)

| Field | Usage |
|---|---|
| `avgTimeToComplete` | ML feature |
| `completionSuccessRate` | ML feature |
| `postponementCount` | ML feature |

### Temporal Context (derived at call time)

- `dayOfWeek`, `hourOfDay` — extracted from the current wall-clock time at scoring time.

---

## 3. Outputs

### Priority Score

A single **integer in the range 0–100**, produced by:

```typescript
Math.round(finalScore * 100)
```

This value is:

- Returned directly to **TaskManager**, which persists it on the task record.
- Forwarded to **SchedulingEngine** when tasks are ranked for time-slot assignment.

### Feature Importance (model introspection)

The underlying `XGBoost` model exposes `model.feature_importances_`, which can be surfaced for explainability purposes via the AI/ML Service API.

---

## 4. Responsibilities

- **Feature extraction** — assemble the full `PriorityFeatures` object from the raw task, user context, and behavioural context before invoking the model.
- **ML inference** — call `PriorityClassifier` in the AI/ML Service and receive a predicted priority class (1–5).
- **Business-rule scoring** — independently compute `urgencyScore`, `importanceScore`, and `dependencyScore` using deterministic formulas.
- **Score fusion** — combine ML and rule-based components using the defined weighted formula.
- **Score normalisation** — convert the final float into an integer 0–100.
- **Re-scoring on update** — recalculate and overwrite the stored score whenever a task is updated through `UpdateTask`.

---

## 5. Internal Logic

### 5.1 Feature Vector

Before any scoring, the engine builds the following typed feature object:

```typescript
interface PriorityFeatures {
  // Temporal
  daysUntilDue: number
  hoursUntilDue: number
  dayOfWeek: number        // 0 (Sunday) – 6 (Saturday)
  hourOfDay: number        // 0–23

  // Task characteristics
  estimatedDuration: number
  taskCategory: string
  hasSubtasks: boolean
  dependencyCount: number

  // User context
  currentWorkload: number
  recentCompletionRate: number
  userDefinedPriority: number

  // Historical patterns
  avgTimeToComplete: number
  completionSuccessRate: number
  postponementCount: number

  // Contextual flags
  isRecurring: boolean
  hasDeadline: boolean
  isBlocking: boolean
}
```

### 5.2 ML Model (`PriorityClassifier`)

- **Algorithm**: Gradient Boosting Classifier — `XGBoost`
- **Task**: multi-class classification (`multi:softmax`)
- **Output classes**: `num_class = 5` → priority levels 1–5
- **Key hyperparameters**:

```python
params = {
    'objective': 'multi:softmax',
    'num_class': 5,
    'max_depth': 6,
    'learning_rate': 0.1,
    'n_estimators': 100,
    'subsample': 0.8,
    'colsample_bytree': 0.8
}
```

- **Training data**: historical task completion records.
- **Evaluation metric**: per-priority-level classification accuracy.
- **Location**: `ml/models/PriorityClassifier` inside the AI/ML Service.

### 5.3 Business-Rule Components

| Component | Formula |
|---|---|
| `urgencyScore` | `f(deadline, current_time)` — higher as `daysUntilDue` → 0; spikes when overdue |
| `importanceScore` | `userDefinedPriority + AI_adjustment` |
| `dependencyScore` | `count(blocking_tasks)` that depend on this task |

### 5.4 Score Fusion Formula

```typescript
async function calculatePriority(task: Task): Promise<number> {
  const features = extractPriorityFeatures(task)

  const mlScore         = await priorityModel.predict(features)   // class 1–5, normalised
  const urgencyScore    = calculateUrgency(task.dueDate)
  const importanceScore = task.userDefinedPriority                // + AI adjustment
  const dependencyScore = calculateDependencyScore(task)

  const finalScore = (
    mlScore         * 0.4 +
    urgencyScore    * 0.3 +
    importanceScore * 0.2 +
    dependencyScore * 0.1
  )

  return Math.round(finalScore * 100)   // integer 0–100
}
```

**Weight rationale:**

| Component | Weight | Rationale |
|---|---|---|
| `mlScore` | 40 % | Captures learned patterns beyond simple rules |
| `urgencyScore` | 30 % | Time pressure is the strongest operational signal |
| `importanceScore` | 20 % | Respects explicit user intent |
| `dependencyScore` | 10 % | Prevents blocking other tasks from being overlooked |

---

## 6. Interactions with Other Modules

```
TaskManager
  │  CreateTask / UpdateTask (Task object)
  ▼
PrioritizationEngine
  │── calls ──▶ AI/ML Service › PriorityClassifier   (ML score)
  │── reads ──▶ UserPreferencesManager                (user-defined priority weights)
  │── reads ──▶ ContextAwareness › UserBehaviorProfile (currentWorkload, recentCompletionRate)
  │
  └── returns priority score (0–100) ──▶ TaskManager  (persisted on task record)
                                     ──▶ SchedulingEngine (task ranking for slot assignment)
```

| Counterpart | Direction | Purpose |
|---|---|---|
| **TaskManager** | Inbound (caller) | Triggers scoring on `CreateTask` and `UpdateTask`; receives the final score |
| **AI/ML Service** (`PriorityClassifier`) | Outbound (remote call) | Returns the ML-predicted priority class used as `mlScore` |
| **UserPreferencesManager** | Outbound (read) | Supplies the user-defined priority value and any custom weighting overrides |
| **ContextAwareness** (`UserBehaviorProfile`) | Outbound (read) | Supplies `currentWorkload` and `recentCompletionRate` as ML features |
| **SchedulingEngine** | Outbound (consumer) | Reads the persisted priority score to rank tasks when assigning calendar time slots |
