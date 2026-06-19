# AI Time-Management Agent - Integration Architecture

## Document Information

**Version:** 1.0  
**Date:** 2026-06-19  
**Status:** Draft  
**Author:** Architecture Team

---

## 1. Overview

This document defines the external integration architecture for the AI Time-Management Agent, detailing how the system connects with third-party services, APIs, and external platforms.

---

## 2. Integration Principles

### 2.1 Core Principles

1. **Loose Coupling**: Minimize dependencies on external services
2. **Fault Tolerance**: Graceful degradation when external services fail
3. **Security First**: Secure credential storage and transmission
4. **Rate Limit Compliance**: Respect external API limits
5. **Data Privacy**: Minimize data shared with external services
6. **Standardization**: Use industry-standard protocols (OAuth 2.0, REST, webhooks)
7. **Monitoring**: Track integration health and performance

### 2.2 Integration Patterns

- **API Integration**: Direct REST/GraphQL API calls
- **OAuth 2.0**: Secure authorization for user data access
- **Webhooks**: Real-time event notifications from external services
- **Polling**: Periodic data synchronization
- **Batch Import/Export**: Bulk data transfer
- **SDK Integration**: Use official SDKs when available

---

## 3. Integration Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    AI Time-Management Agent                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │           Integration Service (Core)                    │    │
│  ├────────────────────────────────────────────────────────┤    │
│  │  • OAuth Manager                                        │    │
│  │  • Connection Manager                                   │    │
│  │  • Rate Limiter                                         │    │
│  │  • Retry Handler                                        │    │
│  │  • Data Transformer                                     │    │
│  │  • Webhook Handler                                      │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Integration Adapters (Provider-Specific)         │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │  Calendar  │  Email  │  Video  │  Task Mgmt │  Other    │  │
│  │  Adapters  │ Adapters│ Adapters│  Adapters  │ Adapters  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ HTTPS/OAuth 2.0
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    External Services                             │
├─────────────────────────────────────────────────────────────────┤
│  Google    │  Microsoft  │   Zoom   │  Slack  │  Todoist  │... │
│  Calendar  │   Outlook   │          │         │           │    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Calendar Integrations

### 4.1 Google Calendar Integration

#### 4.1.1 Authentication Flow

```
┌──────────────┐
│   User       │
└──────┬───────┘
       │ 1. Click "Connect Google Calendar"
       ▼
┌──────────────────────┐
│  Integration Service │
└──────┬───────────────┘
       │ 2. Redirect to Google OAuth
       ▼
┌──────────────────────┐
│  Google OAuth        │
│  - User authorizes   │
│  - Grants permissions│
└──────┬───────────────┘
       │ 3. Authorization code
       ▼
┌──────────────────────┐
│  Integration Service │
│  - Exchange code     │
│  - Get access token  │
│  - Get refresh token │
└──────┬───────────────┘
       │ 4. Store tokens (encrypted)
       ▼
┌──────────────────────┐
│  Token Store         │
│  {                   │
│    userId: "...",    │
│    provider: "google"│
│    accessToken: "..."│
│    refreshToken: "..."│
│    expiresAt: "..."  │
│  }                   │
└──────────────────────┘
```

**OAuth Scopes Required**:
- `https://www.googleapis.com/auth/calendar` - Full calendar access
- `https://www.googleapis.com/auth/calendar.events` - Event management

#### 4.1.2 API Operations

**List Calendars**:
```
GET https://www.googleapis.com/calendar/v3/users/me/calendarList
Authorization: Bearer {access_token}

Response:
{
  "items": [
    {
      "id": "primary",
      "summary": "My Calendar",
      "timeZone": "America/Los_Angeles"
    }
  ]
}
```

**Sync Events (Incremental)**:
```
GET https://www.googleapis.com/calendar/v3/calendars/{calendarId}/events
  ?syncToken={syncToken}
  &maxResults=250
Authorization: Bearer {access_token}

Response:
{
  "items": [...],
  "nextSyncToken": "new_sync_token"
}
```

