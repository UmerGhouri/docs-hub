# RingCentral contact sync filling the address book

| | |
|---|---|
| **Ticket** | DDT-95, DDT-205, DDT-581 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-30 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-02 (DDT-95); RingCentral developer support, 2026-06-19 (DDT-205) |
| **Resolved by** | Support team |
| **Systems** | RingCentral (personal contacts, shared contacts directory), Salesforce (Apex, custom metadata, custom settings) |
| **Area** | Caller ID: how client names reach RingCentral |
| **Related** | Not recorded in the tickets |
| **Code** | PR #557 in the `ddt-salesforce` repo |

---

## 1. Summary

A Salesforce integration copied every new Lead, Contact and client Account with a phone number into the **personal address book of every staff member** in RingCentral. Personal books are hard-capped at 10,000 contacts each, so they all filled up. Staff then couldn't save contacts, the integration failed on about 91% of its calls, and stale entries showed the wrong caller name.

Caller ID moved to RingCentral's account-wide **shared contacts directory**, the personal books were emptied, and the old sync was switched off. PR #557 then made the "off" state durable: missing configuration now means **off**, and the switches are off in the repo, so a deploy can no longer turn the sync back on.

## 2. What was reported

- **DDT-95 (2026-06-02):** a staff member got *"Sorry, you've reached the maximum number of contacts"* when saving a new contact in RingCentral. Day-to-day client outreach was blocked for them.
- **DDT-205 (2026-06-19):** RingCentral developer support escalated the firm's **"DDT Salesforce Integration"** app. In 7 days it made **38,078** API requests and **34,575** failed (about 91%), all on *Create Contacts*:
  - `400 PAB-101`: *"Contact cannot be created: max contacts count reached"*
  - `429 CMN-301`: *"Request rate exceeded"*

  RingCentral asked the firm to review the traffic, or suspend the app if it was no longer needed.
- **DDT-581 (2026-07-29):** staff were seeing **"RingCentral Caller"** instead of a client's name on calls. Across the books there were **89,539** such entries, 34% of every book. With the personal books being emptied, the ticket's concern was that the old sync could still refill them.

## 3. Scope

