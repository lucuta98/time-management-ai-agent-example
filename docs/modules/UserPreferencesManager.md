# UserPreferencesManager

## 1. Purpose

The `UserPreferencesManager` (implemented as `PreferencesService` within the User Service Application Layer) stores, validates, and serves all user-configurable settings that personalise the agent's behaviour.

It is the single authoritative source of truth for every preference record in the system. Any AI/ML subsystem, notification pipeline, or scheduling component that needs to adapt its behaviour to a specific user reads from this module. Any surface — REST API, onboarding flow, or profile editor — that allows a user to change their settings writes through it.

---

## 2. Inputs

### API / Use-Case Requests

| Source | Operation | Payload |
|---|---|---|
| `PreferencesController` (REST API) | Read preferences | `userId` (from JWT) |
| `PreferencesController` (REST API) | Write / patch preferences | `userId` + partial `UserPreferences` body |
| `ManagePreferences` use case | Full CRUD on preference records | `userId` + `PreferencesPatch` |
| `UpdateProfile` use case | Partial preference update (e.g., locale change) | `userId` + profile fields that overlap with preferences |
| `RegisterUser` use case | Seed defaults on new account | `userId` |

### Authentication Context

- Every read and write arrives with a verified JWT, validated upstream by `JWTMiddleware` and `AuthService`.
- `TokenService.verifyToken(token)` extracts the `userId` claim; no preference mutation is performed before identity is confirmed.

### Preference Payload Fields

```json
{
  "workingHours": { "start": "09:00", "end": "18:00", "days": ["Mon","Tue","Wed","Thu","Fri"] },
  "notifications": {
    "channels": ["push", "email"],
    "quietHoursStart": "22:00",
    "quietHoursEnd": "07:00",
    "dndEnabled": false
  },
  "ui": { "theme": "dark", "language": "en-US" },
  "calendar": { "defaultCalendarId": "primary", "defaultEventDuration": 30, "timezone": "America/New_York" },
  "privacy": { "dataSharing": false, "analyticsOptIn": true },
  "integrations": { "connected": ["google", "slack"] }
}
```

---

## 3. Outputs

### Returned Data Structures

- **`UserPreferences`** — the full domain model object, returned on any read operation; consumed by downstream services.
- **`PreferencesPatch` acknowledgement** — confirmation (including a new `version` number) returned after a successful write.

### Side Effects

- Persisted records written to `UserRepository` (infrastructure layer) on every successful mutation.
- Version counter incremented on each write to support conflict resolution during multi-device sync.
- Change event emitted to `ChangeTracker` (Sync Service) so that updated preferences are propagated to all connected devices.

### Consumed by Callers

| Consumer | Preference fields used |
|---|---|
| `SchedulingEngine` | `workingHours`, `calendar.timezone`, break/focus-time preferences |
| `PrioritizationEngine` | User-defined priority weights (from `calendar` / custom weight fields) |
| `AgentBehaviorController` | Full preferences object for filtering and personalising recommendations |
| `NLU` | `calendar.timezone` for resolving relative date/time expressions |
| `NotificationScheduler` | `notifications.channels`, `quietHoursStart/End`, `dndEnabled` |
| `NotificationDispatcher` | `notifications.channels` for delivery routing |

---

## 4. Responsibilities

- **CRUD operations** on preference records for any authenticated user via the `ManagePreferences` use case.
- **Default seeding** — populate a complete `UserPreferences` record with safe defaults whenever `RegisterUser` creates a new account.
- **Partial updates** — accept sparse patch objects and merge them into the existing record without overwriting unmentioned fields.
- **Input validation** — reject writes that contain invalid data, including:
  - Non-IANA timezone strings (e.g. `calendar.timezone`)
  - Unrecognised locale / language codes (e.g. `ui.language`)
  - `workingHours.start` ≥ `workingHours.end`
  - Unknown integration identifiers
- **Version tracking** — increment the record's `version` field on every successful mutation to enable optimistic-locking and multi-device conflict resolution.
- **Auth enforcement** — reject any read or write request that does not carry a verified `userId` matching the target preference record.
- **Serving read requests** — return the current `UserPreferences` object efficiently to all internal consumers (AI/ML pipeline, notification pipeline, scheduling engine).

---

