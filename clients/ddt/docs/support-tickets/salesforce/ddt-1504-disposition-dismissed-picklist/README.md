# Disposition save fails: Dismissed not an allowed value

| | |
|---|---|
| **Ticket** | DDT-1504 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-05 |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-11-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, dependent picklist), Dochly letter template |
| **Area** | Disposition Entry: recording the outcome of a court appearance and closing the case |
| **Related** | DDT-1118 (disposition picklist expanded), DDT-1109 (disposition save warning) |
| **Code** | PR #885 in the `ddt-salesforce` repo |

---

## 1. Summary

Saving a dismissal on an Order of Protection case failed with "bad value for restricted picklist field: Dismissed". "Dismissed" was an active value, but `Disposition__c` depends on the case type, and Order of Protection did not allow it, even though the Disposition Entry screen offers Dismissal on that case type. The screen's list of actions and the picklist's dependency had never been checked against each other.

The fix lined the dependency up with the screen, added a guard so a mismatch can no longer crash the save, and adjusted the Dismissed letter for Order of Protection cases.

## 2. What was reported

- **2026-11-04:** a case manager entering a dismissal for one client got:

```
Save failed: Update failed. ... INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST,
bad value for restricted picklist field: Dismissed: [Disposition__c]
```

Nothing on the appearance saved, so the case could not be closed out.

Checking every case type then found the same kind of gap on three more:

| Case type | Open cases | Offered by the screen but refused by the picklist |
|---|---|---|
| Order of Protection | 4 | Dismissal (`Dismissed`), not-guilty trial verdict (`Not Guilty at Trial`), Withdrawal (`Withdrawn`) |
| Code Violation | 4 | Not-guilty trial verdict, Withdrawal |
| SOS RRR | 4 | Withdrawal |
| SOS Abstract Research | 3 | Withdrawal |

No Order of Protection appearance had ever held `Dismissed`.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Court_Appearance__c.Disposition__c` | Field (dependent picklist) | `Dismissed` enabled for Order of Protection; `Withdrawn` enabled for Order of Protection, Code Violation, SOS RRR, SOS Abstract Research |
| `DispositionEntryController` | Apex class | Guard before writing `Disposition__c`; not-guilty translation per case type; hand-kept Withdrawal list replaced by a live check |
| `DispositionEntryControllerTest` | Apex test | New tests for the guard and the new values |
| `scripts/ddt-1504-dismissed-letter-op.apex` | Anonymous Apex script | One-off edit to the "COMP: Dismissed" Dochly letter section |

**Out of scope**

- **No new picklist value.** A dropped Order of Protection petition is recorded as plain `Dismissed`, like every other dismissal. No "OP:" value was added.
- **No record type, permission set, flow or LWC change.**
- **Two unused screen options** (`DWLS/R - Misd`, `DWLS/R - Traffic Felony`) were left alone; no case has ever used them.
- **`Report Sent` (the SOS RRR close value)** is missing from both Opportunity record types. Found in passing; separate ticket.

## 4. How it works (background)

```
Case manager records an outcome on the Disposition Entry screen
  -> screen offers actions per case type (ACTION_BY_CASE_TYPE, from the planner sheet)
  -> DispositionEntryController closes the appearance
  -> DispositionDerivationService.deriveHeadline returns the letter value (e.g. bare "Dismissed")
  -> controller writes Court_Appearance__c.Disposition__c
       (restricted picklist, dependent on Case_Type__c)
  -> value is copied to the case's Final_Case_Disposition__c
  -> Dochly disposition letter uses the value to pick its sections
