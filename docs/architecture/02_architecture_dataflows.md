# AI Time-Management Agent - Data Flows Architecture

## Document Information

**Version:** 1.0  
**Date:** 2026-06-19  
**Status:** Draft  
**Author:** Architecture Team

---

## 1. Overview

This document defines the data flows and processing pipelines within the AI Time-Management Agent system. It illustrates how data moves through the system, transformations applied, and interactions between components.

---

## 2. Data Flow Principles

### 2.1 Core Principles

1. **Event-Driven Architecture**: Asynchronous communication via events
2. **Data Consistency**: Eventual consistency with conflict resolution
3. **Data Transformation**: Clear transformation points between layers
4. **Data Validation**: Validate at system boundaries
5. **Idempotency**: Operations can be safely retried
6. **Audit Trail**: Track all data changes

### 2.2 Data Flow Patterns

- **Request-Response**: Synchronous API calls
- **Publish-Subscribe**: Event broadcasting to multiple consumers
- **Message Queue**: Asynchronous task processing
- **Stream Processing**: Real-time data processing
- **Batch Processing**: Scheduled bulk operations

---

## 3. Primary Data Flows

### 3.1 Task Creation Flow

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ 1. POST /tasks
       │ {title, description, dueDate, priority}
       ▼
┌─────────────────────┐
│   API Gateway       │
│  - Authenticate     │
│  - Validate JWT     │
│  - Rate Limit Check │
└──────┬──────────────┘
       │ 2. Forward Request
       ▼
┌─────────────────────┐
│   Task Service      │
│  TaskController     │
└──────┬──────────────┘
       │ 3. Validate Input
       ▼
┌─────────────────────┐
│  CreateTask UseCase │
└──────┬──────────────┘
       │ 4. Create Domain Model
       ▼
┌─────────────────────┐
│  Task Domain Model  │
│  - Apply Rules      │
│  - Validate State   │
└──────┬──────────────┘
       │ 5. Persist
       ▼
┌─────────────────────┐
│  Task Repository    │
│  - Save to DB       │
│  - Generate ID      │
└──────┬──────────────┘
       │ 6. Publish Event
       ▼
┌─────────────────────┐
│   Message Queue     │
│  TaskCreatedEvent   │
└──────┬──────────────┘
       │
       ├─────────────────────────────────┐
       │                                 │
       ▼                                 ▼
┌─────────────────┐            ┌─────────────────┐
│  AI/ML Service  │            │ Calendar Service│
│  - Calculate    │            │  - Create Time  │
│    Priority     │            │    Block        │
└────────┬────────┘            └────────┬────────┘
         │                              │
         │ 7. Update Priority           │ 8. Create Event
         ▼                              ▼
┌─────────────────┐            ┌─────────────────┐
│  Task Service   │            │ Calendar Service│
│  - Update Task  │            │  - Save Event   │
└────────┬────────┘            └────────┬────────┘
         │                              │
         └──────────┬───────────────────┘
                    │ 9. Publish Updates
                    ▼
         ┌─────────────────────┐
         │   Sync Service      │
         │  - Push to Clients  │
         └──────────┬──────────┘
                    │ 10. WebSocket Push
                    ▼
         ┌─────────────────────┐
         │   Connected Clients │
         │  - Update UI        │
         └─────────────────────┘
```

**Data Transformations**:
1. **Client → API**: JSON request body
2. **API → Service**: DTO (Data Transfer Object)
3. **Service → Domain**: Domain Model
4. **Domain → Repository**: Entity/Record
5. **Repository → Database**: SQL/NoSQL query
6. **Event**: Domain Event object

**Error Handling**:
- Validation errors return 400 Bad Request
- Authentication errors return 401 Unauthorized
- Database errors trigger retry with exponential backoff
- Failed events go to dead letter queue

---

### 3.2 Calendar Sync Flow

```
┌──────────────────┐
│  Sync Scheduler  │ (Cron: every 5 minutes)
└────────┬─────────┘
         │ 1. Trigger Sync
         ▼