## 5. Internal Logic

### Layer Placement

`PreferencesService` lives in the **Application Layer** of the User Service, between the `PreferencesController` (Presentation Layer) and the `UserPreferences` domain model / `UserRepository` (Domain + Infrastructure Layers).

```
PreferencesController  (API Layer)
        │
        ▼
PreferencesService     (Application Layer)  ◄── this module
        │
        ├──► ManagePreferences (use case)
        │
        ├──► UserPreferences   (Domain Model — validates business rules)
        │
        └──► UserRepository    (Infrastructure — persists to DB)
```

### Read Path

1. Receive `userId` extracted from the validated JWT.
2. Call `UserRepository.findPreferencesByUserId(userId)`.
3. Return the hydrated `UserPreferences` object to the caller.

### Write Path

1. Receive `userId` and a `PreferencesPatch` payload.
2. Load existing `UserPreferences` from `UserRepository`.
3. Merge the patch into the existing record (non-destructive merge).
4. Run domain-level validation on the merged object (timezone check, locale check, hour ordering, etc.).
5. Increment `version` counter.
6. Persist via `UserRepository.save(userPreferences)`.
7. Emit a change event to `ChangeTracker` for cross-device sync.
8. Return the updated `UserPreferences` with the new `version` to the caller.

### Default Seeding (on `RegisterUser`)

- Called once per new user immediately after account creation.
- Writes a `UserPreferences` record pre-populated with system defaults:
  - Working hours: Mon–Fri, 09:00–18:00
  - Notifications: push channel enabled, no quiet hours
  - UI: system theme, `en-US` locale
  - Calendar: 30-minute default event duration, UTC timezone
  - Privacy: analytics opt-in `false`, data sharing `false`
  - Integrations: no connected accounts

### Validation Rules (enforced by `UserPreferences` domain model)

| Field | Rule |
|---|---|
| `calendar.timezone` | Must be a valid IANA timezone string (e.g. `"America/New_York"`) |
| `ui.language` | Must be a valid BCP-47 locale code (e.g. `"en-US"`, `"fr-FR"`) |
| `workingHours.start` / `.end` | `start` must be strictly before `end` |
| `notifications.channels` | Must be a non-empty subset of `["push", "email", "sms"]` |
| `integrations.connected` | Each entry must match a known integration identifier |

### Conflict Resolution

- The `version` field is returned to every writer and must be included in subsequent patch requests.
- If the incoming `version` does not match the stored `version`, the write is rejected with a `409 Conflict` response, prompting the client to re-fetch and re-apply.

---

## 6. Interactions with Other Modules

### Inbound (modules that write to `UserPreferencesManager`)

| Module / Entry Point | Interaction |
|---|---|
| `PreferencesController` | Exposes REST endpoints; forwards reads and writes to `PreferencesService` |
| `ManagePreferences` use case | Orchestrates full CRUD; delegates validation and persistence to `PreferencesService` |
| `UpdateProfile` use case | Passes profile changes that overlap with preference fields (e.g., display name, locale) |
| `RegisterUser` use case | Triggers default preference seeding on new account creation |

### Outbound (modules that read from `UserPreferencesManager`)

| Module | Purpose of read |
|---|---|
| `SchedulingEngine` | Enforces working-hours windows, break preferences, and focus-time blocks when assigning tasks to calendar slots |
| `PrioritizationEngine` | Applies user-defined priority weights when computing task priority scores |
| `AgentBehaviorController` | Filters and personalises AI recommendations based on the full preferences profile |
| `NLU` | Resolves the user's timezone when parsing relative date/time expressions (e.g. "tomorrow at 9") |
| `NotificationScheduler` | Reads quiet-hours window and DND flag to schedule notification delivery appropriately |
| `NotificationDispatcher` | Reads enabled channels to route notifications via push, email, or SMS |

### Infrastructure Dependencies

| Component | Role |
|---|---|
| `UserRepository` | Persistence layer — reads and writes `UserPreferences` records |
| `AuthService` / `JWTMiddleware` | Authenticate every inbound request before any preference access |
| `TokenService` | Verifies user identity (`verifyToken`) before any preference mutation |
| `ChangeTracker` (Sync Service) | Receives change events after every successful preference write for cross-device synchronisation |
