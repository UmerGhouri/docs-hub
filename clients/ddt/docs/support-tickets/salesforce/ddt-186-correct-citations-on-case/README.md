# Adding, removing or correcting citations on a case

| | |
|---|---|
| **Ticket** | DDT-186, DDT-568, DDT-642, DDT-1153, DDT-1173, DDT-1227, DDT-1267 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-08 |
| **Priority** | Medium |
| **Reported by** | Case Manager and Firm staff, 2026-06-17 to 2026-10-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (`Citation__c` records, picklist metadata, Flow) |
| **Area** | Case data: the citations (charges) recorded on a client's Account and Court Case |
| **Related** | DDT-1103 and DDT-1474 (same picklist change, other charges) |
| **Code** | PR #594 (DDT-1173) and PR #654 (DDT-1267) in the ddt-salesforce repo. The other five tickets were data fixes with no code. |

---

## 1. Summary

Staff regularly need a citation on a client's file removed, added or re-classified. Most of these requests are a single record fix: a citation was entered in error and has to be deleted. Two of them were different: the charge the client actually received did not exist in the Citation Type picklist, so it could not be recorded at all until a new value was added to the picklist and deployed.

This page covers both kinds: deleting a wrong citation, and adding a missing charge to the picklist and then correcting the record.

## 2. What was reported

