# Event

## 1. Model Purpose

The Event model represents calendar events created and managed by users. It serves as the central data structure for storing calendar entries with temporal data, managing attendees and RSVP state, setting multi-channel reminders, and synchronising bidirectionally with external calendar providers (Google Calendar, Outlook, Apple Calendar). Events also link to ScheduleBlocks for time-blocking and feed into AnalyticsRecord for meeting-time tracking.

## 2. Fields and Types

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique identifier for the event |
| `userId` | UUID | FK to UserProfile; the event owner |
| `calendarId` | UUID | FK to Calendar; the container calendar |
| `title` | String | Event name (1–500 characters) |
| `description` | String | Detailed event information |
| `location` | String | Physical address or virtual meeting URL |
| `startTime` | DateTime | Event start time (stored in UTC) |
| `endTime` | DateTime | Event end time (stored in UTC) |
| `isAllDay` | Boolean | If `true`, startTime and endTime are date-only (00:00:00 UTC) |
| `recurrenceRule` | RecurrenceRule | Optional; defines recurring event pattern (RRULE format) |
| `attendees` | Array\<Attendee\> | Each Attendee: `{ userId, email, status: Enum(Accepted, Declined, Tentative) }` |
| `reminders` | Array\<Reminder\> | Each Reminder: `{ minutesBefore: Integer, channel: Enum(Push, Email, SMS) }` |
| `status` | Enum | Event state: `Confirmed`, `Tentative`, `Cancelled` |
| `visibility` | Enum | `Public` or `Private` |
| `color` | String | Hex colour code (e.g. `#2979F2`) |
| `externalId` | String | ID in the external calendar provider (e.g. Google event ID) |
| `source` | String | Origin: `"google_calendar"`, `"outlook"`, `"internal"` |
| `createdAt` | DateTime | Timestamp of record creation (UTC) |
| `updatedAt` | DateTime | Timestamp of last modification (UTC) |

### Calendar Sub-Model

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique calendar identifier |
| `userId` | UUID | FK to UserProfile |
| `name` | String | Display name |
| `type` | Enum | `Primary`, `Work`, `Personal`, `Shared` |
| `color` | String | Hex colour code |
| `isDefault` | Boolean | Whether this is the user's default calendar |
| `syncEnabled` | Boolean | Whether external sync is active |
| `externalProvider` | String | Provider name (e.g. `"google"`, `"outlook"`) |
| `externalId` | String | Calendar ID at the external provider |

## 3. Constraints

- `title`: Required; 1–500 characters
- `startTime` must be before `endTime`
- If `isAllDay` is `true`, both `startTime` and `endTime` must be date-only (time part = `00:00:00 UTC`)
- All DateTime fields stored in UTC; clients convert to user's local timezone for display
- `externalId` is unique per `source` per user (composite unique constraint on `userId + source + externalId`)
- `status` must be one of: `Confirmed`, `Tentative`, `Cancelled`

## 4. Relationships

- **belongs to** UserProfile (`userId`)
- **belongs to** Calendar (`calendarId`)
- **linked to** ScheduleBlock (`eventId`; optional — set when the event represents a time block)
- **linked to** AnalyticsRecord (`eventId`; meeting time entries reference this event)
- **synced with** external providers via Integration (bidirectional, keyed on `externalId + source`)

## 5. Usage Examples

### Example 1: Recurring Team Standup with Attendees

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "550e8400-e29b-41d4-a716-446655440001",
  "calendarId": "550e8400-e29b-41d4-a716-446655440002",
  "title": "Engineering Team Standup",
  "description": "Daily 15-minute sync on sprint progress",
  "location": "Zoom: https://zoom.us/j/123456789",
  "startTime": "2026-06-20T09:00:00Z",
  "endTime": "2026-06-20T09:15:00Z",
  "isAllDay": false,
  "recurrenceRule": "FREQ=DAILY;BYDAY=MO,TU,WE,TH,FR;UNTIL=20261231T090000Z",
  "attendees": [
    { "userId": "user-001", "email": "alice@company.com", "status": "Accepted" },
    { "userId": "user-002", "email": "bob@company.com",   "status": "Accepted" },
    { "userId": "user-003", "email": "carol@company.com", "status": "Tentative" }
  ],
  "reminders": [
    { "minutesBefore": 15, "channel": "Push" },
    { "minutesBefore": 1,  "channel": "Email" }
  ],
  "status": "Confirmed",
  "visibility": "Private",
  "color": "#2979F2",
  "externalId": null,
  "source": "internal",
  "createdAt": "2026-06-01T14:30:00Z",
  "updatedAt": "2026-06-01T14:30:00Z"
}
```

### Example 2: Synced Google Calendar Event

```json
{
  "id": "660e8400-e29b-41d4-a716-446655550000",
  "userId": "550e8400-e29b-41d4-a716-446655440001",
  "calendarId": "550e8400-e29b-41d4-a716-446655440005",
  "title": "Q3 Planning Session",
  "description": "Strategic roadmap review with stakeholders",
  "location": "Conference Room A / Hybrid",
  "startTime": "2026-06-25T10:00:00Z",
  "endTime": "2026-06-25T11:30:00Z",
  "isAllDay": false,
  "recurrenceRule": null,
  "attendees": [
    { "userId": "user-001", "email": "alice@company.com", "status": "Accepted" },
    { "userId": "user-004", "email": "diana@company.com", "status": "Accepted" }
  ],
  "reminders": [
    { "minutesBefore": 30, "channel": "Push" }
  ],
  "status": "Confirmed",
  "visibility": "Public",
  "color": "#D32F2F",
  "externalId": "1a2b3c4d5e6f7g8h9i0j@google.com",
  "source": "google_calendar",
  "createdAt": "2026-06-18T08:00:00Z",
  "updatedAt": "2026-06-19T16:45:00Z"
}
```

## 6. Notes for Future Extensions

- AI-suggested optimal meeting durations based on historical patterns
- Smart conflict detection with travel-time and buffer-time awareness
- Video conferencing link auto-generation (Zoom, Google Meet, Microsoft Teams)
- Attendee availability scoring across multiple time zones
- Event templates for common meeting types (1:1, standup, planning, retrospective)
