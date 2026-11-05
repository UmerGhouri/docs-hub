# The 19 red flag rule records


| | |
|---|---|
| **Ticket** | DDT-792 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-06 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom metadata records) |
| **Area** | Matter red flags: configuration |
| **Related** | DDT-789, DDT-791, DDT-798, DDT-1268 |
| **Code** | PR #636 in the ddt-salesforce repo |


---


## 1. Summary

One `Red_Flag_Rule__mdt` record was written for each red flag code: 16 active and 3 inactive. Each holds the message, severity, group, order and button. Messages were copied word for word from the old formula fields as they were in production, so the parity check (DDT-798) could compare exactly.

## 2. What was reported

- 2026-08-21: write the 19 rule records, 16 active and 3 shipped inactive, with wording carried over verbatim and a severity per flag.

## 3. Scope

**In scope**

- 19 records named `Red_Flag_Rule.CODE`.

**Out of scope**

- Any formula or code change; nothing read the records at this stage.
- A button for `PASTDUE_DISPO`: no Opportunity field holding the Id to open was identified. Adding one later is a Setup edit.

## 4. How it works (background)

The formula `Red_Flag_Codes__c` (DDT-790) emits codes. The panel joins each code to its record by `Code__c` and shows the record's message and button. Severity drives the colour and icon: Blocker means the case cannot move, Warning means fix it this week, Info means fix it when you are in there. Before this, every flag was the same red.

## 5. Root cause

Not applicable: new configuration, not a fix.

## 6. Solution

| Group | Code | Severity | Active | Button |
| -- | -- | -- | -- | -- |
| Court | `PASTDUE_DISPO` | Blocker | yes | none |
| Court | `NO_APPEARANCES`, `NONE_SCHEDULED`, `NO_FUTURE_DATE` | Blocker | yes | New Court Appearance |
| Case Setup | `NO_CASE_NUMBER`, `NO_CASE_DESC` | Warning | yes | Inline edit on the matter |
| Case Setup | `NO_CITATIONS` | Warning | yes | New Citation |
| Client Data | `NO_DOB`, `NO_DL_NUM`, `NO_CDL_STATUS`, `NO_STATE_ISSUED`, `NO_ADDRESS` | Warning | yes | Inline edit on the client account |
| Client Data | `NO_HOW_HEARD`, `NO_LANGUAGE` | Info | yes | Inline edit on the client account |
| Money | `PAYMENT_PAST_DUE`, `NO_PAYMENT_YET` | Blocker | yes | Open Invoice |
| Money | `INV_MISMATCH` | Warning | no | Open Invoice, bypass `Bypass_Invoice_Red_Flag__c` |
| Case Setup | `NO_WELCOME_CALL`, `NO_INTRO_CALL` | Warning | no | Inline edit |

Why:

- `NO_APPEARANCES` and `NONE_SCHEDULED` overlapped almost completely (107 and 109 firings). Both were ported for parity and put in the same group so the duplication was visible; it was fixed later in DDT-1318.
- `INV_MISMATCH` shipped inactive while the invoice rule was being decided (DDT-789); DDT-1268 switched it on.
- The two call rules were commented out in the old formula and shipped inactive.

## 7. Testing

- Checked against production on 2026-10-06: the two call-rule fields exist on Opportunity; two case-type values in the old Welcome Call list are no longer active, noted in the record's help text; Court Appearance and Citation both have a lookup to the matter, so New buttons can pre-fill it.
- No code is a prefix of another (checked when the files were generated).
- Check-only validation against production together with the type.
- Read-back after deploy: 19 records, 16 active; the parity check later matched messages to the old fields.

## 8. Step by step

1. Retrieve the old formula fields from production, not from the repo.
2. Write one record per code with the exact old message.
3. Assign severity and group; choose a button type per flag.
4. Verify every field a record names exists in production.
5. Deploy after the metadata type is in production.

## 9. Rollback

Delete the 19 records. Nothing read them at that stage.

## 10. Known limits

- `NO_WELCOME_CALL` and `NO_INTRO_CALL` look like a one-click switch but are not: their condition was never added to the formula. Ticking Active alone shows nothing; turning either on needs a formula line and edit access on its field.
- `PASTDUE_DISPO` has no button.

## 11. Reuse for similar issues

- When replacing display logic, copy the old wording exactly first so a parity check can compare like with like; reword afterwards.
- Ship rules you might want later as inactive records, but write down whether they also need a formula line.
