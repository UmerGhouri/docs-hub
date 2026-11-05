# Task / Event Auto-Link Related Records (re-link on update)

| | |
|---|---|
| **Status** | Built and tested in the sandbox. Production deploy date: TBD (not found in records) |
| **Requested** | TBD (not found in records) |
| **Built** | Early Aug 2026. The release manifest is dated 5 Aug, the flows were last changed 6 Aug |
| **Objects** | Task, Event, Account (Household / Individual), `AccountContactRelation`, `Investments__c` (Financial Account) |
| **Type** | Two before-save record-triggered flows + one Apex helper change |
| **Release** | "New General Task + Auto-Link Release" (`manifest/package.autoLinkRelease.xml`) |
| **Change set** | TBD (not found in records) |
| **Test class** | `TestinvestmentsTrigger` (Run specified tests) |

---

## 1. Summary

Two flows fill in a Task's or Event's **Name** (`WhoId`) and **Related To** (`WhatId`) from whichever one the user entered. Before this release they ran **on create only**. If someone later changed the Name to a different person, Related To kept pointing at the old Household.

This release made three changes:

1. **Both flows now run on update too.** They re-run when Name or Related To changes. A new **Safe to Set Related To?** check stops them overwriting a Related To the user set on purpose.
2. **`InvestmentsTriggerHelper`:** when **Client Name** is added or changed on a Financial Account, the **Name** on that Financial Account's Tasks is set to the new person. Related To stays on the Financial Account.
3. **`newGeneralTaskForm`:** the modal waits until every field is ready before it shows the form. See [01](../01-new-general-task-button/README.md).

---

## 2. Client request (as received)

TBD (not found in records). This work came before the earliest session transcript we have (2 Sep 2026). The behaviour below comes from the release manifest and the post-deployment guide written at the time (`docs/Post-Deployment-Steps-AutoLink-Release.md`).

---

## 3. Scope

**In scope**
- `Task_Auto_Link_Related_Records` (label **Task – Auto-Link Related Records**)
- `Event_Auto_Link_Related_Records` (label **Event - Auto Link Related Records**). Same logic, on Event.
- `InvestmentsTriggerHelper.handlerAfterUpdate()`: the Client Name → Task Name re-map
- `newGeneralTaskForm` load fix (and the `newGeneralTaskAction` / quick action that ship with it)

**Out of scope (deliberately excluded from the release)**
- `Event_Auto_Link_Household`. An old, obsolete flow. Don't ship it and don't activate it.
- `investmentsTrigger`. Unchanged. Its after-update block already calls `handlerAfterUpdate`.
- FlexiPages and page layouts. Deploying them would overwrite Production.

---

## 4. How the auto-link works

Both flows are **before-save** on `CreateAndUpdate` and only start when:

```
ISNEW() || ISCHANGED({!$Record.WhoId}) || ISCHANGED({!$Record.WhatId})
```

The first decision, **Task is created from**, picks a path from the Id prefixes:

| Path | When | What it sets |
|---|---|---|
| **Financial Account** | Related To starts with `a0D` (`Investments__c`) | **Name** = the Financial Account's Client Name (`Account_Name__r.PersonContactId`) |
| **Individual Client** | Name starts with `003` (Contact) | Finds the contact's ACRs (excluding Relationship = Accountant, Referral Source, Pastor, Trustee). Keeps those whose Account is **Household (GVA)** or **Business Household**. Sets **Related To** to the first one, but only if *Safe to Set Related To?* says yes. |
| **Household & Individuals** | Related To starts with `001` **and it's a new record** | If Related To is a Household (GVA), leave it. If it's a Business Household, leave it. Otherwise (an Individual): if the person has exactly one ACR, set Name = person and Related To = that Household. If not, set Name = person and **clear Related To**. |

### The new guard: *Safe to Set Related To?*

On update, the flow replaces Related To only if **any** of these is true:

- the record is new
- Related To is blank
- the current Related To is a Household (GVA), a Business Household (`Business`), or an Individual

If Related To points at anything else (a Financial Account, a Case and so on), the user chose it on purpose, so the flow leaves it alone.

### The Apex change (`InvestmentsTriggerHelper.handlerAfterUpdate`)