**In scope (PR #557)**

| Component | Type | Change |
|---|---|---|
| `RingCentralSyncSettings` | Apex class | The fallback used when no settings record exists now has create, update and delete sync all **off** |
| `RingCentralSyncSettingsTest` | Apex test | New: three tests (section 7) |
| `RingCentral_Sync_Settings.Default` | Custom metadata record | `Enable_Create_Sync__c` = `false` |
| `RingCentral_Extension__mdt` (all 37 records) | Custom metadata records | `Is_Active__c` = `false` (26 were `true`) |
| `2026-09-30-ddt-581-disable-rc-contact-triggers.apex` | Anonymous Apex script | Sets `Disabled__c = true` on the three RingCentral trigger settings |
| `2026-09-30-ddt-581-purge-rc-sync-queue.apex` | Anonymous Apex script | Deletes Pending rows from `RingCentral_Sync_Queue__c` |
| `docs/deploys/ddt-581-rc-contact-sync-fails-closed.md` | Deploy doc | Production state before the change, steps, rollback |

**In scope (outside Salesforce)**

- Each staff member's personal RingCentral address book, emptied by a separate purge (80,450 contacts deleted across the first 10 extensions by 2026-07-29).
- The account-wide shared contacts directory, loaded from Salesforce.

**Out of scope**

- **No Apex class was deleted.** DDT-581 asked to switch the gates off and fix the default first, and to propose removing classes separately once nothing else is shown to depend on them.
- **Call events and the webhook path.** `RingCentralSubscriptionRenewal`, `RingCentralCallAnalyzer`, `RingCentralWebhook` and the call-routing path were deliberately left alone. Only the personal-contact sync was in scope.
- **Changes to the RingCentral app's credentials.** Sequenced separately.

## 4. How it works (background)

Before the change:

```
A Lead, Contact or client Account with a phone number is inserted
  -> RingCentral trigger handler fires (unless its Triggers_Settings__c row is Disabled)
  -> RingCentralQueueService checks the create-sync switch
       (RingCentral_Sync_Settings.Default, Enable_Create_Sync__c)
  -> one RingCentral_Sync_Queue__c row is written
  -> RingCentralSyncScheduler.executeQueue runs every minute (via GlobalMinuteScheduler)
       returns early if no RingCentral_Integration_User__mdt record is active
  -> one "Create Contact" call per ACTIVE RingCentral_Extension__mdt record
```

**One new person meant one Create Contact call per active staff extension.** DDT-205 counted 29 extensions, so 29 calls per record.

Two RingCentral rules decide everything here:

- **Personal contacts are hard-capped at 10,000 per extension.** RingCentral does not raise it (`PAB-101`).
- **A personal contact takes precedence over the shared directory** for caller ID. A stale personal entry hides the correct name in the directory.

## 5. Root cause

**Why the books filled.** Copying every person into every staff member's book could never fit: DDT-205 recorded more than 34,000 contacts against 10,000 slots per user. When it was checked in June, **all 29** books were at the full 10,000. DDT-581 records all 26 staffed books at exactly 10,000, and the sync failing since 2026-06-03. From then on almost every create call was rejected, and the retries fired fast enough to be rate-limited too.

**Why calls showed "RingCentral Caller".** The integration creates a placeholder Lead named "RingCentral Caller" for an unknown inbound caller (3,174 were still in Salesforce on 2026-07-29). That placeholder name was pushed into every book. When staff later renamed the Lead, the book was never updated, and because personal entries beat the shared directory, the stale name kept winning.

**Why turning it off didn't stay off.** Two separate traps:

1. **The settings code failed open.** When no `Default` settings record was found, `RingCentralSyncSettings.getSettings()` built one in memory with `Enable_Create_Sync__c = true`. Deleting, renaming or omitting that one record from a deploy would have silently restarted the sync.
2. **The switch was turned off in Setup but left on in git.** On 2026-09-30, production had `Enable_Create_Sync__c = true` again, although it had been `false` on 2026-07-29. The repo still held `true`, so a deploy from git most likely put it back.

**Production on 2026-09-30, read before the change:**

| Item | 2026-07-29 | 2026-09-30 |
|---|---|---|
| `Enable_Create_Sync__c` on `Default` | false | **true** |
| Active `RingCentral_Extension__mdt` records | 29 | 26 of 37 |
| Active `RingCentral_Integration_User__mdt` records | not recorded | **0 of 2** |
| `RingCentral_Sync_Queue__c` | 90,973 rows | **23,946 Pending**, none processed in 30 days, still growing |
| The three RingCentral `Triggers_Settings__c` rows | `Disabled__c = false` | `Disabled__c = false` |

So on 2026-09-30 nothing was reaching RingCentral, but **only** because both integration users happened to be inactive. Re-activating either would have pushed 23,946 queued contacts into the books that had just been emptied.

**The lesson.** A switch flipped in Setup is not durable if the repo disagrees. And a default that turns a feature on when its configuration is missing turns it on at exactly the moment nobody is looking.

## 6. Solution

### 6.1 Unblock the user (DDT-95)

The fix was guidance, on 2026-06-03:

- **Save new contacts to another source.** In the save dialog, change *Source* from "RingCentral" to Google or Outlook. The user confirmed saving to Google worked.
- **Or delete a few old contacts** to make room at once.
- **For a bulk cleanup:** RingCentral has no bulk delete in its app, so export the personal contacts to CSV, remove the stale rows, and re-import. Test on a small batch first: imports may merge instead of replacing, and there is a limit of about 2,000 rows per import.

### 6.2 Stop the runaway sync (DDT-205)

The investigation on 2026-06-19 found that the custom metadata switch `Enable_Create_Sync__c` controls **only** the contact create sync, so turning it off stops the failing calls without affecting call logging or other RingCentral features. By 2026-07-29 (DDT-581), the `Default` record had create, update and delete sync all off, and the per-minute queue jobs were no longer scheduled.

### 6.3 Move caller ID to the shared directory (DDT-581)

- Caller ID moved to RingCentral's account-level **shared contacts directory**: one list for the whole account, capped at 50,000, with about 35,900 contacts loaded from Salesforce.
- Personal books were emptied. Every deletion was written to an audit file first, one line per contact, so any contact can be recreated.
- Checked: an attorney with an empty personal book saw correct client names on new and past calls, from the shared directory alone.

### 6.4 Make "off" durable (PR #557, merged 2026-09-30)

The settings fallback now fails closed:

```apex
} else {
    // DDT-581. Missing configuration must never mean "sync on": the personal-contact
    // sync was retired in favour of the shared directory, and this fallback used to
    // default create-sync to true, so deleting or omitting the Default record silently
    // resumed refilling every address book. Every switch now fails closed.
    settingsCache = new RingCentral_Sync_Settings__mdt(
        Enable_Create_Sync__c = false,
        Enable_Update_Sync__c = false,
        Enable_Delete_Sync__c = false,
        // ...
    );
}
```

And in the repo:

- `RingCentral_Sync_Settings.Default` ships with `Enable_Create_Sync__c = false`.
- All 37 `RingCentral_Extension__mdt` records ship with `Is_Active__c = false`, so even a re-enabled switch has nobody to sync to.

Why it was done this way:

- **Metadata, not only data.** A value changed only in Setup reverts on the next deploy from git. That is what happened to the create switch.
- **Several independent brakes.** The switch, the inactive extensions, the inactive integration users and the disabled triggers each stop the sync on their own. No single record is the only thing holding it off any more.
- **Production compared first.** The production copies of all 38 records were retrieved and compared with git before the change; they matched apart from line endings.

### 6.5 Scripts run by a person after the deploy

Both were marked HOLD in the PR, to be run by a person in production:

1. **Disable the triggers** (`2026-09-30-ddt-581-disable-rc-contact-triggers.apex`). Sets `Disabled__c = true` on the three `Triggers_Settings__c` rows, so the handler stops at the trigger. These are custom setting **data**, which the repo can't carry, so the script is the record. It refuses to write unless it finds exactly 3 rows.
2. **Purge the queue** (`2026-09-30-ddt-581-purge-rc-sync-queue.apex`). **Destructive.** Run only on approval and after the trigger script, or new rows keep arriving:

```apex
List<RingCentral_Sync_Queue__c> batch = [
    SELECT Id FROM RingCentral_Sync_Queue__c WHERE Status__c = 'Pending' LIMIT 9000
];
delete batch;
System.debug(
    'DELETED ' + batch.size() + ' REMAINING ' +
    [SELECT COUNT() FROM RingCentral_Sync_Queue__c WHERE Status__c = 'Pending']
);
```

Why it is written this way:

- **9,000 per run**, under the 10,000-row DML limit per transaction. Re-run until it prints `REMAINING 0` (three runs for 23,946 rows).
- **The Recycle Bin is not emptied.** The first version also emptied it, which doubled the DML to 18,000 rows for 9,000 records, so it hit the limit and deleted nothing. Leaving the bin alone also keeps a 15-day undo.
- **The rows hold nothing unique.** Each is a copy request for a Lead, Contact or Account that is still in Salesforce.

## 7. Testing

**Apex tests (`RingCentralSyncSettingsTest`):**

| Test | What it proves |
|---|---|
| `aMissingDefaultRecordFailsClosed` | With the `Default` record treated as missing, create, update and delete sync are all off |
| `theShippedDefaultRecordHasCreateSyncOff` | The `Default` record in the org has create sync off |
| `noExtensionIsASyncTargetAndNoIntegrationUserIsActive` | No extension and no integration user is active |

The first test reaches the fallback through a `@TestVisible` flag, `simulateMissingDefault`, because custom metadata records are visible to tests and would otherwise always be found.

**Validation:** the change was validated against production as a check-only deploy. The partial-copy sandbox could not take it, because it has no `RingCentral_Extension__mdt` type.

**Live state after the deploy (2026-10-01):** all 37 extension records inactive, both integration users inactive, all three triggers disabled.

**Read again on 2026-11-05:**

| Query | Result |
|---|---|
| `SELECT DeveloperName, Enable_Create_Sync__c, Enable_Update_Sync__c, Enable_Delete_Sync__c FROM RingCentral_Sync_Settings__mdt` | `Default`: all three `false` |
| `SELECT Name, Disabled__c FROM Triggers_Settings__c WHERE Name IN ('TriggerOnLeadRingCentral','TriggerOnContactRingCentral','TriggerOnAccountRingCentral')` | all three `Disabled__c = true` |
| `SELECT Is_Active__c FROM RingCentral_Extension__mdt WHERE Is_Active__c = true` | 0 rows |
| `SELECT COUNT() FROM RingCentral_Sync_Queue__c` | 0 |

**A false alarm worth knowing.** On 2026-10-01 inbound call events looked like they had stopped after this change, and the question was raised on the ticket. It was retracted the same morning: a local timestamp had been read as UTC, so an ordinary overnight lull looked like a stoppage. Compared against the same hours on the previous four weekdays, lead creation was normal. **These settings do not stop leads being created from calls.**

**Not covered:** no automated test of the end-to-end path (trigger to queue to RingCentral), and no recorded check of RingCentral's own request logs for the 24-hour quiet period DDT-205 asked for.

## 8. Step by step

1. **Read production first.** Query the settings record, the extension and integration-user records, the trigger settings and the queue count (the queries in section 7). Record what you find before changing anything.
2. **Compare production with git** for every record you will touch. If they disagree, find out why before deploying.
3. **Fix the fallback** in `RingCentralSyncSettings.getSettings()` so every switch defaults to `false`, and add `RingCentralSyncSettingsTest`.
4. **Set the switches off in the repo:** `Enable_Create_Sync__c = false` on `Default`, `Is_Active__c = false` on every extension record.
5. **Validate against production** with `RingCentralSyncSettingsTest`. A sandbox without the extension metadata type will not take it.
6. **Get the pull request approved and deploy.**
7. **Run the trigger script.** Check the `BEFORE` and `AFTER` lines in the log.
8. **With approval, run the purge script**, and re-run until it prints `REMAINING 0`.
9. **Re-run the queries from step 1** and confirm every brake reads as off.
10. **Tell users** who still need personal contacts to save them to Google or Outlook, not the RingCentral source.

## 9. Rollback

- **Code and metadata:** redeploy `RingCentralSyncSettings` and the 38 records from the previous commit on `main`. That re-arms the fail-open default, so only do it for a real reason.
- **Triggers:** the same script with `true` changed to `false`.
- **Queue purge:** restorable from the Recycle Bin for 15 days. After that, the rows can be regenerated by turning create sync back on, which is the thing this fix exists to prevent.
- **Personal contacts:** each deletion was audited first, so any contact can be recreated.

## 10. Known limits

- **The Apex classes are still there.** The sync is switched off by several brakes, not removed. Anyone re-activating an extension record, an integration user and the switch together brings it back.
- **The trigger settings are data.** A sandbox refresh or a new org will not carry `Disabled__c = true`; only the script records it.
- **The shared directory has its own cap** of 50,000. About 35,900 were loaded, which leaves headroom but not unlimited headroom.
- **Placeholder Leads still exist.** Unknown callers still create "RingCentral Caller" Leads. They no longer reach personal books, but the name problem returns if anything starts copying them again.

## 11. Reuse for similar issues

**How to recognise one:** a vendor reports a high failure rate on an integration, or users hit a storage cap in the vendor's app that they never filled by hand.

- **Count the fan-out.** One record times the number of targets is the real request volume. Check it against the vendor's cap before building, not after.
- **Find out which entry wins.** When two sources supply the same value (personal contacts versus a shared directory), the higher-precedence one hides the other, stale or not.
- **Make "off" durable.** Change the value in the repo, not only in Setup, or the next deploy undoes it.
- **Fail closed.** A feature flag's fallback for missing configuration should be off.
- **Use more than one brake** for anything that writes to an outside system, and check each one.
- **Purge scripts:** run under the DML limit, keep the Recycle Bin as your undo, and count again at the end.
- **Read timestamps in the right timezone** before calling an outage. Compare against the same hours on earlier days.
