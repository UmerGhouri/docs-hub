# Duplicate leads from the retargeting landing page

| | |
|---|---|
| **Ticket** | DDT-478 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-27 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-07-22 |
| **Resolved by** | Support team |
| **Systems** | Firm website (landing page), Salesforce (Lead) |
| **Area** | Intake: leads arriving from the Brand Retargeting Campaign landing page |
| **Related** | None recorded |
| **Code** | None. Fixed on the website side |

---

## 1. Summary

Leads submitted through the Brand Retargeting Campaign landing page were arriving in Salesforce twice. The cause was on the website side, and it was fixed there, not in Salesforce.

## 2. What was reported

On 2026-07-22 intake reported that two people who had submitted the landing page form each had duplicate Lead records. When the problem was looked into, about **28 duplicates** had been created over the previous two weeks.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Brand Retargeting Campaign landing page | Firm website | Fixed so a submission no longer creates two Leads |

**Out of scope**

- **Salesforce duplicate and matching rules.** No change recorded.
- **Merging the existing duplicates.** Cleanup of the roughly 28 was proposed; whether it was done is not recorded in the ticket.

## 4. How it works (background)

```
Visitor submits the landing page form
  -> the website posts the submission to Salesforce
  -> a Lead is created
```

A Lead created twice for the same submission points at the sender (the form or the integration posting it), not at Salesforce, unless Salesforce automation is itself creating the second record.

## 5. Root cause

The duplicates came from the **website side**. Firm management confirmed on 2026-07-22 that it had been fixed there. The specific fault (for example a double submit or a replayed post) is not recorded in the ticket.

## 6. Solution

Fixed on the website. Nothing was changed in Salesforce.

## 7. Testing

Not recorded in the ticket. The ticket asked for a test submission; its result was not recorded.

## 8. Step by step

What was done, in order:

1. **Intake reported** two example people with duplicate Leads from the landing page.
2. **The duplicates were counted:** about 28 over the previous two weeks.
3. **Firm management confirmed** the cause was fixed on the website side.
4. **Cleanup of the existing duplicates was proposed** to firm management. The outcome is not recorded in the ticket.

To repeat it, establish whether the second Lead comes from the sender before changing anything in Salesforce, hand the fix to whoever owns the website, and get sign-off before merging existing duplicates.

## 9. Rollback

Not applicable: nothing was changed in Salesforce.

## 10. Known limits

- **Salesforce has no recorded safeguard for this source.** If the website posts twice again, duplicates will be created again.
- **Existing duplicates** may still be in the org; cleanup is not recorded.

## 11. Reuse for similar issues

**How to recognise one:** pairs of identical Leads that all come from one web source.

- **Look at the creator and the timing** before touching duplicate rules. Two posts from the sender cannot be fixed by Salesforce alone.
- **Fix it at the source** where possible; Salesforce duplicate rules are the safety net, not the fix.
- **Get sign-off before merging** existing duplicates, since a merge cannot be undone cleanly.