```apex
// Only when Client Name is populated - clearing it leaves the Task's Name alone
Boolean clientChanged = newInv.Account_Name__c != oldInv.Account_Name__c
                        && newInv.Account_Name__c != null;
...
// Re-map Name to the Individual now linked on the Financial Account.
// Related To stays on the Financial Account.
if(clientChangedInvestmentIds.contains(tsk.WhatId)) {
    Id personContactId = personContactByAccountId.get(inv.Account_Name__c);
    if(personContactId != null && tsk.WhoId != personContactId) {
        tsk.WhoId = personContactId;
        taskChanged = true;
    }
}
```

It reuses the existing loop that already copies GVA Asset Management and Firm ID down to the Financial Account's Tasks (non-recurring Tasks only, partial-success update).

---

## 5. Root cause

The flows were **create-only**, so later edits to Name never moved Related To. Changing Client Name on a Financial Account didn't reach Tasks that already existed. The exact client report is TBD (not found in records).

---

## 6. Solution design

### Components

| Type | Name | Change |
|---|---|---|
| Flow | `Task_Auto_Link_Related_Records` | Start changed to `CreateAndUpdate` + the entry formula above. Added the `Safe_To_Set_Related_To` decision, which reads the current Related To through `Get_Current_Related_To`. |
| Flow | `Event_Auto_Link_Related_Records` | Same changes, on Event. Created in the sandbox on 29 Jul 2026. |
| Apex Class | `InvestmentsTriggerHelper` | `handlerAfterUpdate` also watches Client Name (`Account_Name__c`) |
| LWC | `newGeneralTaskForm` (+ `newGeneralTaskAction`, `Account.New_General_Task_1`) | Holds the form until every field is ready |

### Hardcoded record type Ids (existing, not added by this release)

`Get_Account_For_ACR` and `Get_Household` filter on literal Ids:

| Record type | Id in the flows |
|---|---|
| Household (GVA) | `<Household (GVA) record type Id>` (18-char literal, see the flow XML) |
| Business Household | `<Business Household record type Id>` (18-char literal, see the flow XML) |

If Production's Ids differ, the flows stop finding Households **with no error**. The new `Safe_To_Set_Related_To` decision uses `RecordType.DeveloperName` and isn't affected.

---

## 7. How to test in the sandbox

**Test A: Task auto-link on create and update**
1. Create a Task with **Name** = an Individual who has a Self relationship to a Household. Save. **Related To** is that Household.
2. Change **Name** to a different Individual. Save. **Related To** changes to the new Household.

_Screenshot in the private copy: 01-task-relink-on-update.png (Related To follows Name)._

**Test B: a deliberate Related To is kept**
1. Set a Task's **Related To** to a Financial Account, then change **Name**. Save.
2. Related To still shows the Financial Account. Only Household, Business Household or Individual values get replaced.

**Test C: Client Name re-map**
1. On a Financial Account with no Client Name, create a Task against it with Name blank.
2. Set the Financial Account's **Client Name**. Save.
3. The Task's **Name** is now that person, and **Related To** still shows the Financial Account.

_Screenshot in the private copy: 02-client-name-remap.png (Task Name re-mapped)._

**Test D:** repeat Test A with an **Event**.

---

## 8. Test classes and coverage

| Component | Test class | Coverage |
|---|---|---|
| `InvestmentsTriggerHelper` | `TestinvestmentsTrigger` | TBD for this release (not found in records). Later, in Sep 2026, it was 82% (see [09](../09-orion-rep-id-sync/README.md)). |

Flows and the LWC need no Apex test class.

> ⚠️ **Pre-existing failing tests.** `InvestmentClientLinkerTest` had 2 assertion failures in the sandbox (`linksWhenSingleMatch_ignoresDashes`, `doesNotLinkWhenDuplicateSsn`). They failed on the code already deployed, not because of this release, and the class builds its own data, so they will fail in Production too. **Run local tests will fail.** Use *Run specified tests* → `TestinvestmentsTrigger`, and confirm org-wide coverage still meets 75%.

---

## 9. Deployment

**Change set name:** TBD (not found in records).

Components (from `manifest/package.autoLinkRelease.xml`):