**Create Event**:
```
POST https://www.googleapis.com/calendar/v3/calendars/{calendarId}/events
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "summary": "Team Meeting",
  "start": {
    "dateTime": "2026-06-20T10:00:00-07:00",
    "timeZone": "America/Los_Angeles"
  },
  "end": {
    "dateTime": "2026-06-20T11:00:00-07:00",
    "timeZone": "America/Los_Angeles"
  },
  "attendees": [
    {"email": "attendee@example.com"}
  ]
}
```

**Watch for Changes (Webhook)**:
```
POST https://www.googleapis.com/calendar/v3/calendars/{calendarId}/events/watch
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "id": "unique-channel-id",
  "type": "web_hook",
  "address": "https://our-app.com/webhooks/google-calendar"
}
```

#### 4.1.3 Rate Limits

- **Queries per day**: 1,000,000
- **Queries per 100 seconds per user**: 1,000
- **Queries per 100 seconds**: 10,000

**Rate Limit Handling**:
```
IF response.status == 429 THEN
  retry_after = response.headers['Retry-After']
  wait(retry_after)
  retry_request()
END IF
```

#### 4.1.4 Error Handling

| Error Code | Description | Action |
|------------|-------------|--------|
| 401 | Invalid credentials | Refresh access token |
| 403 | Insufficient permissions | Request re-authorization |
| 404 | Calendar/Event not found | Remove from local DB |
| 429 | Rate limit exceeded | Exponential backoff |
| 500 | Server error | Retry with backoff |

### 4.2 Microsoft Outlook/Office 365 Integration

#### 4.2.1 Authentication Flow

**OAuth 2.0 with Microsoft Identity Platform**:

```
Authorization Endpoint:
https://login.microsoftonline.com/common/oauth2/v2.0/authorize

Token Endpoint:
https://login.microsoftonline.com/common/oauth2/v2.0/token

Scopes:
- Calendars.ReadWrite
- Calendars.ReadWrite.Shared
- offline_access
```

#### 4.2.2 API Operations

**Microsoft Graph API Base URL**: `https://graph.microsoft.com/v1.0`

**List Calendars**:
```
GET /me/calendars
Authorization: Bearer {access_token}

Response:
{
  "value": [
    {
      "id": "calendar_id",
      "name": "Calendar",
      "color": "auto",
      "canEdit": true
    }
  ]
}
```

**Get Events**:
```
GET /me/calendars/{calendarId}/events
  ?$filter=start/dateTime ge '2026-06-01T00:00:00Z'
  &$orderby=start/dateTime
Authorization: Bearer {access_token}
```

**Create Event**:
```
POST /me/calendars/{calendarId}/events
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "subject": "Team Meeting",
  "start": {
    "dateTime": "2026-06-20T10:00:00",
    "timeZone": "Pacific Standard Time"
  },
  "end": {
    "dateTime": "2026-06-20T11:00:00",
    "timeZone": "Pacific Standard Time"
  },
  "attendees": [
    {
      "emailAddress": {
        "address": "attendee@example.com"
      },
      "type": "required"
    }
  ]
}
```

**Delta Query (Incremental Sync)**:
```
GET /me/calendarView/delta
  ?startDateTime=2026-06-01T00:00:00Z
  &endDateTime=2026-12-31T23:59:59Z
Authorization: Bearer {access_token}

Response includes deltaLink for next sync
```

**Webhooks (Change Notifications)**:
```
POST /subscriptions
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "changeType": "created,updated,deleted",
  "notificationUrl": "https://our-app.com/webhooks/outlook",
  "resource": "/me/events",
  "expirationDateTime": "2026-06-20T11:00:00.0000000Z",
  "clientState": "secret-state-token"
}
```

#### 4.2.3 Rate Limits

- **Throttling limit**: 10,000 requests per 10 minutes per app per tenant
- **Concurrent requests**: 4 per app per tenant

### 4.3 Apple Calendar (CalDAV) Integration

#### 4.3.1 CalDAV Protocol

**Connection Setup**:
```
Server: caldav.icloud.com
Port: 443 (HTTPS)
Path: /[user_id]/calendars/

Authentication: Basic Auth or App-Specific Password
```

