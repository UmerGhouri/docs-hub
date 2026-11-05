# One formula field for all matter red flags


| | |
|---|---|
| **Ticket** | DDT-790 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-06 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula field, permission set) |
| **Area** | Matter red flags |
| **Related** | DDT-789, DDT-791, DDT-792, DDT-798, DDT-1268, DDT-1291, DDT-1318 |
| **Code** | PR #637 in the ddt-salesforce repo |


---


## 1. Summary

A hired matter's red flags came from three long text formula fields that printed an image and a sentence per rule. They were split into three only because the output text did not fit in one field. This ticket added one formula field, `Opportunity.Red_Flag_Codes__c`, that emits a short code per firing rule (for example `;NO_DOB;`). Metadata records turn each code into wording, severity and a button. All 16 active rules fit in one field, and three defects in the old logic were fixed on the way.

## 2. What was reported

- 2026-08-21: add one Formula (Text) field on Opportunity that emits a short code per firing rule and makes the three-field split unnecessary. Remove the dead client-type gate and a partner-firm exclusion flag, and rewrite the address check.

## 3. Scope

**In scope**

- New field `Opportunity.Red_Flag_Codes__c`.
- New permission set `Red_Flag_Panel_Access` with read on the field (added to later by DDT-795).

**Out of scope**

- The three old fields stayed in place and kept working; they were deleted later in DDT-800.
- `INV_MISMATCH` was left out of the formula until the invoice roll-up existed (DDT-1268).
- Nothing reads the field yet; the panel arrives in DDT-793 and DDT-794.

## 4. How it works (background)

The old fields were `Court_Case_Red_Flag__c`, `_2__c` and `_3__c`. Each rule spent about 170 characters on `IMAGE(...)` plus a message plus a line break. Together they held 5,207 characters of formula source against a 3,900 limit per field. Emitting a short code instead brings all active rules into one field. Reports can filter on the codes, which a click-configured condition could not offer.

## 5. Root cause

Three defects were found reading the old formulas (project plan, measured on 726 open hired matters):

1. Seven client-data checks (DOB, DL number, Language, How Heard, Commercial License, State Issued, Address) sat behind a gate on `Account.OLD_Client_Type__c`. That field was blank on 706 of 726 matters, so the checks never ran for 97% of cases.
2. The address check used `ISBLANK(Account.BillingAddress)`. ISBLANK on a compound address never returns true, so it had fired 0 times.
3. The invoice rule fired on 322 of 726 matters (handled in DDT-789).

## 6. Solution

One field, every rule emitting `;CODE;`. The version that first shipped (1,471 characters):

```
IF(NOT(Did_Not_Hire__c) && TEXT(Hire_Status__c) = "Hired",
IF(TEXT(Case_Type__c) <> "Code Violation" && Court_Appearances_Past_Due__c > 0, ";PASTDUE_DISPO;", "") &
IF(TEXT(Case_Status__c) = "Open" && Court_Appearances_Scheduled__c < 1, ";NO_APPEARANCES;", "") &
...
IF(ISBLANK(Account.BillingStreet), ";NO_ADDRESS;", "") &
IF(ISBLANK(Account.Client_Birthday_mmddyyyy__c), ";NO_DOB;", "") &
...
IF(Amount > 0 && Invoice__r.fw1__Total_Paid_Amount__c == 0, ";NO_PAYMENT_YET;", "")
, "")
```

Deliberate changes from the old fields, and why:

| Change | Why |
| -- | -- |
| Client-type gate removed | Blank on almost every open hired matter |
| Partner-firm exclusion flag removed | True on 0 open hired matters, so it changed nothing |
| Address checks `BillingStreet` | The compound-address check never fired |
| `INV_MISMATCH` left out for now | Waiting on the invoice roll-up (DDT-1268) |
| Money rules inside the Hired gate | The panel sits on hired matters, so nothing changes there |
| `NO_PAYMENT_YET` reads the invoice's paid amount | Production's old field did this; the repo copy and the ticket were behind production |

Design rules: each code is wrapped in semicolons on both sides, so a report filter for `contains ";NO_DOB;"` cannot half-match a longer code; no code is a prefix of another; no comments inside the formula, because they count toward the limit.

## 7. Testing

- Every field the formula reads was confirmed in production by describe.
- Before deploy, the formula was re-implemented in Python and run on all 733 open hired matters, then compared with the old fields record by record: 358 identical, 372 lose `INV_MISMATCH` (expected), 10 gain a client-data flag (expected), 0 unexplained.
- Check-only validation against production: the formula compiles within the size limit.
- After deploy, the real record-by-record check (DDT-798): 732 matters compared, 0 unexplained.
- Not covered: no Apex test at this stage; formula behaviour tests came later (DDT-1291).

## 8. Step by step

1. Measure the old formula source length per field to find why it was split.
2. Write one Text formula that emits a delimited code per rule, with no comments.
3. Check that no code is a prefix of another.
4. Confirm every referenced field exists in production.
5. Simulate the formula against production data and compare record by record with the old fields.
6. Grant read on the new field through an additive permission set and assign it to whoever runs the parity check.
7. Validate check-only, deploy, then run the parity check.

## 9. Rollback

Delete `Opportunity.Red_Flag_Codes__c` and the `Red_Flag_Panel_Access` permission set. Nothing read them at that stage.

## 10. Known limits

- Changing when a flag fires still needs a formula edit and a deploy; only wording, order, severity and buttons are Setup edits.
- The formula has a hard 3,900-character limit; later changes took it to 1,977 characters.

## 11. Reuse for similar issues

- Signs of the same problem: several formula fields that each hold part of one list, or a formula near the size limit because it prints text and images.
- Emit short codes from the formula and keep display text in custom metadata.
- Look for gates on fields that are blank on most records; they silently switch whole blocks of logic off.
- `ISBLANK` on a compound address field never fires; test a component field such as the street.