```

## 5. Root cause

The ticket offered three possible causes. It was none of them exactly:

- **Not a missing or inactive value.** `Dismissed` is active on `Disposition__c`.
- **Not a label versus API-name mismatch.** Label and value are both `Dismissed`.
- **The real cause:** `Disposition__c` is a **dependent picklist controlled by `Case_Type__c`**, and Order of Protection did not enable `Dismissed`.

Two lists were never compared. The screen's per-case-type actions offered Dismissal on Order of Protection. The field's dependency refused the value that action produces. `deriveHeadline` returns a bare `Dismissed` whatever the case type, and the controller wrote it unchecked, so the whole save failed.

A related drift turned up: a hand-kept set of case types allowed to close by Withdrawal (`CLOSING_ACTION_CASETYPES`) left out Non-Moving Violation, which the field does enable, so those withdrawals had been saving with no letter value.

## 6. Solution

### 6.1 Line the dependency up with the screen

Decisions made on 2026-11-03:

- `Dismissed` enabled for **Order of Protection**. A petitioner dropping the order is a dismissal, and it counts as a win in the won/lost reports like any other dismissal.
- `Withdrawn` enabled for **Order of Protection, Code Violation, SOS RRR and SOS Abstract Research**. Withdrawal means the firm withdrew as counsel, which can be true on any case type.

### 6.2 Translate a not-guilty verdict per case type

`DERIVED_DISPOSITION_BY_CASETYPE` maps a derived `Not Guilty at Trial`:

- Code Violation becomes `Not Liable` (the letter has a section for it).
- Order of Protection becomes `OP: Denied`, the same value the Plenary OP Hearing ruling stamps.

Both were already enabled for their case type **and** on both Opportunity record types. That matters: a sandbox test showed the record type's value list is enforced on an Apex write to the case's `Final_Case_Disposition__c`.

### 6.3 A guard at the one place the value is written

Whatever produced the value, if the case type does not enable it, the field is left blank and the save reports why through the existing DDT-1109 warning, for example:

> "Dismissed" is not a disposition an Expungement & Sealing case can carry, so it was left blank. Set it by hand.

The rest of the save lands and the case closes.

Why it's done this way:

- **The enabled values come from the field's live describe** (`dispositionEnabledFor`), so the guard can't drift from the picklist the way the hand-kept list did.
- **Keyed by API value, not label.** The org's existing `DependentPicklistUtil` keys by label, and two values (`Felony`, `Misdemeanor`) have labels that differ from their API names. The decoder was checked against Salesforce's own describe in a sandbox: 25 of 25 case types matched, and the sandbox and production dependency maps were identical.
- **The hand-kept Withdrawal set is gone**, replaced by the same live check.

### 6.4 The letter

The "COMP: Dismissed" letter section said there are no fines and no convictions on the client's driving record, which is wrong for an Order of Protection client. Dochly templates are record data, not metadata, so a script wrapped the sentence in a condition:

```
Before: There are no fines and no convictions on your driving record.
After:  {{#IF Case_Type__c != 'Order of Protection'}}There are no fines and no convictions on your driving record.{{/IF}}
```

Every other case type prints as before. The `!=` form matches conditions already used in live templates. The script refuses to write unless the sentence occurs exactly once, and is safe to re-run.

## 7. Testing

- **Check-only validation against production:** 90 of 90 `DispositionEntryControllerTest` tests passed. The production deploy also ran 90 tests with 0 failures.
- **Proved able to fail, twice:**
  - With the guard disabled, the Order of Protection save test failed with the save exception.
  - With the code but without the picklist change, exactly the three tests that depend on it failed (Order of Protection Dismissed, Order of Protection Withdrawn, and the lookup test). They failed by saving blank, not crashing, which is the guard working.
- **Letter script:** a dry run against production found the sentence once and wrote nothing.
- **Live:** after deploy, `Dismissed` and `Withdrawn` were confirmed on the Order of Protection dropdown, and the case manager was asked to re-enter the dismissal.

**Not covered:** the tests close through Negotiated Plea, not the Dismissal or Bench Trial actions, because the test class seeds neither (the derived-value path is the same). Nothing renders the disposition LWC. The letter edit is not exercised by any test.

## 8. Step by step

1. Read the error: `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST` on a value you know is active points at a **dependency or record type**, not a missing value.
2. Confirm the value is active, and compare label and API name.
3. Check the field's controlling field and which controlling values enable the value (Setup, Object Manager, the field, Field Dependencies).
4. Compare every action the screen offers per case type against the dependency, for all case types, not just the reported one.
5. Agree with the business which values each case type should allow.
6. Update the dependency in the field metadata, and confirm each value is also on every record type the value is copied to.
7. In Apex, check each value against the field's live describe before writing it, keyed by API value.
8. Deploy the field and both classes with `--test-level RunSpecifiedTests --tests DispositionEntryControllerTest`.
9. Run the letter script with its dry-run flag first, then for real.
10. Ask the case manager to re-enter the outcome and confirm it saves, the case closes and the letter reads correctly.

## 9. Rollback

- **Code and field:** revert PR #885 and redeploy. Reverting the dependency brings back the failed saves, so only do it with the guard in place.
- **Letter:** run a copy of the script with the before and after text swapped, or edit the section in Dochly and remove the `{{#IF ...}}` and `{{/IF}}` around the sentence.

## 10. Known limits

- **The screen's action list is still hand-kept.** A new case type or action can create a new mismatch. The guard now leaves the disposition blank with a warning instead of failing the save, but a person still has to set it by hand.
- **Record type value lists are separate from the dependency.** A value can be enabled for a case type and still be refused on the case's record type (`Report Sent` is one).
- **Two unused screen options** remain listed.

## 11. Reuse for similar issues

**How to recognise one:** a restricted picklist rejects a value that is active and spelled correctly.

- **Check three things:** active, label versus API name, and **dependency on a controlling field**. Then check record type value lists, which Apex writes also enforce.
- **Compare the UI's options with the field's allowed values** across every controlling value, not only the reported one.
- **Guard writes against the live describe**, keyed by API value, so a mismatch degrades to a warning instead of a failed save.
- **Replace hand-kept lists** of allowed values with a live check; they drift.
- **Dochly template text is data.** Change it with a script that checks the exact text first, and dry-run it.
