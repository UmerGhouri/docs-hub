# New Investment Task Button on Mobile (Screen Flow + LWC)

| | |
|---|---|
| **Status** | Built, tested and active in the sandbox (3 Sep 2026). Production deployment handed to the client on 15 Sep 2026. Production outcome: TBD — not found in records |
| **Requested** | TBD — not found in records (work started 2 Sep 2026) |
| **Delivered** | 3 Sep 2026 (sandbox), change set prepared 8 Sep 2026 |
| **Objects** | `Investments__c` (Financial Account), Task (record type *Investment Task*, developer name `General_Task`) |
| **Type** | Screen flow + flow quick action, existing LWC changed to run inside a flow |
| **Replaces** | Nothing. The old LWC quick action `Investments__c.Investment_Task` is **kept** next to the new one |
| **Change set** | `Investment Task Mobile` (failed in Production, see [06](../06-change-set-savepoint-error/README.md)) |
| **Test classes** | None. There is no Apex |

---

## 1. Summary

The **New Investment Task** button on Financial Account records only worked on desktop. It is an LWC quick action (`investmentReminder`) that sends the user to a hard-coded desktop URL (`/lightning/o/Task/new?...&nooverride=1`). The Salesforce mobile app does not open that route, so on a phone the button did nothing useful.

We added a second button with the same label that launches a new screen flow, `New_Investment_Task`. The flow reads the Financial Account and passes the values into the same `investmentReminder` LWC. The LWC now uses a page reference (`standard__objectPage`) instead of a URL, so the platform builds the right route for desktop or mobile. The old button stays where it was.

---

## 2. Client request (as received)

> Could you also make sure that the custom investment task button on the financial accounts is available on the mobile app please.

Reply sent on 3 Sep:
> The Investment Task button on Financial Accounts is now available on the phone.

---

## 3. Scope

**In scope**
- The **New Investment Task** button on the Financial Account (`Investments__c`) record page
- Keep the 2pm Eastern trading cut-off reminder for GVA Asset Management (GVAAM) accounts
- Keep the standard, layout-driven *New Task: Investment Task* page as the form the user ends up on

**Out of scope**
- The **New General Task** button beside it. It is a URL custom button (WebLink) and cannot be made to work on mobile in place. It needs its own replacement (see §11).
- Removing the old LWC quick action. The client asked to add the new button beside it.
- Changing the Task page layout or any Task automation

---

## 4. Background: what the button actually does

"Financial Account" is the label of the custom object `Investments__c`, set by object translation.

On click, nothing custom renders a form:

1. The user taps **New Investment Task**. This is quick action `Investments__c.Investment_Task`, type *Lightning Web Component*, pointing at `investmentReminder`. It is a quick action, not a custom button.
2. `investmentReminder` reads one field, `GVA_Asset_Management_Accounts__c` (checkbox, label *GVA Asset Management*).
3. If the account is GVAAM **and** it is 2pm–11:59pm Eastern, it shows a small *Reminder* about the trading cut-off. Otherwise it skips straight on.
4. It redirects to the **standard** Task create page for record type *Investment Task* (Id from custom label `InvestmentRecordTypeId`), with *Related To* pre-filled with the Financial Account.

The form the user sees is Salesforce's own create page, built from the page layout **Task Layout**. To change a field on it, edit the layout, not code. The LWC does not call any Apex.

After the Task is saved, all of the trade-request automation (the `PopulateFirmIdOnTask` trigger, the Ready-to-Trade flows, the notification flows and the workflow rules) runs on the server, the same on any device. The mobile change does not affect any of it.

---

## 5. Root cause

