# proj-coach-schemas

**Runtime:** TypeScript library (`@dodo-world/proj-coach-schemas`)  
**Role:** Single source of truth for MongoDB Mongoose schemas and shared enums across Ditto services.

## Why it matters

- **Match status** (`matchings.status`) vs **user status** (`matching_statuses.status`) are defined here—documentation and chatbot routing must use the right collection.  
- Any new field requires a package version bump and consumer updates (today multi-repo; future Nx workspace lib).  

## Key schemas (documentation-relevant)

| Schema | Collection |
|--------|------------|
| `Matching` | `matchings` |
| `MatchingStatus` | `matching_statuses` |
| `User` | `users` |
| `SmsChat` / messages | `sms_chats`, `sms_chat_messages` |
| `School` | `schools` |

## Consumers

`proj-coach-backend`, `profile-analysis-service`, `prompt-manager-backend`, workers, and internal tools import the same types to prevent drift.

## Related

- [../../diagrams/data-model.md](../../diagrams/data-model.md)  
- [../../diagrams/match-state-machine.md](../../diagrams/match-state-machine.md)  
- [../../docs/strategy/monorepo-migration-summary.md](../../docs/strategy/monorepo-migration-summary.md)  