- **DDT-186 (2026-06-17).** One client's account had two citations for the same hearing, one categorised as an SOS Formal Hearing and one as an SOS Informal Hearing. The Formal Hearing one was wrong and was deleted the same day; the Informal Hearing one was kept.
- **DDT-568 (2026-07-28).** A staff member wanted to remove an extra citation (a motion) from a client's Court Case but saw no delete option on the citation record. A second citation, on another client's file, was added to the same ticket as "added incorrectly". Both were deleted on 2026-07-29. The ticket also asked whether that staff member should have delete rights on citations; the decision is not recorded in the ticket.
- **DDT-642 (2026-08-04).** A DUI A1 citation had been added to one client's case in error; the client only received a DUI A2 ticket. The DUI A1 citation was deleted, and on 2026-08-05 it was confirmed in production that it was gone and the DUI A2 citation was intact.
- **DDT-1153 (2026-09-29).** An "Other Petty" citation on one client's Court Case was wrong and was deleted the same day.
- **DDT-1173 (2026-10-01).** A Case Manager asked for a citation "Other Petty: Drivers License Classification Violation". That charge did not exist, so the Case Manager had saved the citation as Other / Other / Other as a placeholder. A new picklist value was added (PR #594) and the placeholder citation was changed to it the same day.
- **DDT-1227 (2026-10-05).** Two Petty citations (operating an uninsured vehicle, and a seat belt violation) did not belong on one client's account and were deleted the same day.
- **DDT-1267 (2026-10-07).** A Case Manager asked to re-classify a citation from "Other Petty" to "Cross Designated Median", then added a second one on the same client from "Other Petty" to "Improper Right Turn". Neither value existed. Both were added (PR #654) and both citations were updated in production on 2026-10-08.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Citation__c` records | Data | Wrong citations deleted (DDT-186, DDT-568, DDT-642, DDT-1153, DDT-1227); placeholder or wrong citation types corrected (DDT-1173, DDT-1267) |
| `Citation_Type_List` | Global value set | New values: `Drivers License Classification Violation` (DDT-1173); `Cross Designated Median` and `Improper Right Turn` (DDT-1267) |
| `Citation__c.Citation_Type__c` | Dependent picklist | Each new value made available under the Case Type **Petty** only |
| `Citation__c.Final_Citation__c` | Restricted picklist | The same new values added |

**Out of scope**

- **Delete permission on citations.** DDT-568 raised whether the requesting staff member should be able to delete citations. No decision is recorded, and no permission was changed under these tickets.
- **Ranking of the new charges.** Each new value gets a `Citation_Ranking__c` record with no rank. Setting a rank was left to the firm (see section 10).
- **Case types other than Petty.** The new values were mapped to Petty only, because each request named a petty charge.

## 4. How it works (background)

```
Citation__c
  Citation_Case_Type__c   (picklist: Petty, DUI, Motion, Other, ...)   controls ->
  Citation_Type__c        (restricted, dependent; values from the Citation_Type_List global value set)
  Final_Citation__c       (separate restricted picklist)

On save, the Citation_After_Save flow:
  - copies Citation_Type__c into Final_Citation__c when Final is blank
  - marks Amended__c = true when the citation type changes but Final does not
  - creates a Citation_Ranking__c for a new citation type
```

Two consequences matter for this page:

- **A restricted picklist rejects any value it does not list.** Staff can edit a citation themselves (a staff member pointed this out on DDT-1267), but only to values that already exist. A charge that is not in `Citation_Type_List` cannot be entered by anyone until the value is deployed.
- **A value missing from `Final_Citation__c` breaks saving.** Because the flow copies the citation type into `Final_Citation__c`, a value added to the global value set but not to `Final_Citation__c` makes every save of that type fail with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST`.

## 5. Root cause

### 5.1 Citations entered in error (DDT-186, DDT-568, DDT-642, DDT-1153, DDT-1227)

A citation was added that the client did not receive: a duplicate with the wrong hearing category, a charge from the wrong DUI class, or a charge that did not belong on the file. How each error was made is not recorded in the tickets. In DDT-568 the requester could not delete the record because no delete option was offered to them.

### 5.2 Charge missing from the picklist (DDT-1173, DDT-1267)

The charges requested did not exist in `Citation_Type_List`. For DDT-1173, production's 214 values were checked on 2026-10-01 and none was a licence classification charge; the nearest existing values (`Violation of Drivers License Restriction`, `DL Restriction Violation`, `No CDL When Required`) describe different offences. For DDT-1267, the requester confirmed that the charge was "not listed". So both requests were really requests for new picklist values, not record fixes. The Case Manager in DDT-1173 had worked around the gap by saving an Other / Other / Other placeholder.

## 6. Solution

### 6.1 Deleting a wrong citation

The wrong `Citation__c` record was deleted and the correct one, where there was one, was left in place. The tickets do not record whether the deletion was done from the record page or with a data tool. DDT-1153 asked to confirm that no automation depends on the citation before deleting it; the result of that check is not recorded in the ticket.

### 6.2 Adding a missing charge

Three metadata files change together. This is the same shape as DDT-1103 (PR #468) and DDT-1474 (PR #862).

1. **`Citation_Type_List`**: add the value. Label and API name identical, like every other value.
2. **`Citation__c.Citation_Type__c`**: add a dependency entry so the value is offered when Case Type is Petty:

    ```xml
    <valueSettings>
        <controllingFieldValue>Petty</controllingFieldValue>
        <valueName>Cross Designated Median</valueName>
    </valueSettings>
    ```

3. **`Citation__c.Final_Citation__c`**: add the same value, or new citations of that type fail to save (section 4).

Why it was done this way:

- **Production retrieved first.** For DDT-1173, all three files were retrieved from production before editing and committed on their own. The two field files differed from `main` only in value order, so nothing was lost by taking production's copy. For DDT-1267 the fresh production retrieve matched `main` exactly.
- **Additive only.** No value was removed or renamed, so no existing citation changes.
- **Petty only.** The requests named petty charges. Other case types can be added later as a separate request.

### 6.3 Correcting the record after the value is live

Set **all three** fields together: Case Type, Citation Type and Final Citation. Sandbox testing for DDT-1173 showed that changing only Case Type and Citation Type leaves Final Citation on the old value, and the flow then sets `Amended__c = true`, recording an amendment that never happened. For DDT-1173 the existing placeholder citation was updated rather than a second citation created.

## 7. Testing

**Deletions.** DDT-642 was checked in production after the fix: the wrong citation was gone and the correct one intact. The requester confirmed DDT-1173 in the Citation tab, and the requesters of DDT-186 and DDT-1153 acknowledged the fix. No verification is recorded for DDT-568 or DDT-1227 beyond the support team's "done".

**New picklist values (DDT-1173),** in the sandbox, with anonymous Apex inside a savepoint that was rolled back:

| Test | Result |
|---|---|
| Insert with Case Type Petty and the new value | Saved; the flow set Final Citation and created a `Citation_Ranking__c` with no rank |
| Insert with Case Type Criminal Misdemeanor and the new value | Rejected with `INVALID_OR_NULL_FOR_RESTRICTED_PICKLIST`, so the dependency holds |
| Update a copy of the placeholder citation, setting all three fields | `Amended__c` stayed false |
| Same update, without Final Citation | `Amended__c` became true (the wrong way to do it) |

A check-only deploy of the three components to production ran `DispositionEntryControllerTest` and `DispositionDerivationServiceTest`, the two Apex classes that read these picklists, and all 85 tests passed. That proves the components deploy and those classes still pass. It did not render the citation screen.

**Not covered:** nobody recorded opening the New Citation screen and seeing the new value under Petty, apart from the requester confirming the corrected citation in the Citation tab. No separate test record is in the repo for DDT-1267.

## 8. Step by step

**To delete a wrong citation**

1. Confirm with the requester exactly which citation is wrong, and which one (if any) must stay.
2. Open the client's Account or Court Case, find the citation in the Citations related list, and delete it.
3. Check the Citations list again: the wrong one is gone, the correct one is still there.
4. Tell the requester.

**To add a missing charge**

1. Confirm the charge really is missing. Check `Citation_Type_List` in production for a value with the same meaning under a different wording.
2. Retrieve `Citation_Type_List`, `Citation__c.Citation_Type__c` and `Citation__c.Final_Citation__c` from production, and commit that copy before you edit it.
3. Add the value to all three, mapped to the right Case Type (section 6.2).
4. Deploy to a sandbox and test as in section 7, including a rejected insert under a wrong Case Type.
5. Validate against production. Get human approval and deploy:

    ```powershell
    sf project deploy start -o '<prod-alias>' `
      -m "GlobalValueSet:Citation_Type_List" `
      -m "CustomField:Citation__c.Citation_Type__c" `
      -m "CustomField:Citation__c.Final_Citation__c" `
      --test-level RunSpecifiedTests --tests DispositionEntryControllerTest --tests DispositionDerivationServiceTest
    ```

6. Update the citation, setting Case Type, Citation Type and Final Citation together. Check `Amended__c` is still false.
7. Ask the firm to set a rank on the new `Citation_Ranking__c`.
8. Ask the requester to confirm the citation in the Citation tab.

## 9. Rollback

- **A deleted citation** can be restored from the Recycle Bin while it is still there. After that it must be re-entered.
- **A new picklist value:** deactivate it in Setup → Picklist Value Sets → Citation Type List. **Do not delete it once a record holds it**: deleting a global value set value rewrites or blanks every record carrying it.

## 10. Known limits

- **New charges have no rank.** The `Citation_Ranking__c` created for each new value has no `Rank__c`. How an unranked citation affects the Court Case's most-severe citation was not tested. Whether ranks were set afterwards is not recorded.
- **Each new value is offered under Petty only.** The same charge filed as a misdemeanour needs another dependency entry.
- **Who may delete citations** is still an open question from DDT-568.

## 11. Reuse for similar issues

**How to recognise which kind it is:** if the citation exists but is wrong, it is a record fix. If the requester says the charge "is not listed", or has saved a placeholder such as Other / Other / Other, it is a picklist change.

- **Ask whether staff can do it themselves.** Editing a citation to an existing value does not need the support team.
- **Look for a placeholder before creating a record.** In DDT-1173 the right fix was to correct the existing placeholder citation, not to add a second one.
- **For a restricted picklist fed by automation, change every picklist the automation writes to.** Here that is `Final_Citation__c` as well as the global value set.
- **When a flow tracks "amended", set every field the flow compares** in one update, or it records a change that did not happen.
- **Copy the shape of earlier tickets.** DDT-1103, DDT-1173, DDT-1267 and DDT-1474 all added citation types the same way.
