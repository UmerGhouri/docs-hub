# Ready to Trade: Task Did Not Flip or Notify in Production (Investigation)

| | |
|---|---|
| **Status** | Investigation closed, no code shipped. Reproduced cause: a data/Related-To gate. A deployment gap (`change_recordtype` missing from the change set, see 04) has **not** been ruled out. Continued in [12](../12-investment-team-email-blank/README.md) |
| **Reported** | 21 Sep 2026 |
| **Objects** | Task (Activity), `Investments__c`, `Rep_ID__c` |
| **Type** | Investigation without Production access |
| **Feature under test** | Ready-to-Trade flip + trading-team notification, released 28 Aug 2026. See [04: Investment Task Popup Fix](../04-investment-task-popup-fix/README.md) |
| **Change set** | None |

---

## 1. Summary

A Production user set a General Task to **Ready to Trade**. As designed, the old popup no longer appeared. But the task did **not** flip to the Investment Task record type and the trading team got **no email**. The client asked whether some part of the release had not been switched on in Production.

We had **no Production access**, so all checks ran against the sandbox and the repo. We looked at several deployment-type causes (flow left inactive, FLS, org-wide email, items a change set can't carry). The client said the flows were active in Production and the org-wide address was set. What we could reproduce, in the sandbox too, is a **logic/data gate**:

- The flip flow (and the notify flow) only run when `Task.Investment_Team_Email__c` is **not blank**.
- That field is copied once, at task creation, from the Rep ID of the Financial Account in the task's **Related To** (`WhatId`). If Related To is empty or points at a Household/Account, the field stays blank. Then the flip and the notification silently never happen: no error, no toast, no flow error email.

The follow-up request two days later ([12](../12-investment-team-email-blank/README.md)) confirmed this on the Production test task (TEST FLIP), whose Related To was empty.

---

## 2. Client request (as received, 21 Sep)

> I had a user use the new task flip logic for "Ready to Trade" the pop up did not occur which is great but the task did not flip to investment nor notify the trading team. When we pushed this update could you remind me what needed to be updated/turned on. Perhaps I missed turning an aspect on in production.

Follow-up from the developer after the first findings (paraphrased): the flows were already active in Production, FLS was unlikely to be the cause, and the org-wide address was set. What else could it be, and was anything from this work left out of the change set sent to Production?

---

## 3. Scope

**In scope:** find out why the flip/notify didn't fire, using the repo and the sandbox only. No changes (the developer asked us not to change anything).

**Out of scope:** any change to flows, layouts or FLS. Later fix attempts were tried in [12](../12-investment-team-email-blank/README.md) and rolled back.

---

## 4. Background: what the 28 Aug release contains

Sandbox inventory by creation timestamp. All five were created on 2026-08-28:

| # | Component | Type | Sandbox state |
|---|---|---|---|
| 1 | `Trading_Team_Notified_On__c` | Custom field (Activity, DateTime) | Exists, **0 FieldPermissions**, on no layout |
| 2 | `notifyInvestments` | LWC, rewritten in place | Live. Now an empty template that only shows a toast. It no longer calls `InvestmentTaskController` |
| 3 | `Task_Flip_To_Investment_On_Ready_To_Trade` | Flow | **Obsolete**. A dead duplicate with the **same label** as #5 |
| 4 | `Task_Notify_Trading_Team_On_Ready_To_Trade` | Flow (after save) | Active |
| 5 | `change_recordtype` | Flow (before save), v2 | Active. This is the real flip logic, label *"Task: Flip to Investment on Ready to Trade"* |

How it works:

```
User sets Status = Ready to Trade on a General Task (record type General_Task2)
  └─ change_recordtype (before save)
       entry: Status = "Ready to Trade" AND RecordType = General_Task2
              AND ISBLANK(Trading_Team_Notified_On__c)
              AND NOT(ISBLANK(Investment_Team_Email__c))     ← the gate
       → RecordTypeId = General_Task ("Investment Task"), stamp Trading_Team_Notified_On__c
  └─ Task_Notify_Trading_Team_On_Ready_To_Trade (after save)
       entry: NOT(ISBLANK(Trading_Team_Notified_On__c))
              AND NOT(ISBLANK(Investment_Team_Email__c))     ← same gate
       → email to Investment_Team_Email__c, sender: org-wide no-reply address
```

> Record type names are **swapped** on Task: DeveloperName `General_Task` = label "Investment Task", and `General_Task2` = label "General Task".

---

## 5. Hypotheses checked

| # | Hypothesis | How we checked | Result |
|---|---|---|---|
| 1 | Flows were deployed **inactive** in Production. Change sets land flows inactive unless *Deploy processes and flows as active* is on. If so, Production's `change_recordtype` v1 would still be the active version | Read v1 metadata in the sandbox: Create-only, filters on two record type IDs that return **0 rows** in this org, no Status condition. It can never flip anything | Plausible, but the **client says the flows were active**. Never independently verified: **TBD — not found in records** |
| 2 | FLS missing on `Trading_Team_Notified_On__c` | 0 FieldPermissions even in the sandbox | Doesn't block the flow (flows run in system context). It only means nobody can **see** the stamp |
| 3 | Org-wide no-reply address missing | Exists in the sandbox. The client says it's set in Production | Ruled out per client |
| 4 | Items a change set can't carry (Status picklist value, record type picklist) | "Ready to Trade" on `General_Task2` dates from Feb 2026, before this release | Ruled out |
| 5 | **`Investment_Team_Email__c` blank, so the entry criteria are false** | Sandbox data and a live reproduction (below) | **Confirmed.** It's the cause and it happens in the sandbox as well |

### Evidence for #5 (sandbox, 21 Sep)

| Subject | Status | Record type after save | Investment Team Email |
|---|---|---|---|
| TEST FLIP SANDBOX | Ready to Trade | **Investment Task** | populated |
| TEST TASK FLIP | Ready to Trade | **Investment Task** | populated |
| test 93 | Ready to Trade | General Task | *null* |
| ZZ Popup Reset Test | Ready to Trade | General Task | *null* |

- 716 of 818 General-Task-record-type tasks had the email blank.
- Only 18 of 45 Rep IDs had an Investment Team Email filled in at all.
- The developer created task *test 23* on Financial Account "TEST IAR CS". Related To was correct, but that account's Rep ID has **no** Investment Team Email, so the field was blank and the task couldn't flip.

---

## 6. Root cause

**The flip depends on an email field that is a one-time snapshot and is usually blank.**

1. `Investment_Team_Email__c` is written by only one thing: the Process Builder *Update investment field of task from investment field of investment object*. It runs only when `!ISBLANK(WhatId) && (ISNEW() || ISCHANGED(WhatId))` and calls the subflow *Look Up Investment Email From Rep Id on Investment*, which does `SELECT … FROM Investments__c WHERE Id = :WhatId` and copies `Rep_ID_new__r.Investment_Team_Email__c`.
2. So the email is filled **only** if, at creation (or when Related To changes), Related To **is the Financial Account itself** and that account's Rep ID **has** an email.
3. Both flows require the email. When it's blank the entry criteria are false, the flow interview never starts, and nothing is logged anywhere.

Why nobody could see it: `Investment_Team_Email__c` isn't on the General Task layout (`Task-Force General Task Layout`), and the stamp field has no FLS and is on no layout. The layout still shows the **dead** checkbox `Popup_Shown__c` (label *"Notify Investment Team"*), which nothing has set since 27 Aug. It always reads unchecked and makes it look as if the notification never fires.

The full explanation of *why* the email is blank on General Tasks (Related To = Household from the New General Task button, or empty) is in [12](../12-investment-team-email-blank/README.md).

> ⚠️ **Second candidate cause, found later while writing [04](../04-investment-task-popup-fix/README.md):** the `Investment Task Popup Fix` change set that was actually uploaded did **not** include `change_recordtype`. If Production never received v2, its active flip flow is still v1, which can never flip anything (hypothesis 1 above). The email gate and the missing flow are not mutually exclusive, and **neither has been checked in Production**. When you get access, run the two checks in section 8 first: the active version of `change_recordtype` and the Setup Audit Trail.

---

## 7. How to reproduce in the sandbox

1. Pick a Financial Account whose Rep ID **has** an Investment Team Email. Create a **General Task** from that Financial Account so Related To = the Financial Account.
2. Pick a Financial Account whose Rep ID has **no** email, or create the task from a Household. Create a second General Task.
3. On each task set **Status = Ready to Trade** → Save.

Expected: task 1 flips to **Investment Task**. Task 2 stays **General Task**, with no email and no error.

Check the hidden field with SOQL, because it's not on the layout:

```bash
sf data query -o GVA -q "SELECT Id, Subject, Status, RecordType.Name, What_Id_Code__c, Investment_Team_Email__c FROM Task WHERE CreatedDate = TODAY ORDER BY CreatedDate DESC LIMIT 5"
```

_Screenshot in the private copy: 01-task-not-flipped.png (Task that did not flip)._
_Screenshot in the private copy: 02-task-flipped.png (Task that flipped)._

**Testing traps**
- Both flows have *"only when a record is updated to meet the condition requirements"*. A task already at Ready to Trade won't fire again. Set Status to something else, save, then set Ready to Trade again.
- The success **toast** usually won't appear. `notifyInvestments` is shown only while RecordType = `General_Task2`, so Lightning removes it the moment the flip happens.
- Sandbox email deliverability: in later testing ([12](../12-investment-team-email-blank/README.md)) mail sent from the org-wide no-reply address arrived in the recipient's **spam** folder.

---

## 8. Checking Production without access

We have no way to read a Salesforce org without logging in. These are the smallest things to ask someone who has Production access for:

1. **Setup → Flows**, searched by **API name** (two flows share the label): the active version of `change_recordtype` and `Task_Notify_Trading_Team_On_Ready_To_Trade`. Or run this in Developer Console with *Use Tooling API* ticked:
   ```sql
   SELECT Definition.DeveloperName, VersionNumber, Status, CreatedDate
   FROM Flow
   WHERE Definition.DeveloperName IN ('change_recordtype','Task_Notify_Trading_Team_On_Ready_To_Trade')
   ORDER BY Definition.DeveloperName, VersionNumber
   ```
2. **Setup → View Setup Audit Trail → download the last-six-months CSV.** It shows whether `change_recordtype` v2 was ever *activated* (we checked this method on the sandbox trail, and it showed the whole release).
3. **The `Investment_Team_Email__c` and Related To values on the task that failed** (a Tasks report works; the field has FLS on 62 profiles).

| Audit trail | Email on the failing task | Verdict |
|---|---|---|
| v2 never activated | either | flow never switched on |
| v2 activated | blank | email gate (this issue) |
| v2 activated | populated | something else: get flow error emails / a debug log |

> Don't refresh the sandbox to "compare with Production". It would wipe unreleased work in the sandbox.

---

## 9. Other findings (not fixed)

- `Trading_Team_Notified_On__c` has **0 FieldPermissions** (not even System Administrator). SOQL says "No such column", so you can't confirm whether the stamp fired.
- "Ready to Trade" is **not** enabled on the Investment Task record type (`General_Task`). After a flip, the task holds a Status value its new record type doesn't allow, so the next UI edit will force a different value.
- `Task_Flip_To_Investment_On_Ready_To_Trade` (Obsolete) has the same label as `change_recordtype`. Deleting it would remove the chance of activating the wrong one.
- The 28 Aug release has no post-deployment doc in `docs/`, unlike earlier releases.
- `Popup_Shown__c` ("Notify Investment Team") is dead but still on the General Task layout as a read-only field.

---

## 10. Lessons and gotchas

- **Never gate a business state change on a snapshot field.** Whether a task flips should depend on Status and record type, not on whether an email address happened to be copied in at creation.
- **Entry-criteria failures are completely silent.** No interview, no error, no log. If a record-triggered flow "does nothing", check its entry formula against the actual field values first.
- **Search flows by API name, not label.** This org has two flows with the same label.
- **Look at the layout before telling a user "check field X".** Here the deciding field wasn't on the layout and a dead lookalike field was.
- **Investigate in the sandbox first.** If the bug reproduces there, it isn't a deployment problem.

---

## 11. Seeing a similar issue?

- *"Ready to Trade did nothing"* → query the task's `Investment_Team_Email__c` and `What_Id_Code__c` (should be `a0D`). If the email is blank, see [12](../12-investment-team-email-blank/README.md).
- *"It worked in the sandbox but not in Production after a change set"* → check the flows' **active version** in Production (change sets deploy flows inactive by default).
- Popup/flip history and the original design: [04](../04-investment-task-popup-fix/README.md).

## Files

- `change_recordtype.flow-meta.xml`: the flip (before save)
- `Task_Notify_Trading_Team_On_Ready_To_Trade.flow-meta.xml`: the notification (after save)
- `Task_Flip_To_Investment_On_Ready_To_Trade.flow-meta.xml`: obsolete duplicate
- `Update_investment_field_of_task_from_investment_field_of_investment_object.flow-meta.xml`: Process Builder that fills the email
- `Look_Up_Investment_Email_From_Rep_Id_on_Investment.flow-meta.xml`: subflow
- `notifyInvestments`: toast component
- `Task_Record_Page1.flexipage-meta.xml`: visibility rule on the toast
