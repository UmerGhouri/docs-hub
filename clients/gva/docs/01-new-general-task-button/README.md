# New General Task Button (Custom LWC Form)

| | |
|---|---|
| **Status** | Built and tested in the sandbox. Production deploy date: TBD (not found in records) |
| **Requested** | TBD (not found in records). Work started around 29 Jul 2026 |
| **Built** | 29–30 Jul 2026. Updated 5 Aug (form waits for all fields to load) and 11 Aug 2026 (Assigned To can be a queue) |
| **Objects** | Task (record type **General Task**, API `General_Task2`), Account (Household & Individual pages) |
| **Type** | Lightning Web Components + LWC quick action + small Apex controller |
| **Replaces** | The URL custom button **New General Task** on the Household/Individual pages (`/lightning/o/Task/new?recordTypeId=…&nooverride=1`) |
| **Change set** | TBD (not found in records). The repo has a CLI manifest: `manifest/package.newGeneralTask.xml` |
| **Test class** | `TaskAssigneeControllerTest` |

---

## 1. Summary

The Household and Individual pages had a **New General Task** URL button that opened Salesforce's standard New Task page with the General Task record type. We replaced it with a custom LWC form in a quick action, **New General Task** (`Account.New_General_Task_1`). The form opens in a modal titled **New Task: General Task** and:

- prefills **Related To** with the record you launched it from, and **Assigned To** with the current user
- lets **Subject** be free text or a value picked from the Task Subject list
- builds **recurring task series** itself (daily, weekly, monthly or yearly), with the same options and limits as the standard form
- has **Save & New**
- (from 11 Aug) lets **Assigned To** be a user or a **queue**, showing only queues that support Task

---

## 2. Client request (as received)

TBD (not found in records). This work came before the earliest session transcript we have (2 Sep 2026), and the original Slack request was not saved. What we know about the goal comes from the setup doc we wrote at the time, `docs/New-General-Task-Button-Setup.md` (30 Jul 2026).

---

## 3. Scope

**In scope**
- A reusable form component (`newGeneralTaskForm`) and a quick-action wrapper (`newGeneralTaskAction`)
- The quick action on **Account** (`New_General_Task_1`), used on the Household and Individual record pages
- An Apex method that lists the queues that can own a Task (`TaskAssigneeController`)

**Out of scope**
- Record pages (FlexiPages) and page layouts. We left them out of the deployment on purpose, because deploying them overwrites the Production copies. Adding the button to a page is a manual post-deploy step (see section 10).
- The **General Task Queue** queue. It was created in the sandbox for testing only. In Production, add Task to the *Supported Objects* of whichever existing queues should receive tasks.
- The Financial Account page. It still uses the old URL button `Investments__c.New_General_Task` (see section 11).
- Contact and Lead pages. Not supported (see section 6).

---

## 4. Background: which "New General Task" is which

The org has several actions with the same label. This causes most of the support questions.

| Where | What it is | Form that opens |
|---|---|---|
| Household / Individual page, **Object-Specific Quick Actions → New General Task** | `Account.New_General_Task_1` → `newGeneralTaskAction` → `newGeneralTaskForm` | **Custom form (this issue)** |
| **Custom Buttons → New General Task** (Account and `Investments__c`) | URL button `/lightning/o/Task/new?recordTypeId=…&nooverride=1` | Standard Salesforce New Task page |
| Activity tab on the Household layout | Standard "Create a Record" quick action `NewGeneralTask` | Standard quick-action layout. It has no recurrence pattern fields. |
| Global Quick Actions → New General Task | Older global actions | Not this one |

> **Record type names are swapped on Task.** `General_Task2` is labelled **General Task**, and `General_Task` is labelled **Investment Task**. Always check the DeveloperName, not the label.

---

## 5. Root cause

TBD: we have no record of what the client wanted changed about the standard form. The design notes in the code and the setup doc point to these goals:

- Prefill Related To and Assigned To from the launching record and the current user
- Allow a free-text Subject as well as the picklist values
- Support recurring series from the same button
- Allow assigning to a queue (added 11 Aug)

---

## 6. Solution design

### Components

| Type | Name | Role |
|---|---|---|
| LWC | `newGeneralTaskForm` | The whole form. Not exposed (`isExposed=false`). Shared by the quick action and the record-page button. |
| LWC | `newGeneralTaskAction` | Quick-action screen (`lightning__RecordAction`). Passes `recordId` to the form and closes the screen on success or cancel. |
| Quick Action | `Account.New_General_Task_1` (label **New General Task**) | `LightningWebComponent` type, points to `newGeneralTaskAction` |
| Apex Class | `TaskAssigneeController` | `getTaskQueues()` returns queues that have Task in their Supported Objects (`QueueSobject.SobjectType = 'Task'`) |
| Test Class | `TaskAssigneeControllerTest` | 1 test. A Task queue is returned and a Case-only queue is filtered out. |
| Record Type | `Task.General_Task2` | Must exist in the target org. Not in the manifest (see section 9). |
| LWC (not deployed) | `newGeneralTaskButton`, `newGeneralTaskModal` | A record-page button + `LightningModal` version of the same form. In the repo but **not** in either deployment manifest. |

