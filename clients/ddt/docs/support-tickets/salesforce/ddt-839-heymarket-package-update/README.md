# Installing the updated Heymarket package

| | |
|---|---|
| **Ticket** | DDT-839 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-26 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-08-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (managed package, Flow), Heymarket SMS |
| **Area** | Text messaging: Heymarket to Salesforce sync |
| **Related** | DDT-522 (earlier silent flow failure caused by changed message values) |
| **Code** | PR #250 in the ddt-salesforce repo (records the upgrade; no code change) |

---

## 1. Summary

Heymarket published a new version of its Salesforce managed package, **1.211.0.3 ("August Release 2026")**, and the firm asked for it to be installed. Support installed it in the partial sandbox first, checked every automation that depends on Heymarket, then installed it in production in a quiet window between two sync runs.

Nothing broke. For production the release added one real feature, opt-out sync, and changed nothing our flows rely on.

## 2. What was reported

A request, not a fault: install the updated package, sandbox first, regression-test the automations that touch the Heymarket Message object, then upgrade production and confirm inbound and outbound messages still sync.

One surprise turned up while preparing: **production was already on 1.199.0.1 while every sandbox sat on 1.181.1.1**, and the production audit trail had no install entry for Heymarket. Heymarket had been push-upgrading production directly. So the real production change was 1.199 to 1.211, not 1.181 to 1.211.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| Heymarket managed package | Managed package | Upgraded to 1.211.0.3 in the partial sandbox and production |
| `docs/deploys/DDT-839/` | Repo documentation | Before/after evidence, package diffs and verification queries |

**Out of scope**

- **Turning on opt-out sync.** The package adds it, but enabling it would schedule a new Apex job, and production had no free scheduled-job slots. Left off.
- **Our own flows.** None needed changing.

## 4. How it works (background)

```
Heymarket (SMS platform)
  -> Delta Sync scheduled jobs (on the hour and half hour)
  -> write heymarket__Message__c records in Salesforce
  -> 4 record-triggered flows on heymarket__Message__c:
       Last_Outreach_Message_to_Lead_Client
       Last_Outreach_Message_to_Lead_Client_Update
       Last_Outreach_Message_Unconfirmed_Send
       Message_Create_Task_on_New_Video_Message
     -> stamp Last Outreach on the Account and create Heymarket tasks
```

Before the install, 22 active flows in production depended on the `heymarket` namespace. No Apex triggers or validation rules of ours touch Heymarket objects.

**The fragile part:** the four Message flows filter on literal text values, `Status__c` in (`sent`, `sending`) and `Type__c = text`. Both fields are plain text, not picklists. If the sync ever started writing different values, the flows would stop firing without any error, and no metadata comparison would show it.

## 5. Root cause

Not applicable: this was a planned upgrade, not a fault.

The one finding worth keeping: comparing two orgs' objects with the REST `describe` call produced a **false** "5 fields removed in 1.211". `describe` only shows the fields the running user can see. All five fields existed; the sandbox user simply had no access to them. Cross-org schema comparisons were redone with `FieldDefinition` through the Tooling API, which ignores field-level security.

## 6. Solution

1. **Identified the package.** Resolved the package version from the ticket through the Tooling API: 1.211.0.3, no install errors detected, not password protected.
2. **Measured what production would actually get** (1.199 to 1.211), using `FieldDefinition`, `ApexClass` and `EntityDefinition`:

   | Surface | Before | After |
   |---|---|---|
   | Heymarket objects | 29 | 30 (adds `heymarket__Opt_Out__c`) |
   | `heymarket__Message__c` fields | 55 | 55, no changes |
   | Heymarket Apex classes | 238 | 254 (+16, none removed) |
   | Invocable actions | 10 | 10, identical signatures |

3. **Installed in the partial sandbox** and checked it: install clean, no flows deactivated, Heymarket permission sets intact, Heymarket background jobs completed.
4. **Ran a functional test in the sandbox:** inserted a synthetic `sent` / `text` message through the REST API and confirmed the Account was stamped `Heymarket SMS` and a completed Heymarket task was created. The test records were deleted afterwards.
5. **Installed in production on 2026-08-26, 12:34 to 12:41 UTC**, timed between the 12:30 and 13:00 Delta Sync runs.

