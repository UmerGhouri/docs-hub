# Generated documents keep the old plaintiff after the Prosecutor Agency changes

| | |
|---|---|
| **Ticket** | DDT-1077 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-24 |
| **Priority** | High |
| **Reported by** | Case staff, 2026-09-21 |
| **Resolved by** | Support team |
| **Systems** | Dochly templates, Salesforce (Court Case data) |
| **Area** | Representation Agreement and Affidavit generation for City/Village cases |
| **Related** | None recorded |
| **Code** | Not applicable: no commit references this ticket |

---

## 1. Summary

When a Court Case's Prosecutor Agency was changed from **Local** to **State**, the Representation Agreement and Affidavit generated from Dochly still showed the **Local** plaintiff. Hand-editing the generated document did not stick for later generations, and a signed document came back showing the old version. The Support team fixed it on 2026-09-24 and corrected a misspelling of "Illinois" in the same documents. **What exactly was changed is not recorded in the ticket.**

## 2. What was reported

- Generated Representation Agreements and Affidavits kept the **Local** plaintiff after the Prosecutor Agency was changed to **State**.
- When a user edited the generated document to the correct State version and sent it, later generations still used the Local version.
- Staff had to remember to hand-edit every generated document.

Wrong plaintiff wording on a client-facing legal document is a compliance and accuracy problem.

After a first response (below), another staff member reported on 2026-09-24 that the problem was **not** fixed: a signed document that came back still showed the incorrect version, it read "Illinos" instead of "Illinois", and this was happening **on every City/Village document**. A second client's Court Case was given as another example.

Two client records were named in the thread. No wider count is recorded.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Representation Agreement | Dochly template | Fixed (detail not recorded) |
| Affidavit | Dochly template | Fixed (detail not recorded) |
| "Illinois" spelling | Template text | Corrected from "Illinos" |

**Out of scope:** not recorded in the ticket.

## 4. How it works (background)

```
Court Case (Opportunity) holds Prosecutor Agency: Local or State
  -> user generates the Representation Agreement / Affidavit in Dochly
  -> template merges the plaintiff wording from the record
  -> user may edit the generated document before sending
  -> document is sent for e-signature and the signed copy comes back
```

The ticket listed three possible causes to check: a cache on Dochly's side, a missing update on the Salesforce side, or a template mapping that reads the wrong source field.

## 5. Root cause

**Not recorded in the ticket.** The three candidate causes in section 4 were listed but the ticket does not say which, if any, it was.

What the thread does record:

- On 2026-09-23 the Support team showed, with screenshots, that a sent document **did** carry the user's edited values when viewed through Dochly's sent-document steps.
- On 2026-09-24 a staff member reported that the **signed** copy still showed the incorrect version, along with the "Illinos" typo, on every City/Village document.
- Later that day the Support team replied: "It's fixed now, and I've also corrected the spelling of Illinois."

## 6. Solution

The fix was made on 2026-09-24, outside the `ddt-salesforce` repo; no commit references this ticket. The only change the ticket names is correcting "Illinos" to "Illinois". The change that made the plaintiff follow the Prosecutor Agency is not described.

## 7. Testing

**Live check:** not recorded in the ticket. The ticket's acceptance criteria asked for Local → State on the Representation Agreement, the same on the Affidavit, and a State → Local regression check; none of these is recorded as run.

**Not covered:** no automated test.

## 8. Step by step

The fix itself is not recorded. To check this problem if it returns:

1. On a test Court Case, set Prosecutor Agency to **Local** and generate the Representation Agreement and Affidavit. Note the plaintiff wording.
2. Change Prosecutor Agency to **State** and generate both again. The plaintiff should change.
3. Change it back to **Local** and generate again, to check the reverse.
4. Send one for e-signature and open the **signed** copy, not only the sent one. Check the plaintiff wording and the spelling of "Illinois".

## 9. Rollback

Not applicable: the change is not recorded, so there is nothing defined to undo.

## 10. Known limits

- **The root cause is unknown**, so it is not possible to say whether other templates that depend on Prosecutor Agency share the problem.
- **Signed copies sent before the fix** keep the wrong wording. Whether any were re-sent is not recorded.

## 11. Reuse for similar issues

**How to recognise one:** a generated document shows an old value after the record has changed, or a hand edit does not survive.

- **Check the signed copy, not just the sent one.** In this ticket the sent document looked right while the signed copy came back wrong.
- **Test both directions** of a value change (Local → State and State → Local).
- **Ask for one concrete record and a screen recording** early; this ticket only moved once examples were provided.
- **Write down what was changed and where** when fixing a template, so the next person does not start from nothing.
