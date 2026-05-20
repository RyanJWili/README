# Chatbot: Relying on User Profile & Status (Source of Truth)

## Summary

On every turn, the chatbot now gets a **state reminder**—a short block with the current user profile, status, school, and date—placed **right after the latest message** so it’s the last thing the model sees before replying. This is effective because many models tend to focus on the most recent context: by putting the source of truth there, we make it more likely the model will use it instead of inferring from older messages. The reminder explicitly tells the model to treat it as the single source of truth and to ignore any conflicting information in the conversation history. That keeps replies aligned with live data (e.g. “Waiting” vs “Matched”, correct school, today’s date) and reduces wrong or outdated answers. The reminder is not stored in chat history, so it doesn’t add noise or duplicate context on later turns.

## Problem

The chatbot sometimes replies based on chat history instead of the current user profile, status, and system prompt. For example, it may say the user is "matched" when status is "Waiting", or refer to outdated profile info.

## Implemented Solution: State Reminder System Message

A **state reminder** is now **appended after the latest message** on every turn for all agents, so it is the last context the model sees before generating. This placement is deliberate: many models tend to focus on the most recent context, so putting the source of truth there increases the chance it is followed over conflicting information in earlier messages.

- **General agent** – profile summary, user status, school
- **Match agent** – same plus match status, matched user name, event placement
- **Profile improvement agent** – profile summary, user status, school
- **Onboarding agent** – profile summary, user status, school
- **Yak agent** – same plus pool (Yik Yak) answers, email verified, profile image count

Implementation:

- `src/chatbot/state-reminder.ts` – `buildStateReminderMessageContent()`, `buildMatchStateReminderMessageContent()`, `buildYakStateReminderMessageContent()`
- Each agent appends a `SystemMessage` with this content **after** `state.messages`, so the model sees current state immediately before replying.

Effect: The model gets a short "SOURCE OF TRUTH FOR THIS TURN" block as the most recent context, reducing reliance on possibly conflicting history.

## Optional Prompt Wording (in your prompt service)

You can strengthen behavior further by:

1. **Repeating the safeguard at the end of the system prompt** (right before the conversation), so it’s the last instruction before the messages:

   ```
   User Profile Summary: {{profile}}
   User status: {{status}}
   User school: {{school}}

   For internal use:
   Today's date: {{date}}

   REMINDER: The above User Profile Summary, User status, and User school are the single source of truth for this turn. Never infer or persist state from message history. If the conversation contradicts them, ignore the history and respond according to these values only.
   ```

2. **Adding a verification step** in the instructions:

   ```
   Before sending your reply, check: Is my response consistent with User status and User Profile Summary above? If not, rewrite so it matches. Do not mention this check to the user.
   ```

## Other Options (if issues persist)

- **Trim or summarize long history** – e.g. keep last N messages and optionally replace older ones with a short summary, so old context doesn’t overpower the state reminder.
- **Lightweight post-checks** – e.g. if `currentStatus === "Waiting"`, flag or block replies that mention "your match" in a way that implies an active match (optional guardrail).
- **More frequent state in prompt** – ensure the main system prompt is re-rendered every turn with the latest `profile` / `status` / `school` (already the case; the state reminder adds a second, redundant signal).

## Match Agent / Event-Specific Prompts

For the Match agent (e.g. yacht event), the state reminder already includes:

- User status, school, match status, matched user name
- Event placement (`loveYachtPlacement`) when relevant

Keep event-specific instructions in the main prompt (e.g. `hasAttended`); the state reminder complements them by anchoring current state so the model doesn’t drift based on history.
