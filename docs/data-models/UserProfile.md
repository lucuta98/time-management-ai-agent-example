# UserProfile

## 1. Model Purpose

The UserProfile model represents a user account within the time management AI agent system. It maintains essential user identity information, authentication credentials, personalisation settings, access control roles, and service quotas. UserProfile serves as the central entity linking all user-generated data (tasks, events, analytics, schedules, etc.) and manages authentication state through JWT tokens issued by AuthService and TokenService.

## 2. Fields and Types

| Field | Type | Description |
|---|---|---|
| `id` | UUID | Unique user identifier; primary key |
| `email` | String | User email address; unique, RFC 5322 compliant |
| `passwordHash` | String | bcrypt hash; never returned in any API response |
| `firstName` | String | User's given name |
| `lastName` | String | User's surname |
| `timezone` | String | IANA timezone string (e.g. `"America/New_York"`, `"Europe/London"`) |
| `locale` | String | BCP 47 locale string (e.g. `"en-US"`, `"de-DE"`) |
| `avatarUrl` | String | URL to the user's profile image; nullable |
| `status` | Enum | Account state: `Active`, `Inactive`, `Suspended` |
| `emailVerified` | Boolean | `false` by default; set to `true` after email confirmation |
| `mfaEnabled` | Boolean | Whether multi-factor authentication is required at login |
| `role` | Enum | Access level: `User`, `Admin`, `TeamLead` |
| `features` | Array\<String\> | Feature flags and entitlements active for this account |
| `quotas` | Object | Usage limits (e.g. `{ maxTasks: 1000, maxCalendars: 10 }`) |
| `createdAt` | DateTime | Account creation timestamp (UTC) |
| `lastLoginAt` | DateTime | Most recent successful login timestamp (UTC) |

### Associated JWT Token Structure

```json
{
  "sub": "user_id",
  "email": "user@example.com",
  "role": "User",
  "iat": 1234567890,
  "exp": 1234568790
}
```

Access token TTL: 15 minutes. Refresh token TTL: 30 days.

## 3. Constraints

- `email`: Unique, valid RFC 5322 format, required
- `passwordHash`: bcrypt-generated; must never be exposed in API responses
- `timezone`: Must be a valid IANA timezone identifier
- `status`: Defaults to `Active` on registration; transitions to `Inactive` or `Suspended` only via admin action
- `emailVerified`: Defaults to `false`; set to `true` only after the email verification flow completes
- `role`: Defaults to `User` on registration
- `mfaEnabled`: When `true`, login requires a second factor before a JWT is issued

## 4. Relationships

- **has one** Preferences (`userId` — 1:1)
- **has many** Task (`userId`)
- **has many** Event (`userId`)
- **has many** AnalyticsRecord (`userId`)
- **has many** ContextSnapshot (`userId`)
- **has many** ScheduleBlock (`userId`)
- **authenticated via** JWT managed by AuthService / TokenService

## 5. Usage Examples

### Example 1: Newly Registered User

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440001",
  "email": "alice@example.com",
  "passwordHash": "$2b$12$R9h7cIPz0gi.URNNX3kh2OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUm",
  "firstName": "Alice",
  "lastName": "Johnson",
  "timezone": "America/New_York",
  "locale": "en-US",
  "avatarUrl": null,
  "status": "Active",
  "emailVerified": false,
  "mfaEnabled": false,
  "role": "User",
  "features": ["basic-tasks", "basic-events"],
  "quotas": {
    "maxTasks": 1000,
    "maxCalendars": 10
  },
  "createdAt": "2026-06-19T10:30:00Z",
  "lastLoginAt": "2026-06-19T10:30:00Z"
}
```

### Example 2: Active Admin User with MFA Enabled

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440002",
  "email": "admin@company.com",
  "passwordHash": "$2b$12$Q1z9kGPq1hj.VSMMX2km3OPST9/PgBkqquzi.Ss7KIUgO2t0jWMUm",
  "firstName": "Bob",
  "lastName": "Smith",
  "timezone": "Europe/London",
  "locale": "en-GB",
  "avatarUrl": "https://cdn.example.com/avatars/bob-smith.jpg",
  "status": "Active",
  "emailVerified": true,
  "mfaEnabled": true,
  "role": "Admin",
  "features": ["basic-tasks", "basic-events", "advanced-analytics", "team-management", "admin-panel"],
  "quotas": {
    "maxTasks": 5000,
    "maxCalendars": 50,
    "maxTeamMembers": 100
  },
  "createdAt": "2025-01-15T08:00:00Z",
  "lastLoginAt": "2026-06-19T14:22:35Z"
}
```

## 6. Notes for Future Extensions

- OAuth / SSO provider linking (Google, Microsoft, GitHub)
- Team and organisation membership with role-based delegated permissions
- Profile completeness scoring to guide onboarding
- GDPR data export (Article 20) and right-to-erasure (Article 17) endpoints
- Passkey / WebAuthn support for passwordless authentication
- TOTP, SMS, and hardware security key options for MFA