┌──────────────────────────┐
│  Integration Service     │
│  - Get User Integrations │
│  - Filter Active Syncs   │
└────────┬─────────────────┘
         │ 2. For Each Integration
         ▼
┌──────────────────────────┐
│  GoogleCalendarSync      │
│  - Get Sync Token        │
│  - Call Google API       │
└────────┬─────────────────┘
         │ 3. GET /calendars/{id}/events
         │    ?syncToken=xyz
         ▼
┌──────────────────────────┐
│  Google Calendar API     │
│  - Return Changed Events │
└────────┬─────────────────┘
         │ 4. Events + New Sync Token
         ▼
┌──────────────────────────┐
│  Integration Service     │
│  - Transform Events      │
│  - Map to Internal Model │
└────────┬─────────────────┘
         │ 5. Transformed Events
         ▼
┌──────────────────────────┐
│  Calendar Service        │
│  - Detect Conflicts      │
│  - Merge Changes         │
└────────┬─────────────────┘
         │ 6. Persist Changes
         ▼
┌──────────────────────────┐
│  Event Repository        │
│  - Upsert Events         │
│  - Update Sync State     │
└────────┬─────────────────┘
         │ 7. Publish Events
         ▼
┌──────────────────────────┐
│  Message Queue           │
│  CalendarSyncedEvent     │
└────────┬─────────────────┘
         │
         ├──────────────────────┐
         │                      │
         ▼                      ▼
┌─────────────────┐    ┌─────────────────┐
│  Sync Service   │    │ Notification    │
│  - Push Updates │    │  Service        │
└────────┬────────┘    │  - Send         │
         │             │    Reminders    │
         │             └─────────────────┘
         ▼
┌─────────────────┐
│  Clients        │
│  - Update UI    │
└─────────────────┘
```

**Sync Strategies**:

1. **Full Sync** (First time):
   - Fetch all events
   - Import to local database
   - Store sync token

2. **Incremental Sync** (Subsequent):
   - Use sync token
   - Fetch only changes
   - Update local database
   - Store new sync token

3. **Conflict Resolution**:
   - Compare timestamps
   - Apply resolution strategy
   - Log conflicts for review

**Data Mapping**:
```
Google Calendar Event → Internal Event
{
  id: "google_event_id",
  summary: "Meeting",
  start: {dateTime: "2026-06-20T10:00:00Z"},
  end: {dateTime: "2026-06-20T11:00:00Z"}
}
→
{
  id: "internal_uuid",
  externalId: "google_event_id",
  title: "Meeting",
  startTime: "2026-06-20T10:00:00Z",
  endTime: "2026-06-20T11:00:00Z",
  source: "google_calendar"
}
```

---

### 3.3 AI Recommendation Flow

```
┌──────────────────┐
│  Analytics       │ (Continuous)
│  Service         │
│  - Track User    │
│    Behavior      │
└────────┬─────────┘
         │ 1. Behavior Data
         ▼
┌──────────────────────────┐
│  User Behavior Profile   │
│  - Productive Hours      │
│  - Task Patterns         │
│  - Completion Rates      │
└────────┬─────────────────┘
         │ 2. Profile Updated
         ▼
┌──────────────────────────┐
│  AI/ML Service           │
│  RecommendationEngine    │
└────────┬─────────────────┘
         │ 3. Analyze Context
         │    - Current Time
         │    - Pending Tasks
         │    - Calendar Events
         │    - Workload
         ▼
┌──────────────────────────┐
│  Pattern Recognition     │
│  - Identify Patterns     │
│  - Calculate Scores      │
└────────┬─────────────────┘
         │ 4. Generate Recommendations
         ▼
┌──────────────────────────┐
│  Recommendation Rules    │
│  - Apply Business Rules  │
│  - Filter by Preferences │
│  - Rank by Relevance     │
└────────┬─────────────────┘
         │ 5. Top Recommendations
         ▼
┌──────────────────────────┐
│  Recommendation Store    │
│  - Save Recommendations  │
│  - Set Expiry            │
└────────┬─────────────────┘
         │ 6. Publish Event
         ▼
