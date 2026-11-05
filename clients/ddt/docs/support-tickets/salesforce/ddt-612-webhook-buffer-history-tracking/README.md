# Turning off field history on the webhook replay buffer

| | |
|---|---|
| **Ticket** | DDT-612 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-11 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-08-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom object metadata), RingCentral webhook processing |
| **Area** | Data storage: the RingCentral webhook retry buffer |
| **Related** | None recorded |
| **Code** | PR #108 in the ddt-salesforce repo |

---

## 1. Summary

Field history tracking was switched on for `Webhook_Replay_Buffer__c`, a working buffer that holds RingCentral webhook deliveries until they are processed. Every retry wrote history rows, so the history table grew to about twice the size of the buffer itself, using about 950 MB. Nobody used the history and no reason for it was found, so tracking was turned off, and the history rows went with it.

## 2. What was reported

- **2026-08-03:** `Webhook_Replay_Buffer__History` held **492,614** rows against **243,489** rows in `Webhook_Replay_Buffer__c`. The ticket asked whether tracking was intentional, whether anything used it, and, if not, to turn it off and reclaim the storage.
- When the change was made the same day, the counts were **486,328** history rows (about 950 MB) against **240,837** buffer rows.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| `Webhook_Replay_Buffer__c` (object) | `enableHistory` true to false |
| `Webhook_Replay_Buffer__c.Attempts__c` | `trackHistory` true to false |
| `Webhook_Replay_Buffer__c.Last_Error__c` | `trackHistory` true to false |
| `Webhook_Replay_Buffer__c.Processed_At__c` | `trackHistory` true to false |

**Out of scope**

- **The buffer rows themselves.** Not touched; the webhook processing that depends on them is unchanged.
- **A retention policy for the buffer.** Not part of this ticket.

## 4. How it works (background)

The object's own description says it is the idempotency and retry buffer for every RingCentral webhook delivery:

```
RingCentral webhook arrives
  -> receiver upserts one Webhook_Replay_Buffer__c row per event (keyed on the webhook id)
  -> processing sets Processed_At__c
  -> on failure: Attempts__c goes up, Last_Error__c is written
  -> a reconciliation job retries any row where Processed_At__c is still blank
```

The three tracked fields are exactly the ones that change on every retry, so each retry wrote about two history rows.

## 5. Root cause

History tracking was enabled on a high-churn operational object with no audit requirement. The fields it tracked change every time a delivery is retried, so the history table grew faster than the object.

**Why it was on:** no documented reason was found. In the repo, tracking first appears in a general commit on 2026-05-11 ("logging system updates") with no explanation.

## 6. Solution

Tracking was disabled in production on 2026-08-03, and PR #108 recorded the same change in git on 2026-08-05 so `main` matches production.

In `Webhook_Replay_Buffer__c.object-meta.xml`:

```xml
<enableHistory>false</enableHistory>
```

In each of `Attempts__c`, `Last_Error__c` and `Processed_At__c`:

```xml
<trackHistory>false</trackHistory>
```

**No separate purge was needed.** Per the resolution comment and the PR, turning history off on the object removed the history table and all its rows, reclaiming about 950 MB.

**Why the object-level switch and not just the fields:** nothing on this object needs history, so turning it off at the object removes the history table entirely and nothing can start filling it again by accident.

## 7. Testing

- The resolution comment (2026-08-03) records `enableHistory=false` deployed and verified, all 486,328 history rows purged, and the buffer untouched.
- **Checked again on 2026-11-05:** `SELECT COUNT() FROM Webhook_Replay_Buffer__History` returns `INVALID_TYPE`, meaning the history object no longer exists in production.
- No Apex tests are involved: this is a metadata setting only.
- **Not recorded:** the before-and-after storage figures from the Storage Usage page that the ticket asked for. Only the about-950 MB estimate is recorded.

## 8. Step by step

1. Measure: count rows in the object and in its `__History` table.
2. Check what is tracked: look for `trackHistory` set to true in the object's field metadata.
3. Confirm nothing reads the history: search the repo for `Webhook_Replay_Buffer__History` and ask the code owner about reports or debugging that use it.
4. **Accept that the history will be gone.** Turning off object history deletes it; export it first if anyone might need it.
5. Set `enableHistory` to false on the object and `trackHistory` to false on each tracked field. Deploy, or untick Track Field History in Setup, then Object Manager.
6. Confirm the history object is gone: the count query should now return `INVALID_TYPE`.
7. Commit the same metadata to git the same day, so a later deploy from `main` doesn't switch tracking back on.

## 9. Rollback

Setting `enableHistory` back to true restarts tracking from that moment. **The deleted history cannot be recovered.**

## 10. Known limits

- **The buffer itself still grows** unless something prunes processed rows. This ticket did not add a retention policy.
- **If tracking is ever re-enabled on these fields, the bloat returns** at about two history rows per retry.

## 11. Reuse for similar issues

**How to recognise one:** a `__History` table larger than the object it tracks, on an object used for queues, buffers, logs or sync state.

- **Compare row counts** of every custom object against its history table.
- **Tracked fields that change on every retry or status update** are the usual cause.
- **Disabling at the object level removes the whole table.** Confirm nobody needs the history first, since it cannot be brought back.
- **Record the change in git the same day**, or a later deploy from `main` will quietly turn tracking back on.

A note for anyone searching the repo: commits tagged "DDT-612" belong to unrelated Drive panel work, not this ticket. The change for this ticket is PR #108.
