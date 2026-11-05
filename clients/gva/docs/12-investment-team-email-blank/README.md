# Investment Team Email Blank on General Tasks

| | |
|---|---|
| **Status** | Resolved with an explanation. **No code shipped.** The automation works when Related To = the Financial Account |
| **Requested** | 23 Sep 2026 |
| **Answered** | 23 Sep 2026 |
| **Objects** | Task (Activity), `Investments__c`, `Rep_ID__c` |
| **Type** | Investigation + client explanation. Sandbox fix attempts were rolled back |
| **Related** | [11: Ready to Trade not working in Production](../11-ready-to-trade-not-working-prod/README.md), [04: Investment Task Popup Fix](../04-investment-task-popup-fix/README.md) |
| **Change set** | None |

---

## 1. Summary

The client saw that the *Look Up Investment Email From Rep ID on Investment* flow showed the email in **Debug**, yet a General Task the client created in Production (TEST FLIP) had a **blank** Investment Team Email, so it couldn't flip.

The automation is not broken:

- The email is copied from the **Rep ID of the Financial Account in the task's Related To**, and only **when the task is created** (or when Related To changes).
- On TEST FLIP, **Related To was empty**, so the automation never ran.
- The Debug run looked successful because Flow Builder ran it in **rollback mode**: it shows the value and then throws the write away.

We confirmed in the sandbox that a General Task with Related To = a Financial Account gets the email automatically and flips to Investment Task on the **original, unchanged flows**. We tried flow changes in the sandbox and then fully rolled them back, at the developer's request. Nothing was deployed.

---

## 2. Client request (as received, 23 Sep)

> Could you please ensure that the investment team email is populated on general tasks as well as investment tasks? I looked at the "Look Up Investment Email From Rep ID on Investment" flow and it says that the email is populated in Debug but when I created a general task to test the flip the investment team email is blank even though the Rep ID associated with the Financial Account has an investment team email.

The client attached three Production screenshots: the flow Debug panel, the TEST FLIP task, and the Rep ID record with its email.

---

## 3. Scope

**In scope:** explain why the field is blank, prove the working path in the sandbox, reply to the client.

