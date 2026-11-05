# Check that converted records already link their touch rows


| | |
|---|---|
| **Ticket** | DDT-815 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-09-16 (analysis only; nothing deployed) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, SOQL) |
| **Area** | Attribution: Lead conversion, Client / Court Case / Matter creation |
| **Related** | DDT-811, DDT-814, DDT-570, DDT-1022 |
| **Code** | No code change |


---


## 1. Summary

The attribution plan proposed a new flow to link Inbound Contact rows to the Client, Court Case and Matter when a Lead converts. The ticket's research showed that Apex already does this, and that a flow would have failed on the main hire path and risked rolling back conversions. The task became a read-only check: are any referral rows missing their links? The answer was no, apart from two go-live test rows. A larger finding (almost no referrals are captured on the Court Case intake path) was split out as DDT-1022.

## 2. What was reported

- 2026-08-24 (DDT-815): do not build the conversion flow. Check the Apex that already does it. Done when there is a written answer, backed by query output, to two numbers: how many Inbound Contact rows created since 2026-07-28 are missing a Client, and how many are missing both a Court Case and a Matter.
- 2026-09-14: reassigned to a developer, with a note to filter to referral rows only, because the DDT-814 touch rows legitimately have no Client until conversion.

## 3. Scope

**In scope**

- One read-only query against production on `Inbound_Contact__c`.
- A review of `ReferralSourceService.linkOnConversion` and its callers.

**Out of scope**

- Any new flow or Apex change: none was needed.
- Capturing referrals on the Court Case intake path. That is a separate, larger piece of work (DDT-1022).

## 4. How it works (background)

`ReferralSourceService.linkOnConversion` finds the Lead's Inbound Contact and sets `Account__c`, `Opportunity__c` (Court Case) and `Matter__c`, keeping a write-once rule for the Client and catching its own errors. It is called from `ClientConvertController`, `HiredMatterConvertController` and twice from `NewPotentialClientController`.

Two reasons a flow would not have worked:

- `HiredMatterConvertController` (the contract-signed path) deliberately never calls `Database.LeadConvert`, because conversions kept failing with `RECORD_IN_USE_BY_WORKFLOW`; the Matter links to the Lead through `Matter__c.Lead__c` instead. In the 90 days before the ticket, 52 of 74 Matters created from a Lead had a Lead that was still not converted. A flow keyed on conversion would never fire there.
- Writing to Inbound Contact during conversion fires its trigger, which updates the Lead, which fires a recompute flow, which updates the Lead again. Inside a conversion that is the same `RECORD_IN_USE_BY_WORKFLOW` failure, and a flow has no try/catch, so it would roll back the whole client and Court Case creation.

## 5. Root cause

Not applicable: there was no linking fault. Evidence, read-only in production on 2026-09-16, filtered to referral rows (`Referrer_Type__c != null`; the object then held 323 rows, 11 of them referrals):

```
SELECT Id, CreatedDate, Lead__c, Account__c, Opportunity__c, Matter__c
FROM Inbound_Contact__c ORDER BY CreatedDate
```

- Missing a Client: 1.
- Missing both a Court Case and a Matter: 2.

Both are go-live-day smoke-test rows created on 2026-07-28 on test Leads. Excluding them, both numbers are 0: every real referral row from 2026-07-30 on has a Client and either a Court Case or a Matter. The production `ReferralSourceService` matched `main`, with the DDT-811 write-once guard live.

## 6. Solution

No change. Step 3 of the plan was marked "already built". The ticket closed on the query evidence, not on the grounds that something was active.

The larger finding was raised separately. The ticket had noted that 481 Leads converted with a Court Case since 2026-07-28 while exactly one Inbound Contact row carried a Court Case. At closing the figure was 1 of 841. Linking cannot fix rows that were never created: nothing captures the referring party on the Court Case intake path. That is DDT-1022.

A standing rule was added to the plan: no new record-triggered flow on Lead that writes to `Inbound_Contact__c`.

## 7. Testing

Not applicable: read-only analysis. The query and its result are in section 5.

## 8. Step by step

1. List every caller of the existing conversion-link method before building anything new.
2. Check which hire paths actually call `Database.LeadConvert`; count Matters whose Lead is still unconverted.
3. Run the query above in production, filtered to referral rows.
4. Explain any row missing a link; exclude known test rows explicitly and say so.
5. If there is a real gap, fix it inside `linkOnConversion` (keeping its try/catch and write-once rule), not in a flow.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The check covers referral rows only. Touch rows from DDT-814 and DDT-818 are linked by other means (the nightly re-link in DDT-823).
- The two test rows were identified but not removed.
- Updating the project's plan file to mark the step built was left to firm leadership; whether it was done is not recorded in the ticket.

## 11. Reuse for similar issues

- Before building automation from a plan, search for code that already does the job and measure whether it works.
- Do not assume conversion fires on every hire path; some paths create records without converting the Lead.
- Avoid flows that write related records during Lead conversion; they hold locks the conversion needs and cannot catch errors.
- When a "linking" gap appears, check whether the rows were ever created before blaming the linker.
