# Preferences

## 1. Model Purpose

The Preferences model stores user-specific configuration and behavioural settings that control how the time management AI agent behaves, schedules, and communicates. It drives working-hour enforcement in the SchedulingEngine, notification delivery in NotificationService, recommendation filtering in AgentBehaviorController, and the UI personalisation layer. Each user has exactly one Preferences record (1:1 with UserProfile).

## 2. Fields and Types

| Field | Type | Description |
|---|---|---|
| `userId` | UUID | PK and FK to UserProfile; one record per user |
| `workingHours` | TimeRange | `{ start: "HH:MM", end: "HH:MM" }` — daily working window |
| `workingDays` | Array\<DayOfWeek\> | Days the user works, e.g. `["Monday","Tuesday","Wednesday","Thursday","Friday"]` |
| `defaultCalendar` | UUID | FK to Calendar; used by default when creating events |
| `notificationSettings` | Object | Nested NotificationPreferences (see below) |
| `notificationSettings.enabledChannels` | Array\<Enum\> | Active delivery channels: `Push`, `Email`, `SMS` |
| `notificationSettings.quietHours` | TimeRange | `{ start: "HH:MM", end: "HH:MM" }` — no notifications during this window |
| `notificationSettings.notificationFrequency` | Enum | `Immediate`, `Batched`, or `Daily` |
| `notificationSettings.categoryPreferences` | Map\<Category, Boolean\> | Per-category opt-in/out (e.g. `{ "TaskDeadline": true, "DigestSummary": false }`) |
| `themePreference` | Enum | UI theme: `Light`, `Dark`, `Auto` |
| `language` | String | BCP 47 language code (e.g. `"en-US"`, `"de-DE"`) |
| `dateFormat` | String | Display format (e.g. `"YYYY-MM-DD"`, `"MM/DD/YYYY"`, `"DD/MM/YYYY"`) |
| `timeFormat` | Enum | `12h` or `24h` |
| `weekStartDay` | DayOfWeek | First day of week in calendar views (`"Monday"` or `"Sunday"`) |
| `privacySettings` | Object | e.g. `{ shareCalendar: false, analyticsOptIn: true }` |
| `integrationSettings` | Object | Provider-specific config per integration |
| `updatedAt` | DateTime | Timestamp of last modification (UTC) |

## 3. Constraints

- `userId`: Unique (one record per user), required, FK to UserProfile
- `workingHours.start` must be before `workingHours.end`
- `workingDays`: Non-empty array; valid values are Monday–Sunday
- `language`: Must be a valid BCP 47 language code
- `timeFormat`: Must be exactly `"12h"` or `"24h"`
- `notificationSettings.quietHours.start` must be before `end` when both are specified

## 4. Relationships

- **belongs to** UserProfile (`userId` — 1:1)
- **consulted by** SchedulingEngine to enforce working-hour and working-day hard constraints
- **consulted by** PrioritizationEngine for user-defined priority weight
- **consulted by** AgentBehaviorController to filter and rank recommendations
- **controls** NotificationService delivery channels, quiet hours, and per-category rules

## 5. Usage Examples

### Example 1: Default US-Based User (9–5 Mon–Fri, English, Dark Theme)

```json
{
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "workingHours": { "start": "09:00", "end": "17:00" },
  "workingDays": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
  "defaultCalendar": "660e8400-e29b-41d4-a716-446655440001",
  "notificationSettings": {
    "enabledChannels": ["Push", "Email"],
    "quietHours": { "start": "18:00", "end": "08:00" },
    "notificationFrequency": "Immediate",
    "categoryPreferences": {
      "TaskDeadline": true,
      "MeetingReminder": true,
      "DigestSummary": false
    }
  },
  "themePreference": "Dark",
  "language": "en-US",
  "dateFormat": "MM/DD/YYYY",
  "timeFormat": "12h",
  "weekStartDay": "Sunday",
  "privacySettings": {
    "shareCalendar": false,
    "analyticsOptIn": true
  },
  "integrationSettings": {
    "googleCalendar": { "enabled": true, "syncFrequency": "realtime" }
  },
  "updatedAt": "2026-06-19T10:30:00Z"
}
```

### Example 2: European User with Quiet Hours and Batched Notifications

```json
{
  "userId": "770e8400-e29b-41d4-a716-446655440002",
  "workingHours": { "start": "08:30", "end": "18:00" },
  "workingDays": ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"],
  "defaultCalendar": "880e8400-e29b-41d4-a716-446655440003",
  "notificationSettings": {
    "enabledChannels": ["Email"],
    "quietHours": { "start": "19:00", "end": "07:00" },
    "notificationFrequency": "Batched",
    "categoryPreferences": {
      "TaskDeadline": true,
      "MeetingReminder": true,
      "DigestSummary": true,
      "LowPriorityUpdate": false
    }
  },
  "themePreference": "Light",
  "language": "de-DE",
  "dateFormat": "DD/MM/YYYY",
  "timeFormat": "24h",
  "weekStartDay": "Monday",
  "privacySettings": {
    "shareCalendar": true,
    "analyticsOptIn": false
  },
  "integrationSettings": {
    "microsoftOutlook": { "enabled": true, "syncFrequency": "hourly" },
    "slack": { "enabled": true, "statusSync": true }
  },
  "updatedAt": "2026-06-19T14:45:00Z"
}
```

## 6. Notes for Future Extensions

- Per-calendar colour and visibility overrides
- AI-learned preferences (auto-detected productive hours surfaced as suggestions)
- Context-sensitive preference profiles (Work mode vs. Personal mode)
- Preference sync across devices with conflict resolution
- Granular notification rules per task category or project
