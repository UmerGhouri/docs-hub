# Confirm NCD button not sending the disposition letter

| | |
|---|---|
| **Ticket** | DDT-288 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-27 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-07-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Flow), S-Docs disposition letter |
| **Area** | Defense cases: confirming the next court date after a continuance |
| **Related** | DDT-287 (disposition email blank, same root field) |
| **Code** | PR #55 in the `ddt-salesforce` repo |

---

## 1. Summary

Clicking **Confirm NCD** (confirm next court date) on a court appearance stopped sending the disposition letter for continued cases. The flow behind the button only creates the letter when `Court_Appearance__c.Disposition__c` is `Continued`, and the new Disposition Entry screen had stopped writing that value for continuances. The fix makes the screen stamp `Continued` again for a genuine continuance.

## 2. What was reported

Staff clicked **Confirm NCD** on a court appearance for a continued case and no disposition letter went out. The ticket names one example account. A matching example (a continued case with a blank template) was also posted on DDT-287. How many letters were missed is not recorded in the ticket.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DispositionEntryController` | Apex class | `saveDisposition` stamps `Disposition__c = 'Continued'` for a genuine continuance |
| `DispositionEntryControllerTest` | Apex test | 3 new tests; the test factory now sets `Case_Type__c` |

**Out of scope**

- **The `CA_Court_Date_Check` flow** behind the button. Its gate (`Disposition__c == 'Continued'`) is correct; the value it checks had stopped being written.
- **The S-Docs letter template.** Unchanged; it reads the same field.

## 4. How it works (background)

```
Attorney records the appearance on the Disposition Entry screen
  -> picks a non-closing action and a next court date
  -> DispositionEntryController.saveDisposition
       (before the fix: Disposition__c written only when the case closes)
Staff click "Confirm NCD" on the court appearance
  -> flow CA_Court_Date_Check
       if Disposition__c == 'Continued' -> create the S-Docs disposition letter job
       else -> nothing, no error
  -> S-Docs "DISPO: Master Template (EMAIL)" reads Disposition__c
```

Before the June 2026 cutover to the action-first Disposition Entry screen, an older `Update_Disposition` flow stamped `Continued` on these appearances.

## 5. Root cause

1. The `CA_Court_Date_Check` flow creates the letter job **only** when `Disposition__c` is `Continued`.
2. The Disposition Entry screen set `Disposition__c` only on the case-closing path.
3. So after the cutover, a continued appearance had a blank `Disposition__c`, and Confirm NCD silently did nothing.

The ticket suggested looking at the button action, the flow, the template, recipients or permissions. None of those had changed; the missing piece was the value the flow gates on. The same blank field also caused the empty template in DDT-287.

## 6. Solution

`saveDisposition` stamps `Continued` when the appearance is a genuine continuance:

```apex
} else if (
  action != null &&
  action.Closes_Case__c != true &&
  action.Captures_Outcomes__c != true &&
  action.Close_Prompt__c == null &&
  action.Action_Group__c != 'Status' &&
  input.nextCourtDateTime != null
) {
  // DDT-288: a non-closing appearance with a confirmed next court date is a continuance.
  ca.Disposition__c = 'Continued';
}
```

Why each condition is there:

- **Not closing, not capturing outcomes, no close prompt.** Those paths set their own disposition.
- **Not a status action** (Warrant Issued, Bond Set, Postponed, Warrant Quashed and Recalled). These carry their own meaning and must not read as "Continued".
- **A next court date is set.** Without one it is not a continuance.
- **`Case_Type__c` is left unchanged.** `Continued` is valid under every case type, so it does not need a controlling value of its own.

## 7. Testing

Three new Apex tests:

| Test | Proves |
|---|---|
| `saveDisposition_continuanceWithNextDateStampsContinued` | A continuance with a next date stamps `Continued` |
| `saveDisposition_continuanceWithoutNextDateDoesNotStampContinued` | No next date, no stamp |
| `saveDisposition_statusGroupActionDoesNotStampContinued` | A status action is never stamped |

The test factory now sets `Case_Type__c`, the controlling value for the dependent `Disposition__c` picklist. 37 of 37 tests in the class passed. The commit records an end-to-end check in a sandbox: the S-Docs job completed and the letter was generated.

**Live check:** the support team asked the reporter to confirm on the example account; the result is not recorded in the ticket.

## 8. Step by step

1. **Open a continued court appearance** whose letter did not go out and check `Disposition__c`. Blank confirms this issue.
2. **Open the `CA_Court_Date_Check` flow** and confirm its decision gates the letter on `Disposition__c` equal to `Continued`.
3. **Add the `else if` branch** in 6 to `DispositionEntryController.saveDisposition`, on top of production's current version.
4. **Add the three tests** in 7, and set `Case_Type__c` in the test factory.
5. **Validate against production** with `DispositionEntryControllerTest`, get the PR approved, deploy.
6. **Record a continuance** on the Disposition Entry screen with a next court date, click **Confirm NCD**, and check the S-Docs job and letter.

## 9. Rollback

Redeploy the previous version of `DispositionEntryController`. That brings back the silent failure, so do it only if the new branch itself misbehaves.

## 10. Known limits

- **Appearances saved before the fix** still have a blank `Disposition__c`. Re-saving them through the screen stamps the value; no backfill is recorded in the ticket.
- **The flow still fails silently** when `Disposition__c` is anything other than `Continued`. A future change that stops writing the value would break the button again with no error.
- **A new status action** must be added to the `Status` action group, or it will be stamped `Continued`.

## 11. Reuse for similar issues

**How to recognise one:** a button or flow "does nothing", with no error, after an upstream process was replaced.

- **Read the gate first.** Find the decision that decides whether the flow acts and check the field it reads on a failing record.
- **When replacing an old flow with new code, list every field the old flow wrote.** Here the old flow stamped `Continued` and the new screen did not.
- **Test the negatives too.** The no-next-date and status-action tests stop the fix from over-stamping.