**Discover Calendars**:
```
PROPFIND /[user_id]/calendars/ HTTP/1.1
Host: caldav.icloud.com
Depth: 1
Content-Type: application/xml

<?xml version="1.0" encoding="utf-8" ?>
<propfind xmlns="DAV:">
  <prop>
    <displayname/>
    <resourcetype/>
  </prop>
</propfind>
```

**Get Events**:
```
REPORT /[user_id]/calendars/[calendar_id]/ HTTP/1.1
Host: caldav.icloud.com
Content-Type: application/xml

<?xml version="1.0" encoding="utf-8" ?>
<calendar-query xmlns="urn:ietf:params:xml:ns:caldav">
  <prop>
    <calendar-data/>
  </prop>
  <filter>
    <comp-filter name="VCALENDAR">
      <comp-filter name="VEVENT">
        <time-range start="20260601T000000Z" end="20261231T235959Z"/>
      </comp-filter>
    </comp-filter>
  </filter>
</calendar-query>
```

**Create/Update Event**:
```
PUT /[user_id]/calendars/[calendar_id]/[event_id].ics HTTP/1.1
Host: caldav.icloud.com
Content-Type: text/calendar

BEGIN:VCALENDAR
VERSION:2.0
BEGIN:VEVENT
UID:[event_id]
DTSTART:20260620T100000Z
DTEND:20260620T110000Z
SUMMARY:Team Meeting
END:VEVENT
END:VCALENDAR
```

---

## 5. Email Integrations

### 5.1 Gmail Integration

#### 5.1.1 Use Cases

1. **Task Extraction**: Parse emails to create tasks
2. **Email Reminders**: Send task/event reminders via email
3. **Calendar Invitations**: Send meeting invites

#### 5.1.2 Gmail API Operations

**OAuth Scopes**:
- `https://www.googleapis.com/auth/gmail.readonly` - Read emails
- `https://www.googleapis.com/auth/gmail.send` - Send emails

**List Messages**:
```
GET https://gmail.googleapis.com/gmail/v1/users/me/messages
  ?q=is:unread label:inbox
  &maxResults=10
Authorization: Bearer {access_token}
```

**Get Message Content**:
```
GET https://gmail.googleapis.com/gmail/v1/users/me/messages/{messageId}
  ?format=full
Authorization: Bearer {access_token}
```

**Send Email**:
```
POST https://gmail.googleapis.com/gmail/v1/users/me/messages/send
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "raw": "base64_encoded_email_content"
}
```

#### 5.1.3 Email Parsing for Tasks

**Pattern Recognition**:
```
Subject: "TODO: Complete project report"
→ Extract: Task = "Complete project report", Priority = High

Body contains: "deadline: Friday"
→ Extract: DueDate = next Friday

Body contains: "should take about 2 hours"
→ Extract: EstimatedDuration = 2 hours
```

### 5.2 SMTP/IMAP Integration

**Generic Email Support**:

**SMTP Configuration**:
```
{
  "host": "smtp.example.com",
  "port": 587,
  "secure": true,
  "auth": {
    "user": "user@example.com",
    "pass": "encrypted_password"
  }
}
```

**IMAP Configuration**:
```
{
  "host": "imap.example.com",
  "port": 993,
  "secure": true,
  "auth": {
    "user": "user@example.com",
    "pass": "encrypted_password"
  }
}
```

---

## 6. Video Conferencing Integrations

### 6.1 Zoom Integration

#### 6.1.1 OAuth Setup

**OAuth Scopes**:
- `meeting:write` - Create meetings
- `meeting:read` - Read meeting details

#### 6.1.2 API Operations

**Create Meeting**:
```
POST https://api.zoom.us/v2/users/me/meetings
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "topic": "Team Standup",
  "type": 2,
  "start_time": "2026-06-20T10:00:00Z",
  "duration": 30,
  "timezone": "America/Los_Angeles",
  "settings": {
    "join_before_host": true,
    "mute_upon_entry": true
  }
}

Response:
{
  "id": 123456789,
  "join_url": "https://zoom.us/j/123456789",
  "password": "abc123"
}
```

**Get Meeting Details**:
```
GET https://api.zoom.us/v2/meetings/{meetingId}
Authorization: Bearer {access_token}
```

#### 6.1.3 Integration Flow

