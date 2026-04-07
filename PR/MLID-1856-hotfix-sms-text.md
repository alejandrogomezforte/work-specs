# [MLID-1856] Update checkout SMS text

## Summary

- Updated the checkout (symptom) SMS message template with revised copy per stakeholder request
- Reformatted template string for readability (multi-line concatenation, matching other templates)
- Updated test assertions to cover new message content

## Changes

| File | Change |
|------|--------|
| `messageTemplate.ts` | New SMS text: simplified language, explicit "call 911", discharge link in own paragraph |
| `messageTemplate.spec.ts` | Added assertions for new text content |

## PR link

https://github.com/LocalInfusion/li-call-processor/pull/new/hotfix/MLID-1856-update-checkout-sms-text