**Why the timing:** installing while a sync batch is running risks a half-processed batch. The gap between two half-hourly runs was the safest window.

## 7. Testing

**Production, before and after the install:**

| Check | Result |
|---|---|
| Active flows in the org | 445 before, 445 after, none deactivated |
| Message object fields | 55, unchanged |
| Heymarket permission set assignments | Unchanged |
| Scheduled Apex jobs | Unchanged; the install scheduled nothing |
| Heymarket background job failures | 0 all day |

**Live smoke test:**

- **Inbound:** sync never stopped; a message arrived at 12:50 UTC, after the install. Both Delta Sync jobs kept firing, and the 13:00 run completed fully on the new version with 0 errors.
- **Outbound, confirmed at 13:35 UTC:** 4 `sent` / `text` messages since the install; Accounts stamped `Heymarket SMS` that day went from 5 to 7; 7 Heymarket tasks created. All four Message flows were firing.
- **Status and type values unchanged**, so the flows' text filters still match.

**Not covered:** real message sync cannot be tested in any sandbox (no Heymarket configuration and no sync jobs there), so the sync itself could only be proven in production. Two production flows that use a batch send action could not be exercised in the sandbox; the action's signature is unchanged in 1.211.

## 8. Step by step

1. **Check the version in every org first.** Heymarket may have push-upgraded production ahead of the sandboxes.
2. **Inventory dependencies in production**, not the repo: query `MetadataComponentDependency` for the `heymarket` namespace.
3. **Diff the versions with the Tooling API** (`FieldDefinition`, `ApexClass`, `EntityDefinition`). Never diff `describe` output between orgs.
4. **Install in the partial sandbox.** Count active flows and scheduled jobs before and after.
5. **Run the functional test**: insert a `sent` / `text` message through the REST API (not anonymous Apex, see section 10) and check the Account stamp and the task.
6. **Pick an install window** between two Delta Sync runs, with no Heymarket batch running.
7. **Install in production** and repeat the before/after counts.
8. **Smoke test:** wait for the next Delta Sync, then confirm new inbound messages, new outbound `sent` messages, Account stamps and Heymarket tasks.

## 9. Rollback

No rollback plan is recorded in the ticket. A managed package upgrade is not something you simply reverse, and uninstalling the package deletes its data, including all message history, so the repo's deploy rules treat it as an irreversible action needing sign-off and a backup. The mitigation was the sandbox rehearsal and the timed window.

## 10. Known limits

- **Heymarket can push upgrades to production on its own schedule**, so production and the sandboxes can drift apart again.
- **Message status and type are free-text fields.** A future release that writes different values would silently stop the four flows. Re-run the status/type count after every upgrade.
- **Testing with anonymous Apex gives a false failure.** Querying the Account in the same transaction as the message insert reads it before the flow has run. Use a normal API insert.
- **A naive "wait for Heymarket jobs to finish" check never passes.** Two scheduled-job registration rows sit permanently `Queued`. Exclude `JobType = 'ScheduledApex'` from the check.
- **Enabling opt-out sync later needs a free scheduled Apex slot.**
- **The four flows are outbound-only.** If only inbound texts arrive for a while, stamps and task counts stay flat; that is expected.

## 11. Reuse for similar issues

**How to recognise one:** any managed package upgrade (Heymarket, Dochly, fw1 and so on).

- **Compare what production will actually get**, not what the sandbox jump looks like. They can differ.
- **Compare schemas with Tooling API queries**, which ignore field-level security.
- **Count active flows and scheduled jobs before and after.** A drop means the upgrade deactivated something.
- **Find automations that filter on literal values** written by the package. Those break silently.
- **Time the install between the package's own scheduled runs.**
- **Prove the business result**, not just the install: a stamped record and a created task, in production.
