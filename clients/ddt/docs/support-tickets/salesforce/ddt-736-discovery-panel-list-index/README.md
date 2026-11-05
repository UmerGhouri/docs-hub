# Case Management Discovery panel List index out of bounds error

| | |
|---|---|
| **Ticket** | DDT-736 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-17 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-14 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Aura) |
| **Area** | Court Case (Opportunity) record page: the Case Management / Discovery panel |
| **Related** | None recorded |
| **Code** | PR #172 in the ddt-salesforce repo |

---

## 1. Summary

Opening some Court Case (Opportunity) records showed a red `List index out of bounds: 0` toast, and the Case Management / Discovery panel did not render. The panel's Apex tried to auto-create a discovery checklist record and read the first row of a settings list without checking that the list had any rows. Case types with no discovery checklist have no settings row, so the read threw.

The fix guards the empty case in Apex and stops the Aura helper from toasting when there is simply nothing to show. On unmapped case types the panel is now silently absent and the rest of the page loads cleanly.

## 2. What was reported

A red error toast reading `List index out of bounds: 0` when opening certain Court Case records, with the Discovery panel missing. The reported example had case type `Investigation`.

Measured when the ticket was filed: **3,121 of 38,921** Court Cases (8.0%) had a case type with no settings row **and** no existing `Court_Case_Management__c` child. Every one of them threw on load.

| `Case_Type__c` | Count |
|---|---|
| SOS Formal Hearing | 868 |
| SOS RRR | 752 |
| Code Violation | 600 |
| Investigation | 277 |
| Other | 242 |
| Violation of Sentence | 131 |
| Order of Protection | 117 |
| SOS Informal Hearing | 72 |
| Civil Asset Forfeiture | 38 |
| SOS Out of State Hearing | 23 |
| DWLS/R - Misd | 1 |

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `CourtCaseManagement` | Apex class | `getObjectDetails()` guards an empty settings list and a settings row naming a record type that no longer exists, in both branches that look one up |
| `CourtCaseManagementTest` | Apex test | Three new methods (see section 7) |
| `CaseManagementDiscoveryHelper.js` | Aura helper | A successful response with no record renders nothing; a real server failure still toasts |
| `docs/deploys/case-management-settings-guard.md` | Deploy doc | Review doc for the change |

**Out of scope**

- **Building checklists for the unmapped case types.** The decision on the ticket was code fix only. Six of the case types (Code Violation, Investigation, Other, Violation of Sentence, Order of Protection, Civil Asset Forfeiture) do not track discovery, so the panel being absent is the correct end state for them.
- **Mapping the four SOS variants and `DWLS/R - Misd` to existing checklists.** Confirmed to use the same evidence lists as the existing SOS Hearing and DWLS/R record types, but adding those settings rows would auto-create about 1,716 `Court_Case_Management__c` records on the next page loads. That is a data change with a write side effect, so it was deferred to its own ticket.

## 4. How it works (background)

```
User opens a Court Case record
  -> CaseManagementDiscovery (Aura) calls CourtCaseManagement.getObjectDetails()
  -> no Court_Case_Management__c child exists, so it auto-creates one:
       query Court_Case_Management_Settings__c for the case's Court_Case_Management_Type__c
       take row 0, read its RecordType__c
       look up that record type on Court_Case_Management__c and insert the record
  -> the panel renders the checklist for that record type
```

`Court_Case_Management_Settings__c` is a list custom setting. `Court_Case_Management__c` has 9 record types and the settings table had 10 rows covering exactly those 9. None of the case types in section 2 were mapped.

## 5. Root cause

In the auto-create branch, `getObjectDetails()` called `lstSettings.get(0)` with no emptiness check:

```apex
String gsRecordTypeID = Schema.SObjectType.Court_Case_Management__c
    .getRecordTypeInfosByName()
    .get(lstSettings.get(0).RecordType__c)   // throws when the list is empty
    .getRecordTypeId();
```

For an unmapped case type the list is empty, so this throws `List index out of bounds: 0`.

Production's class had last been modified in 2022 and matched `main` byte for byte, so this was a long-standing bug, not repo drift.

Two more defects of the same kind were found while the code was open:

- **A settings row naming a record type that no longer exists.** `getRecordTypeInfosByName().get(name)` returns null for an unknown name, and the code chained `.getRecordTypeId()` straight off it, throwing a `NullPointerException` on the same line.
- **The sibling branch** (an existing child record whose record type differs from the settings row) carried the identical chain. A blank or stale `RecordType__c` there also walked into the null dereference.

**Why fixing Apex alone was not enough.** With the Apex returning normally and an empty result, the Aura helper still treated "no record" as a failure and read `getError()` off a successful response. The user would have seen a red toast with an empty message instead of one with an error message.

## 6. Solution

### 6.1 Apex: resolve the record type first, then null-check it

