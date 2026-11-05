# Investment Task Popup Fix ("Ready to Trade" flips the task and notifies the trading team)

| | |
|---|---|
| **Status** | Deployed to Production 10 Sep 2026, possibly **incomplete**: the flip flow `change_recordtype` was not in the uploaded change set. Unless Production already had v2 from an earlier change set (not confirmed), the flip can't run there. This is one of two candidate causes of the Production failure in [11](../11-ready-to-trade-not-working-prod/README.md) |
| **Requested** | TBD (not found in records) |
| **Built** | 28 Aug 2026 (sandbox) |
| **Change set review** | 9 Sep 2026. Client deployed 10 Sep. Follow-up about the Process Builder on 10–11 Sep |
| **Objects** | Task (record types **General Task** `General_Task2` → **Investment Task** `General_Task`), Activity (new field) |
| **Type** | LWC rewrite + before-save flow + after-save email flow + new field. Plus one Process Builder added by mistake |
| **Change set** | `Investment Task Popup Fix` |
| **Test class** | `investmentTaskControllerTest` |

---

## 1. Summary

Before this change, when a **General Task** reached **Ready to Trade**, the `notifyInvestments` LWC showed a **Yes/No popup**. Clicking Yes called Apex (`InvestmentTaskController.createInvestmentTask`), which switched the record type to **Investment Task**. The old version polled Apex every 2 seconds. The `change_recordtype` flow that was meant to automate this had never run: it used hardcoded record type Ids that don't exist in the org, and it ran on Create only.

We replaced the popup with automation:

1. **`change_recordtype`** (rewritten, label *Task: Flip to Investment on Ready to Trade*): on save with Status = Ready to Trade, switches the record type to Investment Task and stamps **`Trading_Team_Notified_On__c`**.
2. **`Task_Notify_Trading_Team_On_Ready_To_Trade`** (new): sends **one** email to the investment team when that stamp goes from blank to filled.
3. **`notifyInvestments`** (rewritten): renders nothing and makes no Apex calls. It only shows a "Trading team notified" toast when it sees the record type switch.

> **Record type names are swapped on Task.** `General_Task2` = label **General Task**. `General_Task` = label **Investment Task**.

---

## 2. Client messages (as received)

The original request: TBD (not found in records). The build happened on 28 Aug, before the earliest session transcript we have.

**After deployment (10 Sep), reply to our change-set message:**
> Deployed and activated task flow. Update investment field of task from investment field of investment object flow says it came over but I am not seeing it under flows. Is the original version called something else?
>
> I do not see this flow name in the sandbox either

**Later (21 Sep), the report that led to [11](../11-ready-to-trade-not-working-prod/README.md):**
> I had a user use the new task flip logic for "Ready to Trade" the pop up did not occur which is great but the task did not flip to investment nor notify the trading team. When we pushed this update could you remind me what needed to be updated/turned on. Perhaps I missed turning an aspect on in production.

---

## 3. Scope

**In scope**
- Switch General Task → Investment Task automatically on **Ready to Trade**
- Notify the trading team **exactly once** per task
- Remove the popup and its Apex polling

**Out of scope**
- The Investment Team Email itself. Both flows need `Investment_Team_Email__c` filled in. Why it was often blank is covered in [12](../12-investment-team-email-blank/README.md).
- Page layout and FLS changes. `Trading Team Notified On` is a system field and deliberately isn't on any layout.

---

## 4. Background: what changed in the sandbox

All five items below had changed in the org, and the local repo was stale (last pulled 17 Jun 2026). We retrieved them on 9 Sep before reviewing the change set.

| Component | Change |
|---|---|
| `notifyInvestments` (LWC) | Rewritten. The old version polled `getTaskForLWC` every 2s, showed a Yes/No modal and called `createInvestmentTask` / `markPopupShown`. The new one wires `getRecord` on `Task.RecordType.DeveloperName` and fires a sticky success toast on a `General_Task2 → General_Task` switch. No Apex, no `GeneralTaskRecordTypeId` label import. |
| `change_recordtype` (Flow) | Rewritten in place (the description says 2026-08-28). Now **before-save, CreateAndUpdate**, looks the record type up by DeveloperName, and stamps the new field. The old version had two hardcoded record type Ids, which don't exist in this org, and ran on Create only, so it never executed. |
| `Task_Notify_Trading_Team_On_Ready_To_Trade` (Flow) | **New.** After-save `emailSimple` to `$Record.Investment_Team_Email__c`, from the org-wide no-reply address |
| `Trading_Team_Notified_On__c` | **New** DateTime on **Activity** (not Task) |
| `Update_investment_field_of_task_from_investment_field_of_investment_object` | A **Process Builder**, not a flow. Criteria fix only (see section 6). |
| `Task_Flip_To_Investment_On_Ready_To_Trade` | Has the same logic and the **same label** as the rewritten `change_recordtype`, but its status is **Obsolete** with no active version. A dead duplicate: do not ship it. |

