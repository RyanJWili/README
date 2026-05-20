# Linq Line Spam Guard Runbook

## Scope

Phase 1 behavior for flagged Linq lines is intentionally conservative:

- `ditto_numbers.status = flagged` hard-blocks user-facing outbound sends
- no auto-reassignment of `sms_chats.dittoNumber`
- no SMS fallback
- no user-visible error

## Canonical Status Source

Linq line status is ingested by `imsg-service`, not `proj-coach-backend`.

- Linq emits `phone_number.status_updated`
- `imsg-service` handles that webhook and writes `ditto_numbers.status = active | flagged`
- `proj-coach-backend` reads that persisted status and enforces the guard in `IMessageService`

Do not add a second Linq webhook ingress in this repo unless the provider architecture changes.

## Enforcement Points

Blocked sends are enforced in:

- `IMessageService.sendMessage`
- `IMessageService.initChat`
- `IMessageService.sendTypingIndicator`

When blocked, backend writes an `imsg_logs` row with:

- `status = blocked`
- `provider = line_status_guard`
- `errorMessage = line_flagged:<operation>`

## Default / Tagged Sender Rules

- active defaults only
- active tagged pools only
- requesting a tag with no active lines fails closed
- flagged lines cannot be set as default through internal config

Existing users pinned to a flagged line remain pinned. Their outbound sends are blocked until the line is restored to `active` or they are reassigned manually in a later workflow.

## Operator Checks

1. Inspect line status via `GET /internal/config/ditto-number`
2. Confirm the affected number shows `status = flagged`
3. Attempt a normal outbound send from a chat pinned to that line
4. Confirm no provider send occurs
5. Confirm an `imsg_logs` row exists with `provider = line_status_guard`
6. Confirm fresh default/tagged selection does not choose the flagged line
7. After recovery, set the line back to `active` and verify sends resume

## Production Notes

- Verify `imsg-service` is still subscribed to Linq `phone_number.status_updated`
- Verify Linq webhook versioning remains explicit on the provider side
- Treat `message.failed` as observability only, not as the source of truth for line flagging
- If a line remains flagged for an extended period, use an affected-user audit before any manual reassignment