┌──────────────────────────┐
│  Message Queue           │
│  RecommendationGenerated │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│  Notification Service    │
│  - Create Notification   │
│  - Schedule Delivery     │
└────────┬─────────────────┘
         │ 7. Send Notification
         ▼
┌──────────────────────────┐
│  Client                  │
│  - Display               │
│    Recommendation        │
│  - User Feedback         │
└────────┬─────────────────┘
         │ 8. Accept/Reject
         ▼
┌──────────────────────────┐
│  AI/ML Service           │
│  - Update Model          │
│  - Learn from Feedback   │
└──────────────────────────┘
```

**Recommendation Types & Triggers**:

| Type | Trigger | Frequency |
|------|---------|-----------|
| Task Scheduling | New task created | Immediate |
| Priority Adjustment | Deadline approaching | Daily |
| Break Reminder | Continuous work detected | Real-time |
| Focus Time | Calendar analysis | Daily morning |
| Meeting Optimization | Weekly calendar review | Weekly |

**ML Pipeline**:
```
Raw Data → Feature Extraction → Model Inference → Post-Processing → Recommendation
```

---

### 3.4 Real-Time Sync Flow

```
┌──────────────────┐
│  Client A        │
│  (Web Browser)   │
└────────┬─────────┘
         │ 1. WebSocket Connection
         ▼
┌──────────────────────────┐
│  Sync Service            │
│  WebSocketManager        │
│  - Maintain Connection   │
│  - Track Active Clients  │
└────────┬─────────────────┘
         │ 2. Register Connection
         │    userId → [connections]
         ▼
┌──────────────────────────┐
│  Connection Registry     │
│  {userId: [conn1, conn2]}│
└──────────────────────────┘

         [User makes change on Client B]

┌──────────────────┐
│  Client B        │
│  (Mobile App)    │
└────────┬─────────┘
         │ 3. Update Task
         │    PUT /tasks/{id}
         ▼
┌──────────────────────────┐
│  Task Service            │
│  - Update Task           │
│  - Publish Event         │
└────────┬─────────────────┘
         │ 4. TaskUpdatedEvent
         ▼
┌──────────────────────────┐
│  Message Queue           │
└────────┬─────────────────┘
         │ 5. Event Consumed
         ▼
┌──────────────────────────┐
│  Sync Service            │
│  - Identify User         │
│  - Get Connections       │
└────────┬─────────────────┘
         │ 6. Broadcast to Connections
         ▼
┌──────────────────────────┐
│  WebSocket Push          │
│  {                       │
│    type: "task.updated", │
│    data: {...}           │
│  }                       │
└────────┬─────────────────┘
         │ 7. Receive Update
         ▼
┌──────────────────┐
│  Client A        │
│  - Update UI     │
│  - No Conflict   │
└──────────────────┘
```

**Conflict Scenario**:
```
Client A and Client B both modify same task offline

Client A                    Server                  Client B
   │                          │                        │
   │ 1. Modify Task (offline) │                        │
   │    timestamp: T1         │                        │
   │                          │  2. Modify Task        │
   │                          │     (offline)          │
   │                          │     timestamp: T2      │
   │                          │                        │
   │ 3. Come online           │                        │
   │ Push changes             │                        │
   ├─────────────────────────>│                        │
   │                          │ 4. Save (T1)           │
   │                          │                        │
   │                          │  5. Come online        │
   │                          │     Push changes       │
   │                          │<───────────────────────┤
   │                          │ 6. Detect Conflict     │
   │                          │    (T1 vs T2)          │
   │                          │                        │
   │                          │ 7. Apply Resolution    │
   │                          │    Strategy            │
   │                          │    (Last Write Wins)   │
   │                          │    Keep T2 (newer)     │
   │                          │                        │
   │ 8. Sync Update (T2)      │                        │
   │<─────────────────────────┤                        │
   │ Update UI with T2        │                        │
   │                          │ 9. Confirm Sync        │
   │                          │───────────────────────>│
   │                          │                        │
