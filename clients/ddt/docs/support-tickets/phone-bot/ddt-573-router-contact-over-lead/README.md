# Router record priority: Contact above Lead

| | |
|---|---|
| **Ticket** | DDT-573, DDT-580 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-30 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-07-29 |
| **Resolved by** | Support team |
| **Systems** | Call router (separate service hosted on Railway, with its own Postgres database), RingCentral, Salesforce (Apex) |
| **Area** | Inbound call routing: which routing row wins when a phone number matches more than one Salesforce record, and where a call goes when every option is used up |
| **Related** | DDT-498 (clients ringing intake), DDT-571 (stored backup queue, convert-time deactivation), DDT-572 and DDT-595 (past clients, disabled queues) |
| **Code** | Router: PR #13 in the call router's own repository (DDT-580). Salesforce side of stale-row deactivation: merge commit of 2026-07-30 in the ddt-salesforce repo with no PR number recorded (deploy doc `docs/deploys/lead-convert-deactivate-routing.md`) |

---

## 1. Summary

A phone number can match a client's Account, a Contact and an old Lead at once, and the router keeps one row per source. It picked the winner by a stored "source priority", and that priority ranked **Lead above Contact**. A client whose number sat on their Contact and on an old unconverted Lead, with no number on the Account, went to the intake rep who once owned the Lead instead of their case manager.

DDT-580 swapped the order to Account, then Contact, then Lead, and corrected the priority already stored on every existing row. DDT-573 settled the rest of the routing-row model: a single last-resort queue, deactivating superseded rows, and RingCentral overflow for queues the router can't see are dead.

## 2. What was reported