```
User creates calendar event
    │
    ▼
System detects "video meeting" requirement
    │
    ▼
Call Zoom API to create meeting
    │
    ▼
Add Zoom link to calendar event
    │
    ▼
Update event with meeting details
```

### 6.2 Google Meet Integration

**Automatic Meeting Creation**:
```
When creating Google Calendar event:

POST /calendars/{calendarId}/events
{
  "summary": "Team Meeting",
  "conferenceData": {
    "createRequest": {
      "requestId": "unique-request-id",
      "conferenceSolutionKey": {
        "type": "hangoutsMeet"
      }
    }
  }
}

Google automatically creates Meet link
```

### 6.3 Microsoft Teams Integration

**Create Online Meeting**:
```
POST https://graph.microsoft.com/v1.0/me/onlineMeetings
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "startDateTime": "2026-06-20T10:00:00Z",
  "endDateTime": "2026-06-20T11:00:00Z",
  "subject": "Team Meeting"
}

Response:
{
  "joinUrl": "https://teams.microsoft.com/l/meetup-join/...",
  "joinWebUrl": "https://teams.microsoft.com/..."
}
```

---

## 7. Task Management Integrations

### 7.1 Todoist Integration

#### 7.1.1 API Operations

**API Base URL**: `https://api.todoist.com/rest/v2`

**Authentication**: Bearer token

**Get Tasks**:
```
GET /tasks
Authorization: Bearer {api_token}

Response:
[
  {
    "id": "123",
    "content": "Complete report",
    "due": {
      "date": "2026-06-25",
      "string": "Friday"
    },
    "priority": 4
  }
]
```

**Create Task**:
```
POST /tasks
Authorization: Bearer {api_token}
Content-Type: application/json

{
  "content": "New task",
  "due_string": "tomorrow at 12:00",
  "priority": 4
}
```

**Sync API** (for real-time updates):
```
POST /sync
{
  "sync_token": "*",
  "resource_types": ["items"]
}
```

#### 7.1.2 Data Mapping

```
Todoist Task → Internal Task
{
  id: todoist_id,
  content: "Task title",
  priority: 4 (1-4 scale),
  due: {date: "2026-06-25"}
}
→
{
  id: internal_uuid,
  externalId: todoist_id,
  title: "Task title",
  priority: "high" (mapped from 4),
  dueDate: "2026-06-25T23:59:59Z",
  source: "todoist"
}
```

### 7.2 Asana Integration

#### 7.2.1 API Operations

**API Base URL**: `https://app.asana.com/api/1.0`

**Get Tasks**:
```
GET /tasks?workspace={workspace_id}&assignee=me
Authorization: Bearer {access_token}
```

**Create Task**:
```
POST /tasks
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "data": {
    "name": "New task",
    "workspace": "{workspace_id}",
    "due_on": "2026-06-25"
  }
}
```

**Webhooks**:
```
POST /webhooks
{
  "resource": "{task_id}",
  "target": "https://our-app.com/webhooks/asana"
}
```

### 7.3 Trello Integration

#### 7.3.1 API Operations

**API Base URL**: `https://api.trello.com/1`

**Get Cards (Tasks)**:
```
GET /boards/{boardId}/cards
  ?key={api_key}
  &token={api_token}
```

**Create Card**:
```
POST /cards
  ?key={api_key}
  &token={api_token}
  &idList={list_id}
  &name=Task name
  &due=2026-06-25
```

---

## 8. Communication Platform Integrations

### 8.1 Slack Integration

#### 8.1.1 Use Cases

1. **Task Notifications**: Send task reminders to Slack
2. **Status Updates**: Post productivity summaries
3. **Quick Task Creation**: Create tasks from Slack messages
4. **Meeting Reminders**: Notify about upcoming meetings

#### 8.1.2 Slack API Operations

**OAuth Scopes**:
- `chat:write` - Send messages
- `commands` - Slash commands
- `incoming-webhook` - Incoming webhooks

**Send Message**:
```
POST https://slack.com/api/chat.postMessage
Authorization: Bearer {bot_token}
Content-Type: application/json

{
  "channel": "C1234567890",
  "text": "Reminder: Team meeting in 15 minutes",
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*Team Meeting*\nStarts in 15 minutes"
      }
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": {"type": "plain_text", "text": "Join"},
          "url": "https://zoom.us/j/123456789"
        }
      ]
    }
  ]
}
```