```

**Conflict Resolution Strategies**:
1. **Last Write Wins**: Use most recent timestamp
2. **Field-Level Merge**: Merge non-conflicting fields
3. **User Decision**: Prompt user to choose version
4. **Custom Rules**: Apply domain-specific logic

---

### 3.5 Natural Language Processing Flow

```
┌──────────────────┐
│  Client          │
└────────┬─────────┘
         │ 1. User Input
         │    "Schedule dentist appointment
         │     next Tuesday at 2pm"
         ▼
┌──────────────────────────┐
│  API Gateway             │
└────────┬─────────────────┘
         │ 2. POST /ai/parse-nlp
         ▼
┌──────────────────────────┐
│  AI/ML Service           │
│  NLPController           │
└────────┬─────────────────┘
         │ 3. Parse Request
         ▼
┌──────────────────────────┐
│  NLP Pipeline            │
└────────┬─────────────────┘
         │
         ├─────────────────────────────┐
         │                             │
         ▼                             ▼
┌─────────────────┐         ┌─────────────────┐
│  Tokenization   │         │  Intent         │
│  - Split words  │         │  Classification │
│  - Normalize    │         │  - Identify     │
└────────┬────────┘         │    Action       │
         │                  └────────┬────────┘
         │                           │
         ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│  Entity         │         │  Result:        │
│  Extraction     │         │  "create_event" │
│  - Date/Time    │         └─────────────────┘
│  - Title        │
│  - Priority     │
└────────┬────────┘
         │
         │ 4. Extracted Entities
         ▼
┌──────────────────────────┐
│  Entity Resolver         │
│  - "next Tuesday"        │
│    → 2026-06-24          │
│  - "2pm" → 14:00         │
└────────┬─────────────────┘
         │ 5. Structured Data
         ▼
┌──────────────────────────┐
│  Response Builder        │
│  {                       │
│    intent: "create_event"│
│    entities: {           │
│      title: "Dentist...", │
│      date: "2026-06-24", │
│      time: "14:00"       │
│    },                    │
│    confidence: 0.95      │
│  }                       │
└────────┬─────────────────┘
         │ 6. Return to Client
         ▼
┌──────────────────────────┐
│  Client                  │
│  - Display Confirmation  │
│  - User Confirms         │
└────────┬─────────────────┘
         │ 7. Create Event
         │    POST /events
         ▼
┌──────────────────────────┐
│  Calendar Service        │
│  - Create Event          │
└──────────────────────────┘
```

**NLP Processing Steps**:

1. **Tokenization**: Split text into words/tokens
2. **Intent Classification**: Identify user's intent
   - create_task
   - create_event
   - update_task
   - query_schedule
   - set_reminder

3. **Entity Extraction**: Extract key information
   - Dates: "tomorrow", "next week", "June 20"
   - Times: "2pm", "morning", "14:00"
   - Durations: "30 minutes", "2 hours"
   - Priorities: "urgent", "high priority"
   - Titles: Main subject of the request

4. **Entity Resolution**: Convert to concrete values
   - Relative dates → Absolute dates
   - Time expressions → 24-hour format
   - Fuzzy text → Structured data

5. **Confidence Scoring**: Rate parsing confidence
   - High (>0.9): Auto-execute
   - Medium (0.7-0.9): Confirm with user
   - Low (<0.7): Ask clarifying questions

---

### 3.6 Analytics Data Flow

```
┌──────────────────────────────────────────────────┐
│  User Activity (Multiple Sources)               │
├──────────────────────────────────────────────────┤
│  • Task completions                              │
│  • Time tracking entries                         │
│  • Calendar events                               │
│  • User interactions                             │
└────────┬─────────────────────────────────────────┘
         │ 1. Activity Events
         ▼
┌──────────────────────────┐
│  Event Stream            │
│  (Kafka/RabbitMQ)        │
└────────┬─────────────────┘
         │ 2. Consume Events
         ▼