| Component Type | Name |
|---|---|
| Flow | `Task_Auto_Link_Related_Records` |
| Flow | `Event_Auto_Link_Related_Records` |
| Apex Class | `InvestmentsTriggerHelper` |
| Lightning Web Component Bundle | `newGeneralTaskForm` |
| Lightning Web Component Bundle | `newGeneralTaskAction` |
| Quick Action | `Account.New_General_Task_1` |
| Record Type (only if missing in Prod) | `Task.General_Task2` |

**Pre-deploy checks**
1. Does `Event_Auto_Link_Related_Records` exist in Production? If not, this release adds brand-new Event auto-linking. Tell users.
2. Compare the two hardcoded record type Ids (section 6) with Production.
3. Flows may land **Inactive**. *Deploy processes and flows as active* was found switched off on 10 Sep 2026, and the client had to activate flows by hand after a later change set ([04](../04-investment-task-popup-fix/README.md)). Plan to activate them manually.

**In Production:** Validate → *Run specified tests* → `TestinvestmentsTrigger` → Quick Deploy.

_Screenshot in the private copy: 03-change-set-components.png (Change set components)._

---

## 10. Post-deployment

1. **Check the record type Ids.** Setup → Object Manager → Household & Individual → Record Types → open **Household (GVA)** and **Business Household**, and copy the Id from the address bar. If either differs, update **Get Account For ACR** (conditions 2 and 3) and **Get Household** in **both** flows (6 values in total), then **Save As New Version → Activate**.

   _Screenshot in the private copy: 04-record-type-id-url.png (Record type Id in the URL)._

2. **Confirm both flows are Active.** Setup → Flows → *Task – Auto-Link Related Records* and *Event - Auto Link Related Records*. **Don't activate** *Event – Auto-Link Household*.

   _Screenshot in the private copy: 05-flows-active.png (Flows active)._

3. Add the New General Task button to pages and assign the record type. See [01](../01-new-general-task-button/README.md), section 10.

**Rollback:** Setup → Flows → the flow → *View Details and Versions* → activate the previous version. This goes back to create-only behaviour but does not undo Related To values already written. Reverting the Client Name re-map needs a developer, because it's Apex.

---

## 11. Lessons and gotchas

- **Flows that change Related To on update need a guard.** Without *Safe to Set Related To?*, any Name edit would wipe a Related To the user picked deliberately.
- **Hardcoded record type Ids fail silently.** Prefer `RecordType.DeveloperName`, as the new decision does.
- **`InvestmentsTriggerHelper` is shared.** It has been changed again by later releases (Orion Rep ID [09](../09-orion-rep-id-sync/README.md), Account Funded [13](../13-account-funded-date-all-accounts/README.md), Orion SSN [16](../16-orion-ssn-birthday-apex/README.md)). Deploying any copy replaces the whole Production file, so always deploy the latest sandbox version and validate first.
- **The Investment Team Email depends on Related To.** It's looked up from the Financial Account in Related To. If auto-link puts a Household there instead, the email is blank. See [12](../12-investment-team-email-blank/README.md).
- **Failing tests elsewhere in the org** block *Run local tests*. Always use specified tests here (org-wide coverage was ~33% in the sandbox in Sep 2026).

---

## 12. Seeing a similar issue?

- *"Related To didn't update when I changed the Name"* → check the flow is Active and on the latest version, and that the current Related To is a Household, Business Household or Individual. Anything else is protected on purpose.
- *"Related To is never filled in"* → compare the hardcoded record type Ids with the org.
- *"Changing Client Name on a Financial Account didn't change the Task's Name"* → the new Client Name must be a Person Account (it needs a `PersonContactId`). Clearing Client Name deliberately leaves Tasks alone. Recurring Tasks are skipped.

## Files

- `Task_Auto_Link_Related_Records.flow-meta.xml`
- `Event_Auto_Link_Related_Records.flow-meta.xml`
- `InvestmentsTriggerHelper.cls` (`handlerAfterUpdate`)
- `investmentsTrigger.trigger` (unchanged, calls the helper)
- Manifest: `manifest/package.autoLinkRelease.xml`
- Original post-deploy guide: `docs/Post-Deployment-Steps-AutoLink-Release.md`