**Slash Command** (`/task`):
```
User types: /task Complete report by Friday

Slack sends POST to our webhook:
{
  "command": "/task",
  "text": "Complete report by Friday",
  "user_id": "U1234567890",
  "channel_id": "C1234567890"
}

Our app responds:
{
  "text": "Task created: Complete report",
  "response_type": "ephemeral"
}
```

**Interactive Components**:
```
User clicks button in Slack message

Slack sends POST to our webhook:
{
  "type": "block_actions",
  "actions": [
    {
      "action_id": "complete_task",
      "value": "task_123"
    }
  ]
}

Our app processes action and responds
```

### 8.2 Microsoft Teams Integration

#### 8.2.1 Bot Integration

**Send Adaptive Card**:
```
POST https://graph.microsoft.com/v1.0/teams/{teamId}/channels/{channelId}/messages
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "body": {
    "contentType": "html",
    "content": "<attachment id=\"1\"></attachment>"
  },
  "attachments": [
    {
      "id": "1",
      "contentType": "application/vnd.microsoft.card.adaptive",
      "content": {
        "type": "AdaptiveCard",
        "body": [
          {
            "type": "TextBlock",
            "text": "Task Reminder",
            "weight": "bolder",
            "size": "medium"
          }
        ],
        "actions": [
          {
            "type": "Action.OpenUrl",
            "title": "View Task",
            "url": "https://our-app.com/tasks/123"
          }
        ]
      }
    }
  ]
}
```

---

## 9. Integration Service Architecture

### 9.1 Integration Service Components

```
┌─────────────────────────────────────────────────┐
│         Integration Service                     │
├─────────────────────────────────────────────────┤
│                                                  │
│  ┌────────────────────────────────────────┐    │
│  │  OAuth Manager                         │    │
│  │  - Token storage (encrypted)           │    │
│  │  - Token refresh                       │    │
│  │  - Authorization flows                 │    │
│  └────────────────────────────────────────┘    │
│                                                  │
│  ┌────────────────────────────────────────┐    │
│  │  Connection Manager                    │    │
│  │  - Active connections registry         │    │
│  │  - Health monitoring                   │    │
│  │  - Connection lifecycle                │    │
│  └────────────────────────────────────────┘    │
│                                                  │
│  ┌────────────────────────────────────────┐    │
│  │  Rate Limiter                          │    │
│  │  - Per-provider rate limits            │    │
│  │  - Request queuing                     │    │
│  │  - Backoff strategies                  │    │
│  └────────────────────────────────────────┘    │
│                                                  │
│  ┌────────────────────────────────────────┐    │
│  │  Retry Handler                         │    │
│  │  - Exponential backoff                 │    │
│  │  - Circuit breaker                     │    │
│  │  - Dead letter queue                   │    │
│  └────────────────────────────────────────┘    │
│                                                  │
│  ┌────────────────────────────────────────┐    │
│  │  Data Transformer                      │    │
│  │  - Format conversion                   │    │
│  │  - Field mapping                       │    │
│  │  - Validation                          │    │
│  └────────────────────────────────────────┘    │
│                                                  │
│  ┌────────────────────────────────────────┐    │
│  │  Webhook Handler                       │    │
│  │  - Signature verification              │    │
│  │  - Event processing                    │    │
│  │  - Subscription management             │    │
│  └────────────────────────────────────────┘    │
│                                                  │
└─────────────────────────────────────────────────┘
```

### 9.2 Provider Adapter Pattern

**Abstract Adapter Interface**:
```typescript
interface CalendarAdapter {
  // Authentication
  authorize(userId: string): Promise<AuthResult>
  refreshToken(userId: string): Promise<Token>
  
  // Calendar operations
  listCalendars(userId: string): Promise<Calendar[]>
  getEvents(calendarId: string, dateRange: DateRange): Promise<Event[]>
  createEvent(calendarId: string, event: Event): Promise<Event>
  updateEvent(eventId: string, event: Event): Promise<Event>
  deleteEvent(eventId: string): Promise<void>
  
  // Sync operations
  syncEvents(calendarId: string, syncToken?: string): Promise<SyncResult>
  
  // Webhook operations
  subscribeToChanges(calendarId: string, callbackUrl: string): Promise<Subscription>
  unsubscribe(subscriptionId: string): Promise<void>
}
```