---

## 5. Root cause

| Problem | Cause | Fix |
|---|---|---|
| The record type switch depended on a user clicking a popup | Built as an LWC + Apex prompt, with polling | Before-save flow switches the record type automatically |
| The existing `change_recordtype` flow did nothing | Hardcoded record type Ids that don't exist + Create-only trigger | Look the record type up by DeveloperName, run on Create and Update |
| Risk of repeat emails on every save | No send-once marker | `Trading_Team_Notified_On__c` is stamped once. The email flow fires only when it goes from blank to filled. |

---

## 6. Solution design

### Components

| Type | Name | Role |
|---|---|---|
| Flow | `change_recordtype` (*Task: Flip to Investment on Ready to Trade*) | Before-save. Entry: `Status = "Ready to Trade"` AND record type `General_Task2` AND `Trading_Team_Notified_On__c` blank AND `Investment_Team_Email__c` not blank. Sets `RecordTypeId` = Investment Task and stamps `Trading_Team_Notified_On__c = NOW`. |
| Flow | `Task_Notify_Trading_Team_On_Ready_To_Trade` | After-save. Entry: stamp not blank AND email not blank, **only when changed to meet the criteria**. Sends one email: subject `Ready to Trade - <Subject>`. Does no record update, so it can't re-trigger anything. |
| Custom Field | `Activity.Trading_Team_Notified_On__c` (DateTime) | Send-once guard. "Blank means not yet notified." |
| LWC | `notifyInvestments` | Toast only. Placed on `Task_Record_Page1`. |
| Process Builder | `Update_investment_field_of_task_from_investment_field_of_investment_object` | Criteria of the *Investment Id is Populated* node changed to `!ISNULL([Task].WhatId) && !ISBLANK([Task].WhatId) && (ISNEW() \|\| ISCHANGED([Task].WhatId))`, so its *Populate Team Email* subflow (*Look Up Investment Email From Rep Id on Investment*) runs only on create or when Related To changes. The client linked this to a **duplicate email issue (multiple sends on the same task)** that they had seen only in the sandbox. |
| Apex Class | `InvestmentTaskController` + `investmentTaskControllerTest` | No longer called by anything for this feature. `resetPopupShown()` was added in the sandbox. Shipped anyway so Production matches the sandbox. |

### How it works

```
User sets Status = "Ready to Trade" on a General Task and saves
   └─ change_recordtype (before save, same transaction, no extra DML)
        → RecordTypeId = Investment Task (General_Task)
        → Trading_Team_Notified_On__c = now
   └─ Task_Notify_Trading_Team_On_Ready_To_Trade (after save)
        → stamp went blank → filled → send ONE email to Investment_Team_Email__c
   └─ notifyInvestments (on the open record page)
        → sees General_Task2 → General_Task → sticky toast "Trading team notified"
Saving again: stamp already set → neither flow fires → no second email
```

---

## 7. How to test in the sandbox

1. Open a **General Task** whose **Related To** is a Financial Account and whose **Investment Team Email** is filled in. If the email is blank, nothing happens, by design.
2. Set **Status = Ready to Trade** and Save.
3. Expected:
   - the record type is now **Investment Task**
   - **Trading Team Notified On** has the current date and time
   - **one** email reaches the investment team address
   - a success toast may appear (see the gotcha in section 11)
4. Save the task again. **No** second email.

_Screenshot in the private copy: 01-task-flipped.png (Task after Ready to Trade)._
_Screenshot in the private copy: 02-trading-team-email.png (Trading team email)._

---

## 8. Test classes and coverage

| Component | Coverage (`investmentTaskControllerTest`, 4/4 passing in the sandbox, 9 Sep 2026) |
|---|---|
| `InvestmentTaskController` | 90% (lines 38–39 uncovered) |

Tests: `testCreateInvestmentTask`, `testMarkPopupShown`, `testResetPopupShown`, `testGetTaskForLWC`. The flows and the LWC need no Apex test. The Apex is optional for this fix, but including it means Production must run tests, so use *Run specified tests*.

---

## 9. Deployment (change set)

**Change set name:** `Investment Task Popup Fix` (created 24 Aug 2026 with 3 components, extended on 9 Sep)

**What was actually uploaded (6 components):**

| Component Type | Name |
|---|---|
| Apex Class | `InvestmentTaskController` |
| Apex Class | `investmentTaskControllerTest` |
| Lightning Web Component Bundle | `notifyInvestments` |
| Custom Field (Activity) | `Trading_Team_Notified_On__c` |
| Flow Definition | `Task_Notify_Trading_Team_On_Ready_To_Trade` |
| Flow Definition | `Update_investment_field_of_task_from_investment_field_of_investment_object` ← **added by mistake** |

