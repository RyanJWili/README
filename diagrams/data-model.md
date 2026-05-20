# Core data model

Logical MongoDB collections and relationships (from `@dodo-world/proj-coach-schemas`). Not every collection is shown—focus on matchmaking and SMS paths.

```mermaid
erDiagram
  users ||--o{ user_profiles : has
  users ||--o{ matching_statuses : has_history
  users ||--o{ sms_chats : has
  users ||--o{ matchings : participates
  users }o--|| schools : belongs_to
  sms_chats ||--o{ sms_chat_messages : contains
  matchings ||--o{ matching_histories : tracks
  matchings ||--o{ match_approvals : may_have
  matching_statuses {
    ObjectId userId FK
    string status
    boolean active
  }
  schools {
    string code PK
    string name
    boolean febLaunchStatus
    string timezone
  }
  users {
    ObjectId _id PK
    string school
    array pool
    string phone
    string status
  }
  sms_chats {
    ObjectId _id PK
    ObjectId user FK
    string state
    string nextIntent
  }
  matchings {
    ObjectId _id PK
    array users
    string status
    string school
  }
```

## Status fields (do not merge)

- **`matching_statuses.status`** — per-user eligibility (`Waiting`, `Matched`, `NeedMoreInfo`, …). Drives chatbot routing.  
- **`matchings.status`** — per-pair workflow (`Making Poster`, `TimeScheduled 1/2`, `Dated`, `PickTimeFailed`, …).  

Full enums and diagrams: [match-state-machine.md](match-state-machine.md).

## Schools

- **Official** schools have `name` + `code` (e.g. `CAL`, `UCLA`).  
- **Unofficial** entries created on first `.edu` signup until ops merges or launches.  
- `users.school` references `schools.code`.

## Shared package

All services import schemas from **proj-coach-schemas** to avoid drift. When adding a field, publish the package and bump consumers.
