# TaskManager

## 1. Purpose

The **TaskManager** is the central domain service of the Task Service. It owns the complete lifecycle of a task — from creation through prioritization, recurrence, and completion — and exposes that lifecycle to the rest of the system through a well-defined layered API.

Its primary reason for existence is to enforce task-domain business rules in one authoritative place, so that no other service needs to understand what "complete a task" or "detect a dependency cycle" means. All reads and writes to task data flow through this module.

---

## 2. Inputs

### HTTP Requests (API Layer)
Received by `TaskController`, `SubtaskController`, and `TaskTemplateController`:

| Operation | Payload |
|---|---|
| Create task | `title`, `description`, `dueDate`, `priority`, `estimatedDuration`, `dependencies[]`, `recurrenceRule` |
| Update task | Task `id` + any subset of mutable fields |
| Delete task | Task `id` |
| Complete task | Task `id` + optional `completionNotes` |
| Bulk create | `Task[]` array |
| List/filter tasks | `TaskFilterDTO`: `userId`, `status`, `priority`, `dateRange`, `tags` |

### Internal / Programmatic Inputs
- **`TaskDTO`** — the application-layer data transfer object carrying validated task fields between the API and use-case handlers.
- **`TaskFilterDTO`** — structured filter criteria passed to `TaskRepository.findByUserId`.
- **Domain events consumed from the message queue** — e.g. priority adjustment signals from the AI/ML Service.
- **Cache reads** — `TaskCache` key pattern `task:{userId}:active` supplies pre-fetched active task lists.

### Validation Constraints (enforced by `TaskValidator`)
- `title`: 1–200 characters
- `dueDate`: must be a valid ISO 8601 date-time string
- `priority`: within the defined enum/range
- `dependencies[]`: must not introduce a circular dependency graph

---

## 3. Outputs

### API Responses
- **Single task object** — returned after create, update, complete, or `findById` operations.
- **Task array** — returned from list/filter queries.
- **Void / acknowledgement** — returned from delete and bulk operations.
- **Validation error** — structured error response when `TaskValidator` rejects input.

### Published Domain Events (via `EventPublisher` → `MessageQueue`)
| Event | Trigger |
|---|---|
| `TaskCreatedEvent` | Successful task creation |
| `TaskUpdatedEvent` | Any field update on an existing task |
| `TaskCompletedEvent` | Task marked as complete |

### Side Effects
- **Persisted records** — rows written or updated in the database via `TaskRepository`.
- **Cache invalidation** — `TaskCache` entries for the owning user are evicted on every mutation.
- **AI recommendations request** — an outbound call to `AIServiceClient` is made after task creation to pre-fetch scheduling and priority recommendations.
- **Calendar time-block request** — sent to the Calendar Service when a task with a due date is created or updated.
- **Notification registration** — sent to the Notification Service so reminders can be scheduled.

---

## 4. Responsibilities

- **Route and parse HTTP requests** for task, subtask, and template operations (`TaskController`, `SubtaskController`, `TaskTemplateController`).
- **Validate all incoming data** against schema rules and domain business rules (`TaskValidator`, `RequestValidator`).
- **Enforce authentication and rate limiting** on every inbound request (`AuthMiddleware`, `RateLimitMiddleware`).
- **Orchestrate the full task-creation workflow** — validation → domain model construction → priority calculation → persistence → event publishing → AI recommendation request (`CreateTask` use case, `TaskOrchestrator`).
- **Orchestrate update, delete, complete, and bulk workflows** (`UpdateTask`, `DeleteTask`, `CompleteTask`, `BulkOperations` use cases).
- **Calculate and maintain task priority scores** using a weighted formula (`PriorityCalculator`).
- **Manage recurring task instances** — generate instances, handle exceptions, compute next occurrence dates (`RecurrenceManager`).
- **Enforce domain invariants** — dependency validation, circular-dependency detection, completion eligibility, overdue detection (`TaskDomainService`, `Task` domain model).
- **Abstract data persistence** behind a repository interface (`ITaskRepository`, `TaskRepository`).
- **Cache active task lists** per user to reduce database load (`TaskCache`).
- **Publish domain events** to the message bus so downstream services can react asynchronously (`EventPublisher`).

---

## 5. Internal Logic

### 5.1 `CreateTask` Use-Case Flow

```
1. TaskValidator validates incoming TaskDTO
       ↓
2. Task domain model constructed (Task entity instantiated)
       ↓
3. PriorityCalculator.calculate() derives initial Priority Score
       ↓
4. TaskRepository.save(task) persists the record
       ↓
5. EventPublisher.publish(TaskCreatedEvent)
       ↓
6. AIServiceClient.requestRecommendations(taskId)   [async, non-blocking]
       ↓
7. TaskDTO returned to caller
```

### 5.2 Priority Calculation (`PriorityCalculator`)

```
Priority Score = (Urgency × 0.4)
              + (Importance × 0.3)
              + (Dependencies × 0.2)
              + (User Preference × 0.1)

Where:
  Urgency         = f(dueDate, current_time)
  Importance      = user_defined_priority + AI_adjustment
  Dependencies    = count(tasks that block this task)
  User Preference = learned weight from historical behaviour
```