```apex
Schema.RecordTypeInfo rtInfo = lstSettings.isEmpty()
    ? null
    : Schema.SObjectType.Court_Case_Management__c
          .getRecordTypeInfosByName()
          .get(lstSettings.get(0).RecordType__c);

if (rtInfo == null) {
    // No checklist for this case type (or a dead record type name):
    // nothing to auto-create. Leave the helper empty and return normally.
}
```

- No settings row means there is no checklist to build, so there is nothing to create. The method returns normally with the helper empty.
- The skip path logs at `WARN`, naming which of the two causes fired. "No settings row" is the expected state for case types that do not track discovery; "settings row names a dead record type" is a misconfiguration someone should fix. They were indistinguishable before.
- In the sibling branch the fallback is different on purpose: a record already exists, so the code falls through and shows that record rather than throwing.
- The existing error for a blank `Case_Type__c` ("please select the case type") is unchanged. That one is a real data problem the user can fix.

### 6.2 Aura helper: split "failed" from "nothing to show"

- `state !== 'SUCCESS'` is a real server failure and still shows a toast.
- A successful response with no case management record returns quietly. None of the panel's branches match, so the panel simply does not appear.

Two smaller fixes in the same toast block, found in review:

- An `INCOMPLETE` state (offline, dropped connection) returns an empty error array, which would also have produced an empty red toast. It now falls back to a plain message.
- `force:showToast` expects the type `error` in lower case. The old `'Error'` silently fell back to default styling, so the toast had never rendered red.

The helper file also had to be brought to a clean eslint pass, because the CI format leg lints changed Aura JS and this file had never passed.

## 7. Testing

Three new methods in `CourtCaseManagementTest`; 6 of 6 passed in the production validation.

| Method | Covers | Against the old code |
|---|---|---|
| `testUnmappedCaseTypeDoesNotThrow` | empty settings list | fails with `List index out of bounds: 0` at the same line the user saw |
| `testStaleRecordTypeNameDoesNotThrow` | settings row naming a dead record type | fails with `NullPointerException` |
| `testMappedCaseTypeStillAutoCreates` | a mapped case type still creates exactly one record | passes on old and new code |

Both failure tests were run against production's unfixed class first and failed, so they are real controls. The third is the control on the refactor: a guard that stopped creation for everyone would turn the first two green while breaking the feature.

**A correction worth knowing.** An early version asserted that production's settings table had no `Investigation` row. It could not prove that: list custom setting data is not visible to tests, so the query only saw rows the test itself inserted (1 row in the test, 10 in production). The assertion was kept, but its comment now says only what it actually guards.

**Live check:** the deploy doc lists opening the reported Court Case and confirming no toast and a clean page as a step for the requester. The result is not recorded in the ticket.

**Not covered:** no automated test renders the Aura component.

## 8. Step by step

1. Reproduce: open a Court Case whose `Case_Type__c` has no `Court_Case_Management_Settings__c` row and no `Court_Case_Management__c` child. Expect the red toast.
2. Count the exposure: Court Cases with no `Court_Case_Management__c` child, grouped by `Case_Type__c`, compared against the case types in the settings table.
3. Confirm production's `CourtCaseManagement` matches git before editing.
4. Apply the guard from 6.1 in both branches that look up a record type.
5. Apply the helper change from 6.2.
6. Add the three tests. Run the two failure tests against the unfixed class first and confirm they fail.
7. Validate against production with `CourtCaseManagementTest`.
8. After approval, deploy. No post-deploy script, permission set or data change.
9. Open the reported record and confirm the page loads with no toast.

## 9. Rollback

Redeploy the previous versions of `CourtCaseManagement` and `CaseManagementDiscoveryHelper.js`. That brings the error back for about 3,000 Court Cases, so only do it if the new code itself misbehaves. No data was changed.

## 10. Known limits

- **The panel is silently absent on unmapped case types.** That is intended for the six case types that do not track discovery. For the SOS variants and `DWLS/R - Misd`, users still see no discovery checklist until the deferred settings change is made.
- **Adding a settings row later has a side effect.** The next page load of each matching Court Case auto-creates a `Court_Case_Management__c` record. Decide whether to backfill deliberately or let it happen on read.
- **Renaming or deleting a `Court_Case_Management__c` record type** leaves settings rows pointing at nothing. The panel no longer crashes on that, but it logs a `WARN` and shows nothing.

## 11. Reuse for similar issues

**How to recognise one:** an error such as `List index out of bounds: 0` or `Attempt to de-reference a null object` on page load for some records but not others, where the difference is a configuration row that exists for some values and not others.

- Look for `.get(0)` on a query result and for chained `.get(name).getSomething()` on a map. Both need a guard.
- Measure the blast radius by grouping the affected records on the field that selects the configuration row.
- Fix the server and the client together. A server that returns "nothing" needs a client that treats "nothing" as normal, not as an error.
- Prove a negative test fails on the old code before trusting it to pass on the new code, and add a positive control so a guard that disables the feature cannot pass.
- Custom setting data is not visible in Apex tests. Do not write an assertion that claims to check production's settings.
