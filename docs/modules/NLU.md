# NLU

## 1. Purpose

The NLU (Natural Language Understanding) module parses free-text user input into
structured, actionable task and event data that downstream modules can process without
any further interpretation of raw language.

It acts as the linguistic entry point of the AI agent: every user utterance — whether
typed into a chat interface or passed via the API — passes through this module before
any scheduling, task creation, or calendar action is taken.

---

## 2. Inputs

- **Raw text string** — the user's free-text utterance, received from the API Gateway
  or client application (e.g. `"Schedule dentist appointment next Tuesday at 2pm"`).
- **User timezone** — retrieved from `UserPreferencesManager`; used by the
  `DateTimeParser` sub-module to resolve relative date/time expressions (e.g.
  `"tomorrow"`, `"next week"`) into precise, timezone-correct timestamps.

---

## 3. Outputs

A single **`NLPResult`** object with the following shape:

```typescript
interface NLPResult {
  intent:     Intent;          // classified user intent (enum value)
  entities: {
    title?:    string;         // task or event title
    date?:     string;         // resolved date expression
    time?:     string;         // resolved time expression
    duration?: string;         // e.g. "1 hour"
    priority?: string;         // e.g. "High"
    person?:   string;         // e.g. "John"
    location?: string;         // e.g. "Conference Room B"
  };
  confidence: number;          // 0–1 composite confidence score
}
```

The `NLPResult` is handed directly to `AgentBehaviorController` for routing and
downstream action.

---

## 4. Responsibilities

- **Tokenize** raw user input into a form consumable by the classification and
  extraction models.
- **Classify intent** — determine which high-level action the user is requesting
  (e.g. create a task, set a reminder, query the schedule).
- **Extract entities** — identify and label structured data items (title, date, time,
  duration, priority, person, location) present in the utterance.
- **Parse date/time expressions** — convert both absolute and relative temporal
  references into resolved timestamps, respecting the user's configured timezone.
- **Assemble the `NLPResult`** — combine intent, entities, and a composite confidence
  score into the single output object passed to downstream modules.
- **Support model improvement** — surface prediction confidence in a form that the
  `AnalyticsEngine` can use to feed corrections back into model training.

---

## 5. Internal Logic

### 5.1 NLP Pipeline

Processing is sequential; each stage feeds the next:

```
Raw Text
  │
  ▼
1. Tokenization          — split text into tokens
  │
  ▼
2. Intent Classification — DistilBERT classifies the user's intent
  │
  ▼
3. Entity Extraction     — NER model labels structured entities in the token stream
  │
  ▼
4. DateTime Parsing      — chrono-node resolves date/time tokens to timestamps
  │
  ▼
5. Result Assembly       — build NLPResult { intent, entities, confidence }
```

### 5.2 Sub-modules

#### IntentClassifier

- **Model**: `distilbert-base-uncased` fine-tuned for sequence classification using the
  Hugging Face `Trainer` API on a labelled corpus of task/event utterances.
- **Output**: an `Intent` enum value plus a raw confidence score.

Supported intents:

```typescript
enum Intent {
  CREATE_TASK     = 'create_task',
  CREATE_EVENT    = 'create_event',
  UPDATE_TASK     = 'update_task',
  DELETE_TASK     = 'delete_task',
  QUERY_SCHEDULE  = 'query_schedule',
  SET_REMINDER    = 'set_reminder',
  RESCHEDULE      = 'reschedule',
}
```

#### EntityExtractor

- **Model**: `dbmdz/bert-large-cased-finetuned-conll03-english` NER pipeline (Hugging
  Face `transformers`).
- **Output**: list of `{ text, type, score, start, end }` spans mapped to the
  `EntityType` enum.

Supported entity types:

| Type       | Example value          |
|------------|------------------------|
| `TITLE`    | `"Dentist appointment"` |
| `DATE`     | `"next Tuesday"`        |
| `TIME`     | `"2pm"` / `"14:00"`    |
| `DURATION` | `"1 hour"`             |
| `PRIORITY` | `"High"`               |
| `PERSON`   | `"John"`               |
| `LOCATION` | `"Conference Room B"`  |

#### TaskParser

Coordinates `IntentClassifier` and `EntityExtractor` to build a complete task/event
description from a single utterance. Representative examples:

| Utterance | Extracted output |
|-----------|-----------------|
| `"Schedule dentist appointment next Tuesday at 2pm"` | Task: `"Dentist appointment"`, Date: next Tuesday, Time: `14:00` |
| `"Remind me to call John tomorrow morning"` | Task: `"Call John"`, Date: tomorrow, Time: `09:00` |
| `"High priority: finish report by Friday"` | Task: `"Finish report"`, Priority: High, Deadline: Friday |

#### DateTimeParser

Wraps the `chrono-node` library to resolve temporal expressions to concrete timestamps.

Supported expression classes:

- **Absolute**: `"June 20, 2026"`, `"2026-06-20"`, `"20/06/2026"`
- **Relative**: `"tomorrow"`, `"next week"`, `"in 3 days"`
- **Time-of-day**: `"2pm"`, `"14:00"`, `"afternoon"`, `"morning"` (→ `09:00`)
- **Ranges**: `"next Monday to Friday"`, `"June 20–25"`

Resolution uses the user's timezone sourced from `UserPreferencesManager` to ensure
that relative expressions anchor to the correct local midnight.

### 5.3 Confidence Score

The final `confidence` value (0–1) is a composite of:

- the `IntentClassifier` softmax probability for the winning intent label, and
- the mean NER span score returned by `EntityExtractor` across all extracted entities.

A low confidence score signals to `AgentBehaviorController` that clarification or a
fallback response may be appropriate.

---

## 6. Interactions with Other Modules

```
Client / API Gateway
        │  raw text
        ▼
      [ NLU ]
        │  NLPResult { intent, entities, confidence }
        ▼
AgentBehaviorController
   ├── CREATE_TASK    → TaskManager
   ├── CREATE_EVENT   → Calendar Service
   ├── SET_REMINDER   → Notification Service
   ├── QUERY_SCHEDULE → Calendar Service / TaskManager
   └── RESCHEDULE     → SchedulingEngine
```

| Direction | Counterpart | What is exchanged |
|-----------|-------------|-------------------|
| **Receives from** | API Gateway / client | Raw free-text utterance |
| **Reads from** | `UserPreferencesManager` | User timezone (for `DateTimeParser`) |
| **Outputs to** | `AgentBehaviorController` | Structured `NLPResult` |
| **Feeds back to** | `AnalyticsEngine` | Confidence scores and user corrections, used to retrain `IntentClassifier` and `EntityExtractor` via the Hugging Face `Trainer` pipeline |