┌──────────────────────────┐
│  Analytics Service       │
│  Event Processor         │
└────────┬─────────────────┘
         │ 3. Process & Aggregate
         │
         ├─────────────────────────────┐
         │                             │
         ▼                             ▼
┌─────────────────┐         ┌─────────────────┐
│  Real-Time      │         │  Batch          │
│  Processing     │         │  Processing     │
│  - Live metrics │         │  - Daily rollup │
│  - Dashboards   │         │  - Reports      │
└────────┬────────┘         └────────┬────────┘
         │                           │
         │ 4. Store Metrics          │
         ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│  Time-Series DB │         │  Analytics DB   │
│  (InfluxDB)     │         │  (PostgreSQL)   │
└────────┬────────┘         └────────┬────────┘
         │                           │
         └─────────────┬─────────────┘
                       │ 5. Query Data
                       ▼
         ┌──────────────────────────┐
         │  Analytics API           │
         │  - Productivity metrics  │
         │  - Time distribution     │
         │  - Trends                │
         └────────┬─────────────────┘
                  │ 6. Return Results
                  ▼
         ┌──────────────────────────┐
         │  Client                  │
         │  - Display Charts        │
         │  - Show Insights         │
         └──────────────────────────┘
```

**Analytics Metrics**:

**Real-Time Metrics**:
- Active users count
- Tasks completed today
- Current productivity score
- Active time tracking sessions

**Aggregated Metrics** (Daily/Weekly/Monthly):
- Total tasks completed
- Time distribution by category
- Productivity trends
- Goal progress
- Meeting time vs. focus time
- Task completion rate

**Data Aggregation Pipeline**:
```
Raw Events → Filter → Transform → Aggregate → Store → Query
```

**Example Aggregation**:
```sql
-- Daily productivity rollup
SELECT 
  user_id,
  DATE(completed_at) as date,
  COUNT(*) as tasks_completed,
  SUM(actual_duration) as total_time_minutes,
  AVG(priority_score) as avg_priority
FROM tasks
WHERE status = 'Completed'
GROUP BY user_id, DATE(completed_at)
```

---

### 3.7 Notification Delivery Flow

```
┌──────────────────────────┐
│  Trigger Sources         │
├──────────────────────────┤
│  • Task deadline         │
│  • Event reminder        │
│  • AI recommendation     │
│  • System alert          │
└────────┬─────────────────┘
         │ 1. Notification Request
         ▼
┌──────────────────────────┐
│  Notification Service    │
│  NotificationScheduler   │
└────────┬─────────────────┘
         │ 2. Check Preferences
         ▼
┌──────────────────────────┐
│  User Preferences        │
│  - Enabled channels      │
│  - Quiet hours           │
│  - DND status            │
└────────┬─────────────────┘
         │ 3. Schedule Notification
         ▼
┌──────────────────────────┐
│  Job Scheduler           │
│  (Cron/Queue)            │
└────────┬─────────────────┘
         │ 4. Trigger at Scheduled Time
         ▼
┌──────────────────────────┐
│  NotificationDispatcher  │
│  - Select Channels       │
│  - Apply Templates       │
└────────┬─────────────────┘
         │
         ├──────────────────────────────┐
         │                              │
         ▼                              ▼
┌─────────────────┐         ┌─────────────────┐
│  Push           │         │  Email          │
│  Notification   │         │  Provider       │
│  Provider       │         │  (SendGrid)     │
│  (FCM/APNs)     │         └────────┬────────┘
└────────┬────────┘                  │
         │                           │
         │ 5. Deliver                │
         ▼                           ▼
┌─────────────────┐         ┌─────────────────┐
│  Mobile Device  │         │  Email Client   │
└────────┬────────┘         └────────┬────────┘
         │                           │
         │ 6. User Interaction       │
         │    (Open/Dismiss/Action)  │
         ▼                           ▼
┌──────────────────────────────────────┐
│  Notification Service                │
│  - Track Delivery Status             │
│  - Record User Action                │
│  - Update Analytics                  │
└──────────────────────────────────────┘
```

**Notification Timing Logic**:
```
IF current_time IN quiet_hours THEN
  delay_until quiet_hours_end