**Concrete Implementations**:
- `GoogleCalendarAdapter implements CalendarAdapter`
- `OutlookCalendarAdapter implements CalendarAdapter`
- `AppleCalendarAdapter implements CalendarAdapter`

### 9.3 Token Management

**Token Storage**:
```
{
  userId: "user_123",
  provider: "google",
  accessToken: "encrypted_token",
  refreshToken: "encrypted_token",
  expiresAt: "2026-06-20T10:00:00Z",
  scopes: ["calendar", "email"],
  createdAt: "2026-06-19T10:00:00Z",
  lastRefreshedAt: "2026-06-19T10:00:00Z"
}
```

**Token Refresh Flow**:
```
Request to external API
    │
    ▼
Check token expiry
    │
    ├─> Token valid → Proceed with request
    │
    └─> Token expired
        │
        ▼
    Refresh token
        │
        ├─> Success → Update stored token → Retry request
        │
        └─> Failure → Mark integration as disconnected
                   → Notify user to re-authorize
```

### 9.4 Rate Limiting Strategy

**Per-Provider Rate Limits**:
```typescript
const rateLimits = {
  google_calendar: {
    requestsPerSecond: 10,
    requestsPerDay: 1000000,
    burstSize: 100
  },
  outlook: {
    requestsPerMinute: 100,
    concurrentRequests: 4
  },
  zoom: {
    requestsPerSecond: 10,
    dailyLimit: 100000
  }
}
```

**Token Bucket Algorithm**:
```
class RateLimiter {
  private tokens: number
  private lastRefill: Date
  
  async acquireToken(provider: string): Promise<void> {
    const limit = rateLimits[provider]
    
    // Refill tokens based on time elapsed
    this.refillTokens(limit.requestsPerSecond)
    
    if (this.tokens > 0) {
      this.tokens--
      return
    }
    
    // Wait until tokens available
    await this.waitForToken(limit)
  }
}
```

### 9.5 Error Handling and Retry

**Retry Strategy**:
```typescript
async function retryWithBackoff<T>(
  operation: () => Promise<T>,
  maxRetries: number = 3
): Promise<T> {
  let lastError: Error
  
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await operation()
    } catch (error) {
      lastError = error
      
      if (!isRetryable(error)) {
        throw error
      }
      
      const delay = Math.pow(2, attempt) * 1000 // Exponential backoff
      await sleep(delay)
    }
  }
  
  throw lastError
}

function isRetryable(error: Error): boolean {
  const retryableStatusCodes = [408, 429, 500, 502, 503, 504]
  return retryableStatusCodes.includes(error.statusCode)
}
```

**Circuit Breaker**:
```typescript
class CircuitBreaker {
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED'
  private failureCount: number = 0
  private lastFailureTime: Date
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (this.shouldAttemptReset()) {
        this.state = 'HALF_OPEN'
      } else {
        throw new Error('Circuit breaker is OPEN')
      }
    }
    
    try {
      const result = await operation()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }
  
  private onSuccess(): void {
    this.failureCount = 0
    this.state = 'CLOSED'
  }
  
  private onFailure(): void {
    this.failureCount++
    this.lastFailureTime = new Date()
    
    if (this.failureCount >= 5) {
      this.state = 'OPEN'
    }
  }
}
```

---

## 10. Webhook Management

### 10.1 Webhook Security

**Signature Verification**:
```typescript
function verifyWebhookSignature(
  payload: string,
  signature: string,
  secret: string
): boolean {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex')
  
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  )
}
```

**Webhook Handler**:
```typescript
app.post('/webhooks/:provider', async (req, res) => {
  const provider = req.params.provider
  const signature = req.headers['x-signature']
  const payload = JSON.stringify(req.body)
  
  // Verify signature
  if (!verifyWebhookSignature(payload, signature, webhookSecret)) {
    return res.status(401).send('Invalid signature')
  }
  
  // Acknowledge receipt immediately
  res.status(200).send('OK')
  
  // Process webhook asynchronously
  await processWebhook(provider, req.body)
})
```