| Check | Result |
|---|---|
| Is the Lightning record page activated for phone? | Yes. `View` → `Investment_Record_Page` is assigned for form factor **Small** and **Large** |
| Does the action bar show on phone? | Yes. The highlights panel has `enableActionsInNative = true` |
| Does the LWC restrict form factors? | No. There is no `supportedFormFactors` block |
| Is there a device setting on the quick action? | No. `QuickAction` metadata has no form-factor field |
| **What breaks** | The LWC navigates with `type: 'standard__webPage'` to a hard-coded `/lightning/o/Task/new?...` URL. The mobile app does not resolve that desktop route. |

So the button already appeared on phones. It broke **after** the tap. It's a code problem, not a config toggle.

Two smaller problems in the same component, both fixed:
- The Eastern-time hour was worked out with `new Date(now.toLocaleString(...))`. Some mobile WebViews return `Invalid Date` for that, so the hour is `NaN` and the reminder never shows.
- The CSS set `.slds-modal__container { width: 1000px }` and hid desktop Aura wrapper classes (`.forceModal`, `.uiModal`, ...). These don't fit a phone.

---

## 6. Solution design

### Options considered

| Option | Verdict |
|---|---|
| Fix only the navigation line in the LWC, keep the LWC quick action | Smallest change. Was recommended first. |
| Replace with a standard *Create a Record* quick action | No code, but loses the 2pm reminder and the full page layout |
| Rebuild the whole form in a screen flow | Loses the layout-driven form, native recurrence, and inline validation errors. Too much work. |
| **Chosen (client's call):** new screen flow that hosts the existing LWC, launched by a new flow quick action, placed beside the old button | Done. The flow reads the record and passes the values in. The LWC still does the time check and the navigation. |

### Components

| Type | Name | Role |
|---|---|---|
| Flow (Screen) | `New_Investment_Task` | **New.** Input `recordId` → *Get Investment* → *Investment Found?* → screen with `c:investmentReminder`. A fault or not-found path goes to an error screen. |
| Quick Action | `Investments__c.New_Investment_Task_Flow` | **New.** Type *Flow*, label **New Investment Task** |
| LWC | `investmentReminder` | **Changed.** Exposed to flow screens, takes 3 inputs, mobile-safe navigation |
| Flexipage | `Investment_Record_Page` | Changed **in the sandbox only** to add the new action. Not deployed to Production. Add the action there by hand. |
| Custom Label | `InvestmentRecordTypeId` | Existing. Value = the *Investment Task* record type Id. Read by the flow and the LWC. |
| Field | `Investments__c.GVA_Asset_Management_Accounts__c` | Existing. Read by the flow and imported by the LWC. |

### How it works

```
New Investment Task (Flow quick action)
   └─ Flow New_Investment_Task
        ├─ Get Investment (Id, GVA_Asset_Management_Accounts__c)  ── fault/not found ──> "Cannot Open" screen
        └─ Screen (no header/footer) → c:investmentReminder
             inputs: recordId, recordTypeId = $Label.InvestmentRecordTypeId,
                     gvaAssetManagement = Get_Investment.GVA_Asset_Management_Accounts__c
             └─ GVAAM and hour >= 14 Eastern → Reminder → Acknowledged
             └─ navigate: standard__objectPage Task/new, recordTypeId, WhatId = recordId
             └─ FlowNavigationFinishEvent (closes the flow)
```

### The LWC serves both buttons

The client kept the old button, so the component has to work from both entry points. It checks whether `recordTypeId` was passed in:

| | New button (flow) | Old button (LWC quick action) |
|---|---|---|
| `recordId` | flow input | injected by the action |
| `recordTypeId` | flow passes the label | falls back to the same label |
| `gvaAssetManagement` | flow's Get Records | read in the LWC with `@wire(getRecord)` |
| What starts routing | `renderedCallback` | the wire callback |
| How it closes | `FlowNavigationFinishEvent` | `CloseActionScreenEvent` |

`close()` sends both events. Each one does nothing in the context it doesn't belong to. A `hasRouted` flag makes sure routing happens only once.

The original plan was to move `handleActionClick()` into `connectedCallback`. We used `renderedCallback` with the `hasRouted` guard instead, because calling navigation before the first render is unreliable.

The key change, the navigation:

```js
this[NavigationMixin.Navigate]({
    type: 'standard__objectPage',
    attributes: { objectApiName: 'Task', actionName: 'new' },
    state: {
        nooverride: '1',
        recordTypeId,
        defaultFieldValues: `WhatId=${this.recordId}`
    }
});
```

Don't wrap `defaultFieldValues` in `encodeURIComponent`. The navigation service encodes `state` values itself.

The Eastern hour, which handles daylight saving:

```js
const hour = Number(new Intl.DateTimeFormat('en-US', {
    timeZone: 'America/New_York', hour: 'numeric', hour12: false
}).format(new Date()));
return hour % 24;   // some engines format midnight as 24
```

Other LWC changes:
- `js-meta.xml`: added `lightning__FlowScreen` with `targetConfigs` for `recordId`, `recordTypeId` and `gvaAssetManagement` (all `inputOnly`). We **kept** `lightning__RecordAction` because the old quick action still uses it. We dropped `lightning__RecordPage` and `lightning__AppPage`, which weren't used on any page.
- HTML: the reminder now sits in its own panel (`role="alertdialog"`) instead of bare `slds-modal__*` markup that relied on the quick-action frame.
- CSS: replaced. `width: 1000px` and the Aura-wrapper hiding rules are gone, and there is now a responsive `max-width`.
- Behaviour change on the **old** button: if the LDS read fails (for example, no field access), the old code silently did nothing. It now assumes GVAAM = true, so the cut-off notice is shown rather than skipped.

---

## 7. How to test in the sandbox

> Hard-refresh the record page first (Ctrl+Shift+R). Action bars are cached.

**Test A: new button on desktop**
1. Open any Financial Account.
2. Click the new **New Investment Task** (the Flow action).
3. The standard *New Task: Investment Task* page opens with *Related To* = the Financial Account.

_Screenshot in the private copy: 01-action-bar-desktop.png (New Investment Task on the action bar)._
_Screenshot in the private copy: 02-new-task-page.png (New Task: Investment Task page)._

**Test B: new button on a phone**
1. Open the same Financial Account in the **Salesforce mobile app on a real device**. Browser emulation doesn't exercise native navigation.
2. Tap **New Investment Task**. The mobile Task create screen for *Investment Task* opens with *Related To* filled in.

_Screenshot in the private copy: 03-mobile-new-task.png (Mobile create screen)._

**Test C: the reminder**
- GVAAM account after 2pm Eastern: *Reminder* shows, **Acknowledged** opens the form.
- Non-GVAAM account: goes straight to the form.
- Outside the 2pm–midnight window you won't see the reminder. To test it, temporarily lower `TRADING_CUTOFF_HOUR` in the sandbox.

_Screenshot in the private copy: 04-reminder.png (Trading cut-off reminder)._

**Test D: regression check on the old button.** On desktop, click the old **New Investment Task** (LWC action). It must still open the same form.

> Both buttons have the **same label**. In Setup they are told apart by developer name: `Investment_Task` (old, LWC) vs `New_Investment_Task_Flow` (new, Flow).

On 3 Sep our developer confirmed the button worked in the sandbox. Whether Test B was run on a physical device: TBD — not found in records.

---

## 8. Test classes and coverage

None. The LWC and the flow call no Apex, so the change set needs no test class. Production showed *Run Apex Tests: Not Required*. `investmentTaskControllerTest` belongs to a different component (`notifyInvestments`). Don't add it.

---

## 9. Deployment

**Change set name:** `Investment Task Mobile`

| Component Type | Name |
|---|---|
| Flow Definition | `New_Investment_Task` |
| Action | `New_Investment_Task_Flow` (Type column shows *Financial Account*) |
| Lightning Web Component Bundle | `investmentReminder` |

- **Do not** add the Lightning Page `Investment Record Page`. A change set replaces the whole page and would overwrite Production's version. Add the action by hand (§10).
- The flow and the LWC depend on custom label `InvestmentRecordTypeId` and field `Investments__c.GVA_Asset_Management_Accounts__c`. These already existed with the original Investment Task button, so they were not added.

> ⚠️ **This change set failed in Production** with *"Programmer error: Cannot compare savepoints from two different transactions"*. Splitting it didn't help. It was replaced by a Workbench zip deploy. See **[06 – Change set savepoint error](../06-change-set-savepoint-error/README.md)** for the full story and the steps.

_Screenshot in the private copy: 05-change-set-components.png (Change set components)._

---

## 10. Post-deployment

1. **Check the flow is Active.** Setup → Flows → *New Investment Task*. If it is Draft, open it and click **Activate**. A Flow quick action is **not shown at all** while its flow has no active version. There's no error, the button is just missing. This happened in the sandbox.
2. **Add the action to the page.** Setup → Object Manager → Financial Account → Lightning Record Pages → edit the page → Highlights Panel → add **New Investment Task** (the Flow one) → Save. Then hard-refresh.
3. Users need permission to run flows. Without it the button errors on click. Check with one advisor profile.

Message sent to the client (8 Sep):
> After deploying, activate the **New Investment Task** flow in Setup → Flows — it can arrive inactive, and the button won't appear at all until it's active. Then open the Financial Account record page in Lightning App Builder and add the **New Investment Task** action to the Highlights Panel, and hard-refresh the page.

---

## 11. Lessons and gotchas

- **"Available on mobile" problems are often after the tap, not about visibility.** Check the page assignment (form factor Small), `enableActionsInNative` and `supportedFormFactors` first. Then read what the code navigates to.
- **Never navigate to a hard-coded `/lightning/...` URL.** Use `standard__objectPage` / `standard__recordPage` page references.
- **URL custom buttons (WebLinks) can't be fixed for mobile.** They have to be replaced with a quick action. *New General Task* is still a WebLink. Its intended replacement LWC (`newGeneralTaskForm`) lacks the fields that 5 Task validation rules need (Subject-driven: `Withdrawal_Type__c`, `Transaction_Value__c`, `New_Model__c`/`Other_Model_Name__c`). Fix that before swapping.
- **An LWC inside a flow:** `recordId` is not injected, so pass it in. `CloseActionScreenEvent` does nothing there, so use `FlowNavigationFinishEvent`. Remove any `@wire` that duplicates an input, or navigation can fire twice.
- **A Draft flow makes its quick action invisible.** Activate it.
- **`--dry-run` doesn't run flow activation checks.** Activation failed with *"isRequired can't be set to false for screen input fields of type boolean"*. The fix was to remove `<isRequired>` from the component field.
- **Two actions with the same label** are hard to tell apart in App Builder. Use the developer name.
- **Flow time logic:** flow formulas have no time-zone function, so keep the Eastern-time check in JS (`Intl.DateTimeFormat`).

---

## 12. Seeing a similar issue?

- A button works on desktop but does nothing on the phone → look for `standard__webPage` / `/lightning/` URLs or a URL custom button.
- You need an LWC quick action to run on mobile and be admin-editable → wrap it in a screen flow like `New_Investment_Task` and launch it with a Flow quick action.
- A Flow quick action is missing from the page → the flow is not active.
- The change set fails with a "savepoints" error → see [06](../06-change-set-savepoint-error/README.md).

## Files

- `investmentReminder.js`
- `investmentReminder.html`
- `investmentReminder.css`
- `investmentReminder.js-meta.xml`
- `New_Investment_Task.flow-meta.xml`
- `Investments__c.New_Investment_Task_Flow.quickAction-meta.xml`
- `Investment_Record_Page.flexipage-meta.xml` (sandbox only)
- Deploy package: `deploy/package.xml`, `deploy/InvestmentTaskMobile.zip`