ELSE IF DND_enabled THEN
  queue_for_later
ELSE IF notification_priority = HIGH THEN
  send_immediately
ELSE
  batch_with_similar_notifications
END IF
```

**Delivery Channels**:
1. **Push Notifications**: Mobile/Desktop apps
2. **Email**: Digest or immediate
3. **SMS**: High-priority only
4. **In-App**: Always available

---

### 3.8 Time Tracking Flow

```
┌──────────────────┐
│  Client          │
└────────┬─────────┘
         │ 1. Start Timer
         │    POST /time-tracking/start
         │    {taskId: "123"}
         ▼
┌──────────────────────────┐
│  Time Tracking Service   │
│  - Create Timer Session  │
│  - Record Start Time     │
└────────┬─────────────────┘
         │ 2. Timer Running
         │    (Client sends heartbeat)
         ▼
┌──────────────────────────┐
│  Active Timer Store      │
│  (Redis)                 │
│  {                       │
│    userId: "user_123",   │
│    taskId: "task_456",   │
│    startTime: "...",     │
│    lastHeartbeat: "..."  │
│  }                       │
└──────────────────────────┘

         [User works on task]

┌──────────────────┐
│  Client          │
└────────┬─────────┘
         │ 3. Stop Timer
         │    POST /time-tracking/stop
         ▼
┌──────────────────────────┐
│  Time Tracking Service   │
│  - Calculate Duration    │
│  - Create Time Entry     │
└────────┬─────────────────┘
         │ 4. Save Entry
         ▼
┌──────────────────────────┐
│  Time Entry Repository   │
│  {                       │
│    userId: "user_123",   │
│    taskId: "task_456",   │
│    startTime: "...",     │
│    endTime: "...",       │
│    duration: 3600,       │
│    category: "work"      │
│  }                       │
└────────┬─────────────────┘
         │ 5. Publish Event
         ▼
┌──────────────────────────┐
│  Message Queue           │
│  TimeEntryCreatedEvent   │
└────────┬─────────────────┘
         │
         ├──────────────────────────┐
         │                          │
         ▼                          ▼
┌─────────────────┐      ┌─────────────────┐
│  Task Service   │      │  Analytics      │
│  - Update       │      │  Service        │
│    Actual Time  │      │  - Update       │
└─────────────────┘      │    Metrics      │
                         └─────────────────┘
```

**Automatic Time Tracking**:
```
Calendar Event Starts
         │
         ▼
┌──────────────────────────┐
│  Calendar Service        │
│  - Detect Event Start    │
│  - Publish Event         │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────────┐
│  Time Tracking Service   │
│  - Auto-start Timer      │
│  - Link to Event         │
└────────┬─────────────────┘
         │
         ▼
Event Ends → Auto-stop Timer → Create Time Entry
```

---

## 4. Data Transformation Layers

### 4.1 API Layer Transformations

**Request Transformation**:
```
HTTP Request → DTO → Domain Model

Example:
{
  "title": "Complete report",
  "due_date": "2026-06-25",
  "priority": "high"
}
→ TaskDTO
→ Task Domain Model
```

**Response Transformation**:
```
Domain Model → DTO → JSON Response