### 10.2 Webhook Subscription Management

**Subscription Lifecycle**:
```
Create Subscription
    │
    ▼
Store subscription details
    │
    ▼
Monitor subscription health
    │
    ├─> Active → Continue receiving events
    │
    ├─> Expiring → Renew subscription
    │
    └─> Failed → Retry creation or fallback to polling
```

**Subscription Renewal**:
```typescript
async function renewWebhookSubscriptions(): Promise<void> {
  const expiringSubscriptions = await getExpiringSubscriptions()
  
  for (const subscription of expiringSubscriptions) {
    try {
      await renewSubscription(subscription)
    } catch (error) {
      // Fallback to polling if renewal fails
      await enablePollingForUser(subscription.userId)
    }
  }
}

// Run every hour
schedule('0 * * * *', renewWebhookSubscriptions)
```

---

## 11. Integration Monitoring

### 11.1 Health Checks

**Integration Health Metrics**:
```typescript
interface IntegrationHealth {
  provider: string
  status: 'healthy' | 'degraded' | 'down'
  lastSuccessfulSync: Date
  errorRate: number
  averageLatency: number
  activeConnections: number
}
```

**Health Check Endpoint**:
```
GET /integrations/health

Response:
{
  "google_calendar": {
    "status": "healthy",
    "lastCheck": "2026-06-19T10:30:00Z",
    "errorRate": 0.01,
    "averageLatency": 250
  },
  "outlook": {
    "status": "degraded",
    "lastCheck": "2026-06-19T10:30:00Z",
    "errorRate": 0.15,
    "averageLatency": 1500
  }
}
```

### 11.2 Monitoring Metrics

**Key Metrics**:
- API call success/failure rate
- API response time
- Token refresh success rate
- Webhook delivery success rate
- Sync latency
- Rate limit hits
- Circuit breaker state changes

**Alerting Rules**:
```
Alert: Integration Down
Condition: error_rate > 50% for 5 minutes
Action: Notify ops team, disable integration

Alert: High Latency
Condition: avg_latency > 5 seconds for 10 minutes
Action: Investigate, consider caching

Alert: Rate Limit Approaching
Condition: requests > 80% of limit
Action: Throttle requests, notify team
```

---

## 12. Data Privacy and Compliance

### 12.1 Data Minimization

**Principles**:
- Only request necessary OAuth scopes
- Store minimal data from external services
- Regularly purge unused integration data
- Allow users to disconnect integrations

### 12.2 Data Encryption

**At Rest**:
- Encrypt OAuth tokens using AES-256
- Encrypt sensitive user data
- Use separate encryption keys per user

**In Transit**:
- All API calls over HTTPS/TLS 1.3
- Certificate pinning for mobile apps

### 12.3 Compliance

**GDPR Compliance**:
- User consent for each integration
- Right to disconnect and delete data
- Data portability (export integration data)
- Privacy policy disclosure

**OAuth Best Practices**:
- Use PKCE for mobile apps
- Implement state parameter for CSRF protection
- Rotate refresh tokens
- Revoke tokens on disconnect

---

## 13. Integration Testing

### 13.1 Test Strategy

**Unit Tests**:
- Test adapter implementations
- Test data transformations
- Test error handling

**Integration Tests**:
- Test OAuth flows (using test accounts)
- Test API operations
- Test webhook handling

**End-to-End Tests**:
- Test complete sync workflows
- Test conflict resolution
- Test error recovery

### 13.2 Mock Services

**Development Environment**:
```typescript
class MockGoogleCalendarAdapter implements CalendarAdapter {
  async getEvents(calendarId: string): Promise<Event[]> {
    return [
      {
        id: 'mock_event_1',
        title: 'Mock Meeting',
        startTime: new Date(),
        endTime: new Date()
      }
    ]
  }
  
  // ... other mock implementations
}
```

---

**Document Status:** Draft  
**Next Review Date:** TBD  
**Approval Required From:** Technical Lead, Integration Team

---

*End of Integration Architecture Document*