### How it works

```
User clicks "New General Task" (quick action on Household / Individual page)
   └─ newGeneralTaskAction  (recordId supplied by the quick action)
        └─ newGeneralTaskForm
             ├─ getObjectInfo(Task) → finds record type General_Task2
             │     (if it's missing, falls back to the default record type, with NO error)
             ├─ getPicklistValues(Subject) for that record type
             ├─ getTaskQueues() (Apex) → queue list for Assigned To
             ├─ lightning-record-edit-form for Priority, Type, Status, Name, Related To, Comments
             └─ Save → collects the fields, adds Subject, OwnerId, RecordTypeId, reminder,
                       recurrence fields → submit()
```

### Business rules worth knowing

| Rule | Detail |
|---|---|
| Related To | The launching record's Id is written to `Task.WhatId`. The object must be valid for WhatId. **Contact and Lead are not**, so the save fails there. |
| Record type | Looked up by DeveloperName `General_Task2`. If the user's profile can't see it, the form silently uses the default record type and shows the wrong Type/Status/Priority values. |
| Recurrence caps | A series is capped by number of occurrences per frequency: Daily 100, Weekly 53, Monthly 60, Yearly 10. **Calculate Latest Date** sets the furthest valid End Date. |
| Yearly | `RecursYearly` / `RecursYearlyNth` don't accept `RecurrenceInterval`, so there's no "Every Other Year" option. |
| Loading | The body stays hidden until `lightning-record-edit-form` fires `load`, with a 5-second fallback. Before 5 Aug, some fields showed only after the user clicked into the form. |

### Supported objects for a new quick action

Account (including Person Accounts), Case, Opportunity, Campaign, Contract, Asset, Product2, `Investments__c`, `Tickets__c`, `Household__c`, `Financial_Plans__c`, `Insurance__c`, `Office__c`, `Rep_ID__c`, `Correspondence__c`, `Compliance_Review__c`, `Know_Your_Client__c`, `Disbursements__c`, `Business_Process__c`, `Client_Interests__c`, `FeeBased_401k__c`.

To add the button to another object: Object Manager → the object → Buttons, Links, and Actions → **New Action** → Action Type *Lightning Web Component*, component `c:newGeneralTaskAction`, label `New General Task`, name `New_General_Task_1`.

---

## 7. How to test in the sandbox

1. Open a **Households & Individuals** record and click **New General Task** in the action bar.
   - The modal title is **New Task: General Task**.
   - **All fields show right away**, without clicking into the form.
   - **Related To** shows the record. **Assigned To** shows your name.

   _Screenshot in the private copy: 01-new-task-modal.png (New Task: General Task modal)._

2. Type a Subject that is not in the list, fill in Due Date, and **Save**. The task appears under the record's activities with the **General Task** record type.
3. Open it again, pick a Subject from the list, and click **Save & New**. The modal stays open and the form is cleared.
4. Tick **Create Recurring Series of Tasks**, choose **Weekly**, and set an End Date more than 53 weeks away. Save is rejected. Click **Calculate Latest Date** and save again.

   _Screenshot in the private copy: 02-recurring-series.png (Recurring series options)._

5. Switch **Assigned To** to **Queues**. Only queues that support Task are listed. Pick one and save.

   _Screenshot in the private copy: 03-queue-assignee.png (Queue assignee)._

---

## 8. Test classes and coverage

| Component | Test class | Coverage |
|---|---|---|
| `TaskAssigneeController` | `TaskAssigneeControllerTest` | TBD (not found in records) |

The LWCs and the quick action need no Apex tests. Note that the setup doc (`docs/New-General-Task-Button-Setup.md`) says "No Apex, so no test classes run". That was written on 30 Jul, **before** the queue feature added `TaskAssigneeController` on 11 Aug. It is out of date: the current deployment includes Apex and needs the test class.

---

## 9. Deployment

**Change set name:** TBD (not found in records).

Components (from `manifest/package.newGeneralTask.xml`, 13 Aug 2026):

| Component Type | Name |
|---|---|
| Lightning Web Component Bundle | `newGeneralTaskForm` |
| Lightning Web Component Bundle | `newGeneralTaskAction` |
| Apex Class | `TaskAssigneeController` |
| Apex Class | `TaskAssigneeControllerTest` |
| Quick Action | `Account.New_General_Task_1` |

