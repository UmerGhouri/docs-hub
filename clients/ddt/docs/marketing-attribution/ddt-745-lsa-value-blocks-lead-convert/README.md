# Lead conversion fails on "Google Local Service Lead"


| | |
|---|---|
| **Ticket** | DDT-745 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-08-17 (reached production) |
| **Priority** | High |
| **Reported by** | Intake team, 2026-08-15 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record type picklist settings) |
| **Area** | Intake: lead conversion |
| **Related** | None recorded |
| **Code** | No code change (Setup change in production) |


---


## 1. Summary

An intake rep could not convert a Lead whose "How did you hear about us?" was "Google Local Service Lead". Conversion failed with a restricted-picklist error. The value existed but was not made available on the record type that a converted client lands on (the Person Account record type). The value was assigned to that record type in production, and conversion with that value no longer hits the error.

## 2. What was reported

- 2026-08-15: converting a Lead fails with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST, bad value for restricted picklist field: Google Local Service Lead: [How_did_you_hear_about_us__c]`. Local Services leads are a regular source, so the value must work end to end.

## 3. Scope

**In scope**

- The client (Account) Person Account record type: making "Google Local Service Lead" available for "How did you hear about us?".

**Out of scope**

- The Lead that triggered the report: it had already been converted by changing its answer to "Google / Online Search" to get past the error. It was not changed back.
- Other picklist values and other record types: not reported.

## 4. How it works (background)

On conversion, the Lead's "How did you hear about us?" is copied to the new client. Leads from Local Services carry the value "Google Local Service Lead". The client's field is a restricted picklist, and on a record type each value must also be switched on for that record type. The conversion screen checks the value against the record type the client is created with.

## 5. Root cause

"Google Local Service Lead" was active on the field but not assigned to the Person Account record type, so the conversion rejected it. Evidence: the error names the field and value, and adding the value to that record type cleared it.

## 6. Solution

In Setup, "Google Local Service Lead" was added to the "How did you hear about us?" values on the Account (Client) Person Account record type. The ticket does not record a matching metadata deploy.

## 7. Testing

- Verified in production by the support team after the change (method not recorded in the ticket).
- Not covered: the ticket asked for a spot-check of other recent Local Services leads; no result is recorded.

## 8. Step by step

1. Read the full error: it names the field and the value.
2. Confirm the value is active on the field.
3. Find which record type the converted record uses, and check the value is assigned to it.
4. Add the value to that record type in Setup.
5. Convert a test Lead with that value through the normal conversion screen.

## 9. Rollback

Remove the value from the record type in Setup. Conversions with that value would fail again.

## 10. Known limits

- Any new value added to the Lead's picklist must also be assigned to the client record type, or the same error returns.
- The change was made in Setup; the ticket does not record it in git.

## 11. Reuse for similar issues

- `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` on conversion usually means the value is not assigned to the target record type, not that the value is missing.
- Test conversion fixes with the same conversion screen the user used.
- When adding a lead source value, add it on the Lead and on every record type it converts into.
