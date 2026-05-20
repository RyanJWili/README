# Conversational Onboarding A/B Testing

## Overview

A/B test conversational onboarding (chat-based) vs control (form-based) using iMessage line assignment. Each Ditto phone number has an `onboardingVariant` field in the `ditto_numbers` collection — set once when the line is added to the DB. Users are randomly assigned to lines by the iMessage provider, which naturally creates the randomized A/B split.

## Line Assignment

10 lines available. Starting with 5 active:

| Group | Count | Variant | `onboardingVariant` value |
|-------|-------|---------|--------------------------|
| Test | 4 lines | chat-based onboarding | `"conversational"` |
| Control | 1 line | form-based onboarding | `"control"` (default) |
| Reserve | 5 lines | inactive | not set yet |

Available line numbers:
```
+14154299449, +14154349801, +14155688132, +14154308563, +14154349774
+14154380638, +14154898138, +14154260535, +14154349815, +14153362896
```

The product/engineering team assigns the variant when adding each line. The assignment is fixed for the duration of the test.

## Setup

To assign a line as conversational:
```javascript
db.ditto_numbers.updateOne(
  { number: "+14154299449" },
  { $set: { onboardingVariant: "conversational" } }
)
```

Lines without `onboardingVariant` or with `"control"` use the form-based flow (default).

## Backend Implementation

### Schema (`ditto-number.schema.ts`)

```typescript
@Prop({ type: String, default: "control" })
onboardingVariant?: "conversational" | "control";
```

### ChatbotService (`chatbot.service.ts`)

```typescript
private async getOnboardingVariant(
  dittoNumber: string | undefined,
): Promise<"conversational" | "control"> {
  if (!dittoNumber) return "control";
  const doc = await this.dittoNumberModel
    .findOne({ number: dittoNumber }, { onboardingVariant: 1 })
    .lean();
  return doc?.onboardingVariant === "conversational" ? "conversational" : "control";
}
```

Called in `handleAIChat`:
```typescript
const onboardingVariant = await this.getOnboardingVariant(chat.dittoNumber);
```

### Tracking

Capture the variant in PostHog for analytics:

```typescript
this.posthog.capture({
  distinctId: userId,
  event: "onboarding_started",
  properties: {
    variant: onboardingVariant,
    dittoNumber: chat.dittoNumber,
  },
});
```

### Metrics to Compare

| Metric | Description |
|--------|-------------|
| Completion rate | % of users who finish onboarding |
| Time to complete | Minutes from first message to email verified |
| Drop-off point | Which field/step users abandon at |
| Fields filled | Average number of optional fields completed |
| Profile quality | Hobby length, photo count, etc. |

## Scaling Up

1. Set `onboardingVariant: "conversational"` on reserve lines to add them to the test group
2. To end the test, set all lines to `"conversational"` or remove the DB check entirely