The score is recalculated on every update to `dueDate`, `priority`, or the dependency list, and stored on the `Task` entity.

### 5.3 Recurrence Management (`RecurrenceManager`)

- Stores recurrence rules using the `RecurrenceRule` domain model (RRULE-compatible).
- On each recurrence trigger, generates the next `Task` instance with a fresh `id` and an incremented `dueDate`.
- Supports exceptions (skip/reschedule a specific occurrence) without altering the root `RecurrenceRule`.
- Exposes `calculateNextOccurrence(rule: RecurrenceRule, from: DateTime): DateTime`.

### 5.4 `Task` Domain Model — Key Business Methods

| Method | Description |
|---|---|
| `complete()` | Validates prerequisites, sets `status = Completed`, raises `TaskCompletedEvent` |
| `updatePriority(newPriority)` | Applies validation, updates `priority`, raises `TaskUpdatedEvent` |
| `addDependency(taskId)` | Appends to dependency list; defers cycle-check to `TaskDomainService` |
| `canBeCompleted()` | Returns `false` if any blocking dependency is not yet completed |
| `isOverdue()` | Returns `true` if `dueDate < now` and `status != Completed` |
| `calculateUrgency()` | Computes a 0–1 urgency coefficient based on time remaining until due |

### 5.5 `TaskDomainService` — Cross-Entity Logic

- **`validateDependencies(task, dependencies[])`** — confirms all referenced dependency `id`s exist and belong to the same user.
- **`checkCircularDependencies(taskId)`** — performs a depth-first graph traversal across the dependency graph; throws `CircularDependencyException` if a cycle is found.
- **`calculateCompletionPercentage(task)`** — counts completed subtasks relative to total subtasks; factors in nested subtask depth.

### 5.6 `TaskCache` Strategy

- **Key pattern:** `task:{userId}:active`
- **TTL:** 5 minutes
- **Population:** cache is populated on the first `findByUserId` call for a user within a TTL window.
- **Invalidation:** any `save`, `update`, `delete`, or `complete` operation for that user deletes the corresponding cache key immediately (write-through invalidation).
- **Scope:** only tasks with `status != Completed` are cached; completed tasks are always fetched directly from the database.

### 5.7 `TaskRepository` — Persistence Interface

Concrete implementation (`TaskRepository`) fulfils the `ITaskRepository` interface:

| Method | Signature |
|---|---|
| `save` | `save(task: Task): Promise<Task>` |
| `findById` | `findById(id: UUID): Promise<Task>` |
| `findByUserId` | `findByUserId(userId: UUID, filters: TaskFilterDTO): Promise<Task[]>` |
| `update` | `update(task: Task): Promise<Task>` |
| `delete` | `delete(id: UUID): Promise<void>` |
| `bulkCreate` | `bulkCreate(tasks: Task[]): Promise<Task[]>` |

`TaskMapper` converts between the `Task` domain entity and the raw database row format handled by `DatabaseContext`.

---

## 6. Interactions with Other Modules

```
┌─────────────────────────────────────────────────────────────────┐
│                        TaskManager                              │
│  (API → Application → Domain → Infrastructure)                 │
└───────────┬──────────────┬───────────────┬──────────────────────┘
            │              │               │
            ▼              ▼               ▼
    AI/ML Service   Calendar Service  Notification Service
    (recommendations (time-block on    (schedule reminders
     + priority       task create /     on task create /
     adjustments)     update)           update)
            │
            ▼
      User Service
    (authentication —
     AuthMiddleware validates
     every inbound request)
            │
            ▼
      Sync Service
    (broadcasts task
     mutations to keep
     all clients in sync)
```

| Downstream Module | Direction | Purpose |
|---|---|---|
| **AI/ML Service** | TaskManager → AI/ML | Request initial priority recommendations and scheduling suggestions after a task is created; receive AI-adjusted importance scores back. |
| **Calendar Service** | TaskManager → Calendar | Create or update a time-block on the user's calendar when a task with a `dueDate` is created, updated, or rescheduled. |
| **Notification Service** | TaskManager → Notification | Register due-date and reminder notifications when a task is created or its `dueDate` changes; cancel notifications on task deletion or completion. |
| **User Service** | All layers → User Service | `AuthMiddleware` validates every inbound JWT against the User Service before any use-case executes. |
| **Sync Service** | All layers → Sync Service | `TaskCreatedEvent`, `TaskUpdatedEvent`, and `TaskCompletedEvent` are forwarded by the Sync Service to connected clients to keep real-time state consistent across devices. |
| **`ITaskRepository` / `TaskRepository`** | Application → Infrastructure | Application layer depends only on the `ITaskRepository` interface; the infrastructure `TaskRepository` provides the concrete database implementation via `DatabaseContext`. |
| **`EventPublisher` / `MessageQueue`** | Application → Infrastructure | Use-cases publish domain events through `EventPublisher`, which writes to `MessageQueue`; consuming services (Calendar, Notification, Sync) subscribe asynchronously. |