- **Leave out** the Lightning record pages and the page layouts. They would overwrite Production.
- **Leave out** the sandbox-only *General Task Queue*.
- `Task.General_Task2` record type: only add it if it is **missing** in Production. In the sandbox its Id carries the sandbox org's instance prefix, so we first suspected it was created in the sandbox. The legacy URL buttons use the same Id, though, and a later check (21 Sep) treated `General_Task2` as already in Production. Check Setup → Object Manager → Task → Record Types before you add it.
- The form changes also shipped in the Auto-Link release (`manifest/package.autoLinkRelease.xml`). See [02](../02-task-event-auto-link-related-records/README.md).

**In Production:** Validate with *Run specified tests* → `TaskAssigneeControllerTest`, then Quick Deploy.

_Screenshot in the private copy: 04-change-set-components.png (Change set components)._

---

## 10. Post-deployment

1. **Check the record type.** Setup → Object Manager → Task → Record Types → **General Task** exists, and is assigned to the profiles that will use the button (Profile → Record Type Settings → Tasks → Edit → add *General Task*).

   _Screenshot in the private copy: 05-profile-record-types.png (Profile record type settings)._

2. **Add the button to each record page** (it isn't added automatically):
   1. Open a Households & Individuals record → gear → **Edit Page**
   2. Click the **Highlights Panel** → **Add Action** → type `new ge`
   3. Pick **Object-Specific Quick Actions → New General Task**. Picking the *Custom Buttons* or *Global* entry gives you the old form, with no error.
   4. **Done** → drag it into position → **Save** (and **Activation…** if the page was never activated) → hard refresh (Ctrl+Shift+R)

   _Screenshot in the private copy: 06-pick-correct-action.png (Pick the Object-Specific Quick Action)._

3. If the page has no editable Actions list (no dynamic actions), add it under Page Layout → **Salesforce Mobile and Lightning Experience Actions** instead.
4. **Queues:** add Task to *Supported Objects* on any Production queue that should appear in Assigned To.

---

## 11. Lessons and gotchas

- **Same label, three different actions.** Most "it opens the old form" reports come from picking the wrong **New General Task** in App Builder.
- **A missing record type fails silently.** `getObjectInfo` falls back to the default record type, and the user sees the wrong picklist values with no error.
- **Visible Action Buttons.** If the Highlights Panel shows 4 buttons and you add a fifth, it is hidden behind the ▾ overflow.
- **Known gap (found 3 Sep 2026, not fixed as far as our records show):** all active Task validation rules are driven by **Subject**, not record type. `Withdrawal_Type_Required`, `Withdrawal_Amt_Required`, `Transaction_Value_Positive` and `Transaction_Value_Negative` need `Withdrawal_Type__c` or `Transaction_Value__c`, and `Require_OtherModelName` needs `Other_Model_Name__c`. The form only has Description, Priority, Status, Type, WhatId and WhoId, so choosing Subject = *Withdrawal* (for example) gives a save error the user can't fix.
- **Financial Account page still has the URL button.** `Investments__c.New_General_Task` is a URL button and doesn't work in the mobile app. The fix is to create the same quick action on `Investments__c` pointing at `c:newGeneralTaskAction`. Fix the validation-rule gap above first. Related: [05](../05-new-investment-task-mobile/README.md).
- **The Activity tab can't do recurrence.** The standard `NewGeneralTask` quick action shows only the `IsRecurrence` checkbox, with no pattern fields. Users who need a recurring series must use this button.

---

## 12. Seeing a similar issue?

- *"Clicking New General Task opens the standard form"* → the wrong action was added to the page. See section 10, step 2.
- *"Type/Status values are wrong in the form"* → `General_Task2` is missing or not assigned to the user's profile.
- *"Save fails with a Related To error"* → the object isn't valid for `Task.WhatId` (for example, Contact or Lead).
- *"Save fails asking for Transaction Value / Withdrawal Type"* → this is the validation-rule gap in section 11.
- *"Recurring save rejected on End Date"* → the occurrence cap. Use **Calculate Latest Date**.
- Need the same form on another object → create a new LWC quick action pointing at `c:newGeneralTaskAction` (section 6).

## Files

- `newGeneralTaskForm/`
- `newGeneralTaskAction/`
- `Account.New_General_Task_1.quickAction-meta.xml`
- `TaskAssigneeController.cls`, `TaskAssigneeControllerTest.cls`
- `General_Task2.recordType-meta.xml`
- Not deployed: `newGeneralTaskButton/`, `newGeneralTaskModal/`
- Manifest: `manifest/package.newGeneralTask.xml`
- Original setup notes: `docs/New-General-Task-Button-Setup.md`
