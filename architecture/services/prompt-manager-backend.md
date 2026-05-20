# prompt-manager-backend

**Runtime:** NestJS on Bun  
**Role:** Versioned prompt CMS and publish workflow for all LLM consumers.

## Responsibilities

- Store prompt templates and metadata (version, author, tags)  
- Publish workflow: draft → review → active version  
- RPC/HTTP API for `proj-coach-backend` chatbot and analysis jobs to fetch **active** prompt by key  

## Consumers

| Consumer | Usage |
|----------|--------|
| Chatbot | System prompts, onboarding, event agents |
| profile-analysis-service | Scoring / analysis prompt packs |
| Internal frontend | Edit and publish UI |

## Design notes

Prompt content should not be edited only in backend repo env files—production paths go through this service so rollbacks and A/B are possible.

Secrets (model API keys) stay in Infisical; this service stores **text** and configuration references only.

## Related

- [../chatbot-evolution.md](../chatbot-evolution.md) — skills migration may split prompts per skill  
- [../../docs/chatbot/system-overview.md](../../docs/chatbot/system-overview.md)  