**What should also have been in it:** `change_recordtype`. An earlier draft contained `Task_Flip_To_Investment_On_Ready_To_Trade` instead. That's the obsolete duplicate with the same label. It was correctly removed, but `change_recordtype` was never added. We flagged this on 9 Sep ("the 'change recordtype' flow is not in this set … until then the notification won't fire"), but the note did not make it into the final message to the client.

**Message sent to the client (9 Sep):**
> **Change Set:** Investment Task Popup Fix
> **Test Class:** investmentTaskControllerTest
> Please activate the following flows:
> • Task: Notify Trading Team on Ready to Trade
> • Update investment field of task from investment field of investment object

_Screenshot in the private copy: 03-change-set-components.png (Final change set components)._

---

## 10. Post-deployment

1. **Activate `Task: Notify Trading Team on Ready to Trade`** (Setup → Flows). The client did this on 10 Sep. *Deploy processes and flows as active* is **off** in this org, so flows and processes always arrive as a new **inactive** version.
2. **Deploy and activate `change_recordtype`.** Without it, nothing stamps `Trading_Team_Notified_On__c`, and Production keeps the old version that never ran. This is the gap found in [11](../11-ready-to-trade-not-working-prod/README.md). In Setup → Flows, search by **API name**: the label matches the obsolete duplicate's label.
3. **The Process Builder: no action.** Our final reply to the client (11 Sep):
   > That one's a Process Builder, not a Flow, and it relates to the duplicate email issue (multiple sends on the same task) — which you pointed out was only happening in the sandbox. I added it to the change set by mistake, so no action needed on it.
   >
   > If you do see duplicate emails on the same task in Production, you can activate it then: Setup → Process Automation → Process Builder → "Update investment field of task from investment field of investment object".

   Production keeps running its previous version of that process, so the `ISNEW() || ISCHANGED(WhatId)` criteria fix is **not live** there.
4. **Org-wide email address.** The org-wide no-reply address must exist and be verified in Production. Change sets don't carry org-wide addresses.

---

## 11. Lessons and gotchas

- **Identical labels, different API names.** `change_recordtype` and `Task_Flip_To_Investment_On_Ready_To_Trade` are both labelled *Task: Flip to Investment on Ready to Trade*. Pick components by **API name**, and check the status (Active vs Obsolete) before adding them.
- **An LWC goes live on deploy, but a flow doesn't.** The change set removed the old popup (the LWC took effect immediately) while the replacement flows arrived inactive or missing. Result: no popup, no switch, no email, and no error.
- **Process Builder ≠ Flow.** A `<processType>Workflow</processType>` file is a Process Builder. It's under Setup → Process Automation → Process Builder, not Setup → Flows. Like flows, it does not arrive already active.
- **The toast is unreliable.** On `Task_Record_Page1`, `notifyInvestments` has a visibility rule `RecordType.DeveloperName = General_Task2`. As soon as the record type switches, the rule hides the component, so it races its own removal (found 21 Sep). Treat the email as the real notification.
- **Both flows need `Investment_Team_Email__c`.** It comes from the Rep ID of the Financial Account in **Related To**. If Related To is a Household or blank, there's no switch and no email ([12](../12-investment-team-email-blank/README.md)).
- **Retrieve before reviewing a change set.** The local repo was two months behind the sandbox.
- **Use a stamp field as the send-once guard**, and trigger the email only when the stamp changes from blank to filled.

---

## 12. Seeing a similar issue?

- *"Popup gone, but the task didn't switch and nobody was emailed"* → check `change_recordtype` (by API name) and `Task_Notify_Trading_Team_On_Ready_To_Trade` are **Active** on the latest version in that org. Then check Investment Team Email is filled in. See [11](../11-ready-to-trade-not-working-prod/README.md).
- *"Can't find a flow the change set says it deployed"* → it may be a Process Builder. Check `<processType>` in the metadata.
- *"Trading team gets the email twice"* → check `Trading_Team_Notified_On__c` is being stamped, and look at the Process Builder criteria fix above.
- *"Toast didn't show"* → expected. See the visibility-rule gotcha.

## Files

- `notifyInvestments/`
- `change_recordtype.flow-meta.xml`
- `Task_Notify_Trading_Team_On_Ready_To_Trade.flow-meta.xml`
- `Trading_Team_Notified_On__c.field-meta.xml`
- `Update_investment_field_of_task_from_investment_field_of_investment_object.flow-meta.xml` (Process Builder. The local copy now shows status `Obsolete`. On 10 Sep 2026 the sandbox had version 6 active, so check the org before relying on either.)
- Obsolete duplicate, do not deploy: `Task_Flip_To_Investment_On_Ready_To_Trade.flow-meta.xml`
- `InvestmentTaskController.cls`, `investmentTaskControllerTest.cls`
- Page: `Task_Record_Page1.flexipage-meta.xml`
