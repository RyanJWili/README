# Chatbot — conversational onboarding

Spec for the **conversational onboarding agent** (current production direction). Users never fill a long form on web first—they answer questions over SMS.

## Principles

- One question per message; react before asking next  
- Short Gen-Z tone; lowercase preferred; minimal emoji  
- Accept messy answers; map slang to structured profile fields via tools  
- Strict **18+** enforcement—never suggest lying about age  
- `.edu` or school email verification via `sendEmailVerificationCode` / `verifyEmailCode`  

## Phase flow (summary)

1. **About you** — birthday, gender, ethnicity, height, hobbies, university year, first impression, referral source  
2. **Preferences** — intention, expected gender, age range, ethnicity preference, physical attraction, match accuracy  
3. **Photos** — real face required; tools to list/delete/upload  
4. **Close** — confirm pool (`wednesday` default); eligibility messaging for non-launched schools  

## Tools (representative)

`updateUserProfile`, email verification tools, `getProfileImageList`, `deleteProfileImages`, `convertYikYakToWednesday` (from yak pool), `handOverToTeam` when stuck.

## Legacy doc

Older **onboarding-agent.md** prompt (wingman persona, slightly different flow) may still exist in prompt-manager history—do not mix versions in production.

## Related

- [system-overview.md](system-overview.md)  
- [../strategy/q2-roadmap.md](../strategy/q2-roadmap.md) — User Memory will augment post-onboarding understanding  

*Synthesized from conversational-onboarding-agent source spec.*