Example:
Task Domain Model
→ TaskDTO
→ {
  "id": "uuid",
  "title": "Complete report",
  "dueDate": "2026-06-25T23:59:59Z",
  "priority": "high",
  "status": "not_started"
}
```

### 4.2 Integration Layer Transformations

**External → Internal**:
```
Google Calendar Event
{
  "id": "google_123",
  "summary": "Meeting",
  "start": {"dateTime": "2026-06-20T10:00:00-07:00"}
}
→ Transform
→ Internal Event
{
  "id": "uuid",
  "externalId": "google_123",
  "title": "Meeting",
  "startTime": "2026-06-20T17:00:00Z",  // Converted to UTC
  "source": "google_calendar"
}
```

**Internal → External**:
```
Internal Event → Transform → Google Calendar Event
(For pushing changes back to Google Calendar)
```

### 4.3 Event Transformations

**Domain Event → Message Queue Event**:
```
TaskCompletedEvent (Domain)
{
  taskId: UUID,
  userId: UUID,
  completedAt: DateTime,
  actualDuration: Integer
}
→ Serialize
→ Message Queue Event
{
  "eventType": "task.completed",
  "eventId": "uuid",
  "timestamp": "2026-06-19T10:30:00Z",
  "payload": {
    "taskId": "...",
    "userId": "...",
    "completedAt": "...",
    "actualDuration": 3600
  }
}
```

---

## 5. Data Consistency Patterns

### 5.1 Eventual Consistency

**Pattern**: Changes propagate asynchronously

**Example**: Task priority update
1. Task Service updates priority
2. Publishes TaskUpdatedEvent
3. AI/ML Service eventually receives event
4. Updates behavior profile
5. System reaches consistent state

**Trade-off**: Temporary inconsistency for better performance

### 5.2 Strong Consistency

**Pattern**: Synchronous updates within transaction

**Example**: Task creation with subtasks
1. Begin transaction
2. Create parent task
3. Create subtasks
4. Commit transaction
5. All or nothing

**Trade-off**: Slower but guaranteed consistency

### 5.3 Saga Pattern

**Pattern**: Distributed transaction across services

**Example**: Schedule meeting with time block
1. Calendar Service: Create event
2. Task Service: Create time block task
3. Notification Service: Schedule reminders
4. If any fails: Compensating transactions

**Compensation Flow**:
```
Success: Event → Task → Notification
Failure: Rollback Notification → Rollback Task → Rollback Event
```

---

## 6. Data Caching Strategy

### 6.1 Cache Layers

```
┌─────────────────────────────────────┐
│  Client-Side Cache                  │
│  - Recent tasks/events              │
│  - User preferences                 │
│  - TTL: Session duration            │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  API Gateway Cache                  │
│  - API responses                    │
│  - TTL: 1-5 minutes                 │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  Service-Level Cache (Redis)        │
│  - Frequently accessed data         │
│  - User sessions                    │
│  - TTL: 5-30 minutes                │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  Database                           │
│  - Source of truth                  │
└─────────────────────────────────────┘
```

### 6.2 Cache Invalidation

**Strategies**:
1. **Time-based**: Expire after TTL
2. **Event-based**: Invalidate on updates
3. **Manual**: Explicit cache clear

**Example**:
```
Task Updated
    │
    ├─> Invalidate task cache
    ├─> Invalidate user's task list cache
    └─> Publish cache invalidation event
```

---

## 7. Error Handling in Data Flows

### 7.1 Retry Strategies

**Exponential Backoff**:
```
Attempt 1: Immediate
Attempt 2: Wait 1 second
Attempt 3: Wait 2 seconds
Attempt 4: Wait 4 seconds
Attempt 5: Wait 8 seconds
Max Attempts: 5
```

**Circuit Breaker**:
```
Closed (Normal) → Open (Failing) → Half-Open (Testing) → Closed
```

### 7.2 Dead Letter Queue

**Failed Events**:
```
Event Processing Fails
    │
    ├─> Retry N times
    │
    └─> If still failing
        └─> Move to Dead Letter Queue
            └─> Manual review/reprocessing
```

---

## 8. Data Flow Monitoring

### 8.1 Metrics to Track

- **Throughput**: Events processed per second
- **Latency**: Time from event creation to processing
- **Error Rate**: Failed events percentage
- **Queue Depth**: Pending events in queue
- **Processing Time**: Time to process each event type

### 8.2 Alerting

**Alert Conditions**:
- Queue depth > threshold
- Error rate > 5%
- Processing latency > SLA
- Service unavailable

---

**Document Status:** Draft  
**Next Review Date:** TBD  
**Approval Required From:** Technical Lead, Architecture Team

---

*End of Data Flows Architecture Document*