- **DDT-573 (2026-07-29).** Firm operations set out the agreed model for a routing row (caller number, primary extension, department fallback, final fallback, source priority) and listed four remaining gaps: the final fallback, the Contact/Lead order, stale rows, and dead queues.
- **DDT-580 (2026-07-29).** The Contact/Lead swap split out as its own small router change. The live tiebreak returned Account 1, Lead 2, Contact 3, so an unconverted Lead outranked a Contact. This is the same "client rings intake instead of their case manager" failure as DDT-498, through a narrower door.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `sourcePriority()` in the router's `src/server.ts` | Router code | Account 1, Contact 2, Lead 3, other 99 |
| Precedence comment in the router's `db/dal.ts` | Router code | Updated to match |
| `db/backfills/ddt-580_source_priority.sql` | One-off SQL | Corrects the priority stored on existing rows |
| Final fallback | Router config | The "Intake + CMT" queue as the last resort (delivered through DDT-571's cascade) |
| Stale-row deactivation | Salesforce Apex + router | Lead row deactivated on conversion (built under DDT-571) |
| RingCentral queue overflow | RingCentral admin | Overflow on the department and intake queues set to the "Intake + CMT" queue |

**Out of scope**

- **Primary extension and department fallback logic.** Unchanged here; that is DDT-571.
- **`isQueue()` and `teamBackupReady()`** in the router's `workers/teamBackup.ts`. They still gate the queue and ungated-match branches and stay.
- **A schema change.** Nothing here needed one.

## 4. How it works (background)

```
Router routing table: one row per (tenant, phone number, source object)
  source object = Account | Contact | Lead | Unknown
  each row stores source_priority, computed when the row is upserted

On an inbound call:
  SELECT the row WHERE phone matches AND active = true
  ORDER BY source_priority ASC, updated_at DESC
  LIMIT 1
```

Because the priority is **stored on the row** at upsert time, changing the function that computes it only affects rows written afterwards.

## 5. Root cause

### DDT-580: the order

`sourcePriority()` returned Lead 2 and Contact 3. The order dated from DDT-498, when Contact rows could not be trusted: `Contact_Owner_Extension__c` used to default to the intake queue for blank and unlisted record types, so Account had to outrank it. That formula was fixed on 2026-07-28 and Contact rows now resolve to the real case manager, so the reason for ranking Lead above Contact was gone.

The phone often sits on the Contact rather than the Account, so the Account row frequently didn't exist to win the tiebreak, and the Lead row did.

### DDT-573: the remaining gaps

- **Final fallback.** If every row would carry the same last-resort queue, that is configuration, not data. Storing one value 81,027 times is how a single legacy value ended up on every row (see DDT-571).
- **Stale rows.** A converted lead's row was never removed, and nothing deactivated superseded rows.
- **Dead queues.** When the router forwards to a queue and RingCentral returns success, that means "redirect accepted", not "a person will answer". The router then releases the call and never learns the outcome, so it **cannot** detect a dead or unstaffed queue at call time.

## 6. Solution

### 6.1 Swap Contact and Lead (DDT-580)

```ts
case 'Account': return 1;   // client
case 'Contact': return 2;
case 'Lead':    return 3;
default:        return 99;  // Unknown
```

Why this order:

- **It defeats the ghost-lead problem by construction**, instead of depending on a cleanup running correctly forever.
- **A Contact is an established relationship; a Lead goes stale the moment it converts.**
- **The worse outcome is gentler.** Under the new order, someone calling about a new matter reaches their case manager, who can pass them to intake. Under the old order, an existing client reached whichever intake rep once owned a stale lead.

**Existing rows.** The ticket asked for the three routing batches to be re-run. Instead, a one-time, idempotent SQL backfill corrected `source_priority` on existing rows, since that was the only column changing. It lives in `db/backfills/`, deliberately **outside** `db/migrations/`, so the migration runner never picks it up.

### 6.2 Final fallback (DDT-573 item 1)

The ticket asked for a code constant rather than a column. What shipped, under DDT-571, was a three-level cascade: the row's department fallback, then a per-row final fallback defaulting to the "Intake + CMT" queue, then a code default. Firm operations reviewed it and approved it as better than the spec. A whitelist of valid queues rejects any unrecognised fallback and drops to the last-resort queue. The "Intake + CMT" queue covers both intake and case management, which suits a call that could come from a client or a prospect, and it times out to voicemail.

### 6.3 Deactivating superseded rows (DDT-573 item 3)

- **Lead conversion:** built under DDT-571. On conversion Salesforce sends `active: false` for the Lead row only; the router treats it as sticky. 7,715 existing ghost Lead rows were deactivated once.
- **Case-manager reassignment:** no separate step needed. The router upserts on the composite key of tenant, phone number and source object, so a reassignment overwrites the existing Account row in place rather than leaving a stale duplicate. Confirmed in production: no phone number had more than one Account row.
- **Deactivate, never delete**, so the row stays for audit and a mistake can be reversed.

### 6.4 Dead queues belong in RingCentral (DDT-573 item 4)

Two different failures, two mechanisms:

- **Bad data** (the stored fallback isn't a real queue): the router's whitelist rejects it and cascades.
- **Dead or unstaffed queue:** handled in RingCentral, which can see it. On the department case-management queues and the all-intake queue, all three overflow points (queue full, maximum wait reached, all agents busy or unavailable) now go to the "Intake + CMT" queue instead of voicemail. The PI Department queue was added later under DDT-595.

## 7. Testing

**DDT-580.** Router change deployed with health green across the deploy window. After the backfill, every row carried the new value for its source, with no mixed values:

| Source | Priority | Rows |
|---|---|---|
| Account | 1 | 17,697 |
| Contact | 2 | 34,399 |
| Lead | 3 | 28,722 |
| Unknown | 99 | 358 |

**DDT-573.** Each item was confirmed against production as above: conversion flips the Lead row inactive and the Account row wins; no phone number has more than one Account row; overflow set on each queue in RingCentral.

**Not covered:** the ticket asked for a pre-deploy count of phone numbers holding both a Lead and a Contact row (the only numbers whose winner changes). The sources do not record that count. No automated router test result is recorded beyond the health check.

## 8. Step by step

1. **Count the numbers whose winner will change:** phone numbers with both an active Lead row and an active Contact row.
2. **Change `sourcePriority()`** to Account 1, Contact 2, Lead 3, other 99, and update the comment in `db/dal.ts`.
3. **Open the router PR off `main`.** The router merge and deploy are gated by firm operations, because production secrets stay out of the PR.
4. **Deploy outside the 2 AM Central routing resync window**, and watch health.
5. **Correct existing rows** with an idempotent SQL backfill kept outside the migrations folder, or re-run `BatchFlowyteContactRouting`, `BatchFlowyteLeadRouting` and `BatchFlowyteAccountRouting`.
6. **Verify** the priority distribution per source has no mixed values.
7. **In RingCentral**, set the overflow on each department and intake queue to the last-resort queue for queue full, maximum wait and all agents unavailable.

## 9. Rollback

Revert `sourcePriority()`, then correct the stored values on existing rows again (a backfill or a full re-push of the three routing batches), since reverting the code alone only affects new upserts. Reverting brings back the Lead-over-Contact failure, so only do it if the new order causes a problem of its own. RingCentral overflow can be pointed back at voicemail in the admin console.

## 10. Known limits

- **Priority is stored, not computed at read time.** Any future change to the order needs the same backfill or a full re-push.
- **The router never learns whether a forwarded call was answered.** Dead-queue protection depends entirely on RingCentral's overflow settings staying in place.
- **The router's environment variables are not set in production**, so the source code literals are the live configuration. Changing a literal changes routing, and editing a Railway variable restarts the service, which clears the router's queue-membership maps; until they reload, matched primaries are forwarded without the availability check.

## 11. Reuse for similar issues

- **When a tiebreak picks the wrong record, check why the order was chosen.** Here the reason (untrustworthy Contact rows) had been fixed a day earlier.
- **If a value is computed at write time and stored, a code change alone doesn't fix existing rows.** Plan the backfill with the change.
- **One value on every row is configuration.** Keep it in one place.
- **Put detection where the signal is.** The router can't see whether a queue is staffed; RingCentral can.
- **Keep one-off data fixes out of the migrations folder**, so a migration runner never replays them.
