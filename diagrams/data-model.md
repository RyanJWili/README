# Core data model

Logical MongoDB collections and relationships (from `@dodo-world/proj-coach-schemas`). Not every collection is shown—focus on matchmaking and SMS paths.

```mermaid
erDiagram
  users ||--o{ user_profiles : has
  users ||--o{ sms_chats : has
  users ||--o{ matchings : participates
  users }o--|| schools : belongs_to
  sms_chats ||--o{ sms_chat_messages : contains
  matchings ||--o{ match_approvals : may_have
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

## Match status (simplified)

Typical progression: matching proposed → scheduling → contact exchanged → dated. Failure states include refused, expired, pick-time failed. See [match-state-machine.md](match-state-machine.md).

## Schools

- **Official** schools have `name` + `code` (e.g. `CAL`, `UCLA`).  
- **Unofficial** entries created on first `.edu` signup until ops merges or launches.  
- `users.school` references `schools.code`.

## Shared package

All services import schemas from **proj-coach-schemas** to avoid drift. When adding a field, publish the package and bump consumers.