**Out of scope (by developer decision):** changing the flows, layouts, the LWC, or validation rules. Also out of scope: email deliverability (the no-reply address arrived in the recipient's spam during tests; the developer said not to pursue it).

---

## 4. How the field gets its value

```
Rep_ID__c.Investment_Team_Email__c          ← typed in by an admin
     ▲
Investments__c.Rep_ID_new__c                ← Financial Account → its Rep ID
     ▲
Task.WhatId (Related To)                    ← must BE the Financial Account (prefix a0D)
     │
Process Builder "Update investment field of task from investment field of investment object" (v6)
   criteria: !ISBLANK(WhatId) && (ISNEW() || ISCHANGED(WhatId))
   └─ subflow Look_Up_Investment_Email_From_Rep_Id_on_Investment (iWhatId = Task.WhatId)
        Get_Investment_Details:  Investments__c WHERE Id = iWhatId
        Populate_Investment_Email_on_task:  Task.Investment_Team_Email__c = Rep_ID_new__r.Investment_Team_Email__c
```

- It's a plain Email field, **not a formula**. It's a one-time copy. Filling in the Rep ID's email later does **not** update existing tasks.
- It can't simply be turned into a formula. Workflow email alerts need a real Email field as the recipient, and formulas have no Email return type.
- The flip (`change_recordtype`) and the notification (`Task_Notify_Trading_Team_On_Ready_To_Trade`) both require this field to be non-blank. See [11](../11-ready-to-trade-not-working-prod/README.md).

---

## 5. Root cause

Three different situations leave the field blank. All three look the same on screen and show no error:

| # | Situation | What happens | Seen in |
|---|---|---|---|
| 1 | **Related To is empty** | PB criterion `!ISBLANK(WhatId)` is false, so the subflow never runs | Production **TEST FLIP** (read from the screenshot) |
| 2 | **Related To = Household / Account** (`001…`) | The subflow's `Investments__c WHERE Id = WhatId` finds nothing and takes the *Do nothing* branch | Sandbox RT TEST B. This is the most common case |
| 3 | Related To = Financial Account, but **its Rep ID has no email** | Copies an empty value | A sandbox test task on a test Financial Account whose Rep ID has no email. 27 of 45 Rep IDs had no email |

**Why General Tasks specifically:** the custom *New General Task* button on Household/Individual pages (LWC `newGeneralTaskForm`, via `newGeneralTaskAction`) sets `WhatId = recordId`, which is the **Household**. Investment Tasks are created from the Financial Account page, so their `WhatId` is the Financial Account.

**Sandbox data, 23 Sep:**

| Related To points at | Tasks | Email populated |
|---|---|---|
| `a0D`: Financial Account | 223 | 187 (84%). The 16% gap is case #3 |
| `001`: Account / Household | 726 | 1 (0.1%) |

**The Debug misunderstanding:** the Debug panel ended with a **Rollback** step. It proves the lookup logic and the Rep ID data are correct. It does **not** prove the field is written when a real task is created.

**Don't be misled by "Account Number" on the task.** On TEST FLIP it is `Account_Number_Editable__c`, a plain text field that links to nothing. There is also a formula `Account_Number__c` that reads through `Investment__c`, a separate lookup the automation doesn't use.

---

## 6. Options considered

| Option | Correct email? | User effort | Outcome |
|---|---|---|---|
| A. Remove the email gate from the flip + add a fallback recipient in the notify flow | Flips always, but notifies a fallback address, not the right team | None | Built in the sandbox, then **rolled back** (developer: don't change the flows) |
| B. Resolve the Financial Account from `Task.Investment__c` when `WhatId` isn't one; PB re-fires on `ISCHANGED(Investment__c)`; add `Investment__c` to the General Task layout | ✅ | Pick the account | Deployed to the sandbox by mistake (see §8), then **rolled back** |
| C. Derive the email from the Household | ❌ in sandbox data, 7 of 12 households with Financial Accounts map to **different** team emails | None | Ruled out |
| D. Activate the existing inactive validation rule `Related_To_Must_Be_Financial_Account` | ✅ | Blocks Household-level tasks | Not done |
| E. Change `newGeneralTaskForm` to default Related To to the household's Financial Account | ✅ | 0–1 click | Proposed, not done |
| **F. Set Related To to the Financial Account** (create the task from the Financial Account page, or change the Related To object dropdown on the Household form) | ✅ | Process only | **Chosen.** Works today with no changes |

> ⚠️ **Related To picker trap:** the object dropdown lists **two** objects labelled "Financial Account":
> `Investments__c` (prefix `a0D`) is the right one. `OASP__Financial_Account__c` (prefix `a0y`, managed package) fails in exactly the same silent way.
> Users can only tell them apart by the icon. We suggested renaming one, but it was not done.

---

## 7. How to test in the sandbox

All of these ran on the **original** flows (`change_recordtype` v2, notify v1, subflow v10, Process Builder v6).

**Test A: Related To = Financial Account (works)**
1. Open a Financial Account whose Rep ID **has** an Investment Team Email.
2. Create a General Task from it. Related To fills in with the Financial Account.
3. Query the task: `Investment_Team_Email__c` is already filled in.
4. Set Status = Ready to Trade → Save. The record type flips to **Investment Task**.

**Test B: Related To = Household (bug)**
1. Open the parent Household → **New General Task** → Save.
2. The email is blank. Ready to Trade → nothing happens and there's no error.

**Test C: from the Household, with Related To changed to the Financial Account**
1. A Household → **New General Task**.
2. In **Related To**, change the object dropdown from *Household & Individual* to **Financial Account** (the `Investments__c` one) and pick one of that household's accounts.
3. Save → the email is filled in → Ready to Trade → flips. Confirmed by the developer on 23 Sep.

Verify with:

```bash
sf data query -o GVA -q "SELECT Subject, Status, RecordType.Name, What_Id_Code__c, Investment_Team_Email__c FROM Task WHERE CreatedDate = TODAY ORDER BY CreatedDate DESC LIMIT 5"
```

`What_Id_Code__c` must read `a0D`. If it shows `a0y`, the wrong "Financial Account" was picked.

_Screenshot in the private copy: 01-related-to-dropdown.png (Household New General Task form with Related To dropdown)._
_Screenshot in the private copy: 02-email-populated.png (Task with email populated)._
_Screenshot in the private copy: 03-task-flipped.png (Task flipped to Investment Task)._

**Sandbox test records created:** RT TEST A, RT TEST B, RT TEST C, plus the developer's own test tasks. RT TEST B shows "Investment Task" only because it flipped while the patched flows were live. On today's flows it would not flip.

**Testing traps:** same as [11](../11-ready-to-trade-not-working-prod/README.md#7-how-to-reproduce-in-the-sandbox). Move Status away and back to re-trigger. The toast won't show. Check **spam** for the notification from the no-reply address.

---

## 8. What changed in the sandbox, and how it was undone (23 Sep)

This matters for anyone who deploys these flows later.

| Time (UTC) | What happened |
|---|---|
| 09:43 | Deployed `change_recordtype` **v3** (email gate removed) and notify flow **v2** (gate removed; new formula `fRecipient` falls back to the real trading-team address) |
| 09:46 | To avoid emailing the real trading team from the sandbox, the fallback in `fRecipient` was changed to **a personal test address** → notify flow **v3** |
| 09:59 | Developer asked for the fallback revert only. The command was interrupted, but the deploy **had already gone through**: notify **v4** (fallback back to the trading address) **plus** unapproved subflow **v11**, Process Builder **v7** and a layout change (`Investment__c` added) |
| 10:03 | Subflow → v10 and PB → v6 reactivated. `Investment__c` removed from the layout |
| 14:01 | Developer decided not to change the flows at all. `change_recordtype` → **v2** and notify → **v1** reactivated. Local source reverted to match |

**End state (sandbox, 23 Sep):** every flow on its **original** active version. Versions v3 (flip), v2–v4 (notify), v11 (subflow) and v7 (PB) still exist as **Obsolete**, because Salesforce never deletes flow versions. Notify **v3 is the version that contains the personal test address.**

Data side effects (sandbox only): `Investment_Team_Email__c` on RT TEST A and RT TEST C was overwritten with the personal test address so the developer would receive the notifications. Editing Rep ID `GVAG` itself was **rejected** by validation rule `GVAAM_Firm_ID_Validation` (*"GVAAM Firm ID cannot equal V9LD or R16C…"*), so the Rep ID is unchanged.

---

## 9. Reply sent to the client

The final version the developer chose (23 Sep):

> Setting Related To to the Financial Account rather than the Household populates the Investment Team Email and flips the task to Investment Task — confirmed in the sandbox here: *(link to the sandbox test task)*
>
> If you're still facing the issue, please test it in the sandbox first and send me the record so I can take a closer look.

Whether the client confirmed it worked in Production: **TBD — not found in records**.

---

## 10. Before any Production deploy of these flows

> ⚠️ **Fallback test address.** During testing, the notify flow's `fRecipient` fallback was pointed at **a personal test address**. It was reverted and the whole formula was later removed. Before any Production deploy of `Task_Notify_Trading_Team_On_Ready_To_Trade`, check that:
> 1. The file you deploy has **no** personal address. On 5 Nov 2026 the local `Task_Notify_Trading_Team_On_Ready_To_Trade.flow-meta.xml` had no `fRecipient` at all and sent to `$Record.Investment_Team_Email__c`. Run `grep -n "fRecipient\|@gmail" <file>` again before each deploy.
> 2. You did **not** get the file by retrieving from the sandbox. `sf project retrieve` brings back the **latest** flow version, which may be one of the Obsolete test versions, not the **active** one. This happened twice on 23 Sep.
> 3. After deploying, the newest version is **Active** in Production. Change sets deploy flows inactive unless *Deploy processes and flows as active* is enabled.

---

## 11. Lessons and gotchas

- **Flow Builder Debug runs in rollback mode by default.** A green debug run proves the logic, not that the record was saved.
- **"Related To" must *be* the Financial Account.** Being related to its Household isn't enough. Check `What_Id_Code__c` (`LEFT(WhatId,3)`).
- **Look-alike labels are a trap.** Two objects named "Financial Account", two flows named "Task: Flip to Investment on Ready to Trade", and the field `Popup_Shown__c` labelled "Notify Investment Team".
- **Interrupting a CLI deploy doesn't cancel it.** Check the org state after any interrupted `sf project deploy`.
- **Flow versions are permanent.** Test-only values (personal addresses) in old versions stay in the org as Obsolete versions. Don't reactivate one without reading it first.
- **An Account can have many Financial Accounts with different investment teams.** "Get it from the Household" is a business decision, not a technical shortcut.

---

## 12. Seeing a similar issue?

- Investment Team Email blank → query `What_Id_Code__c` and `Investment_Team_Email__c` on the task, then work through the three cases in §5 in order.
- Email blank even with Related To = `a0D` → check the Rep ID's **Investment Team Email** (`Rep_ID__c`). It's a data gap, not code.
- Need to fill in existing tasks after fixing Rep IDs → the copy runs only on create or a Related To change. You'd need a backfill (not built).
- Want this to work without depending on users → options B, D or E in §6. All need a client decision.

## Files

- `Look_Up_Investment_Email_From_Rep_Id_on_Investment.flow-meta.xml`: subflow that copies the email
- `Update_investment_field_of_task_from_investment_field_of_investment_object.flow-meta.xml`: Process Builder that calls it
- `change_recordtype.flow-meta.xml`: flip flow (email gate)
- `Task_Notify_Trading_Team_On_Ready_To_Trade.flow-meta.xml`: notify flow
- `newGeneralTaskForm`: Household *New General Task* form (`WhatId = recordId`)
- `newGeneralTaskAction`: quick action wrapper
