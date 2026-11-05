# Existing clients routed to Intake instead of their case manager

| | |
|---|---|
| **Ticket** | DDT-498, DDT-743, DDT-784, DDT-841, DDT-929 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-04 (last of the five closed) |
| **Priority** | Urgent (DDT-498), High (DDT-784, DDT-929), Medium (DDT-743, DDT-841) |
| **Reported by** | Firm staff and Case Managers, 2026-07-23 to 2026-09-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (formula fields, Apex routing batches), RingCentral, the firm's call router service |
| **Area** | Inbound call routing: matching a caller to a Salesforce record and ringing the right person |
| **Related** | DDT-849 (closed-lost potentials route to Intake), DDT-840 (lost-call alert wording) |
| **Code** | `ddt-salesforce`: PR #83, PR #265, PR #307, and a PR merged 2026-07-28 for the Contact formula. Router service: PR #18 and PR #19 in the `rc-call-router` repo. |

---

## 1. Summary

Over six weeks, five tickets reported the same symptom: a caller who is already a client rang the new-lead Intake queue instead of their case manager. They were not one bug. They were five different ways for the router to end up with the wrong destination for a known client's number, or with the right destination too late.

Each was fixed where it lived: a Contact formula that defaulted to Intake, converted Leads overwriting their client's routing, a forward that RingCentral rejected because it arrived after the call had moved on, a staff member's RingCentral department set wrongly, and secondary phone numbers that were never sent to the router.

## 2. What was reported

- **DDT-498, 2026-07-23:** a client's call rang the Defense Intake queue even though their case manager was available. A follow-up review of six more misrouted client numbers found two further causes.
- **DDT-743, 2026-08-14:** a call from a known Lead rang the whole Intake queue for six seconds and was missed, while the Lead's owner sat Available. The request was to ring the record's owner first, then fall back to the queue.
- **DDT-784, 2026-08-20:** calls from active hired clients were answered by Intake reps rather than reaching the Case Management Team (CMT). Reported at a time when all case managers were on calls.
- **DDT-841, 2026-08-25:** a caller matched to an existing Salesforce Contact went down the new-lead path and rang out. Many more examples were added the same evening.
- **DDT-929, 2026-09-03:** a client called from the **secondary** phone number on file and rang Intake for 54 seconds before the call was missed.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `Contact.Contact_Owner_Extension__c` | Formula field | Fallback no longer hard-codes the Intake queue | DDT-498 |
| `FlowyteRoutingService.executeFlowByteLead` | Apex | Skips converted Leads | DDT-498 |
| `BatchFlowyteLeadRouting.start()` | Apex batch | Skips converted Leads | DDT-498 |
| Router service decision worker | Router code (separate repo) | Logs latency on a failed forward; warm database connection pool to cut decision time | DDT-784 |
| `Account.Client_Owner_Extension__c` and `Account.Client_Owner_Fallback_Extension__c` | Formula fields | Closed-lost potentials route to Intake | DDT-841 (with DDT-849) |
| One case manager's RingCentral department | RingCentral admin setting | Changed from Intake to CMT | DDT-841 |
| `BatchFlowyteAccountRouting` and its test | Apex batch | Also pushes `Account.Phone_Secondary__c` | DDT-929 |

**Out of scope**

- **Where a client goes when their case manager and the CMT are all busy.** That is RingCentral queue configuration and a business decision, raised on DDT-498 and DDT-784 and tracked separately.
- **The lost-call alert calling a client a "new lead".** That is DDT-840, the alert's wording, not routing.
- **Routing callers with no live matters to Intake.** An early draft of PR #265 tried it and it was removed in review (see 6.4).
- **A secondary phone field on Contact or Lead.** Contact has none, and Lead carries only `MobilePhone`, so DDT-929's change is Account-only.

## 4. How it works (background)

```
Salesforce                                   Router service                     RingCentral
-----------                                  --------------                     -----------
Account  -> Client_Owner_Extension__c   -\
Contact  -> Contact_Owner_Extension__c   ->  routing table, one row per      Mainline
Lead     -> Lead_Owner_Extension__c     -/   (phone number, source object)    -> recording announcement (~5 s)
   pushed by FlowyteRoutingService                                             -> router extension (~1.25 s window)
   (on change) and the Batch*Routing                                              router looks up caller,
   classes (full re-push)                                                         checks presence, forwards:
                                                                                   MATCH  -> owner's extension
                                                                                   BACKUP -> owner's fallback queue
                                                                                   no match / forward fails
                                                                                     -> Intake queue (native cascade)
```

Three facts drive every one of these tickets:

1. **One phone number can appear on several records** (an Account, its Contact, an old Lead, an emergency contact's Contact). Each pushes its own row.
2. **When rows collide, one wins.** Originally the router's cache was keyed by phone, so the last push won. By DDT-743 the router used a priority order: Account, then Contact, then Lead.
3. **The router's cache does not heal itself.** A corrected formula changes nothing until the routing batches re-push.

## 5. Root cause

Five separate causes, one per ticket (DDT-498 had two).

### DDT-498, cause 1: the Contact formula's default was the Intake queue

`Contact_Owner_Extension__c` ended in a hard-coded Intake queue extension for any Contact whose record type was not one of three whitelisted types. The affected Contact had **no record type**, so it fell through to Intake. The Account behind it held the correct case manager extension, but the formula never read it. Both records pushed the same phone number, and the Contact's wrong value overwrote the Account's right one.

Blast radius measured on the ticket: **17,679 client Contacts** whose formula resolved to Intake while their Account pointed at a real case manager.

### DDT-498, cause 2: converted Leads kept pushing the intake rep

After a review of six more misrouted client numbers (2026-07-28), four turned out to have a second cause. Once a Lead converts, the caller is a client routed by Account and Contact. But the Lead-routing push still sent the **converted** Lead's phone with the **Lead owner's** extension, which is the intake rep who signed them. That overwrote the client-to-case-manager row.

The other two of the six had no Lead at all. Their case manager was on Do Not Disturb, and the router's BACKUP branch at that time sent them to the new-lead Intake queue. That is router configuration, decided separately.

### DDT-743: the router had matched correctly; presence said "unavailable"

The router matched the caller to the right owner and checked presence. Presence is built from several RingCentral signals (telephony status, active calls, meetings, Do Not Disturb), so it can report "unavailable" when the simpler Available indicator in the RingCentral app shows green. It did, and the router fell back to the queue as designed.

A controlled test call then failed for a different reason: the test number already belonged to an existing client's Contact as well as the new test Lead. **The router ranks a Contact above a Lead**, so it used the Contact's routing. Once the owner was set on that client record, the test routed to the intended person. How the ticket was finally closed is not recorded in the ticket.

### DDT-784: the right decision arrived after the call had left

The ticket assumed the router was not consulting Salesforce. It was. The router matched the caller, saw the case manager unavailable, and correctly chose the CMT queue. The **forward** failed with `409 Conflict`.

Leg timings from RingCentral showed why:

| Stage | Time spent |
|---|---|
| Recording announcement | about 5.2 s |
| Router extension | about 1.25 s |
| Rollover to the Intake queue | immediate after that |

The router extension has a line but nothing registered on it, so RingCentral rolls the call over almost at once instead of waiting out a ring timer. The window for the router to act was about **1.25 seconds, not the 10 seconds everyone had assumed**. On the failed calls the router's forward arrived 2.9 and 8.4 seconds after RingCentral had already handed the call to Intake.

Measured over five hours on 2026-08-20: **58 forwards, 52 succeeded, 6 failed (10.3%)**. Five of the six were aimed at a case manager or the CMT queue. Two theories were ruled out with the same data: the target extensions accepted forwards at other times in the window, and the router's WebSocket reconnects did not line up with any failure.

### DDT-841: two causes, neither a router bug

- **The example caller was a closed-lost potential**, not a client. Under the firm-wide rule set on DDT-849, a closed-lost potential who calls back is a new lead and **should** go to Intake. The complaint was superseded by that rule.
- **A senior case manager's RingCentral department was set to "Intake".** In Salesforce they were a Case Manager in the CMT department, and the case manager on **4,572 Accounts**.

### DDT-929: only the primary phone number was ever sent

The routing batches pushed `Account.Phone` and nothing else. The client called from `Account.Phone_Secondary__c`. That number belonged to a Lead in the routing table, so the call followed the Lead's owner, who was on Do Not Disturb, and fell to Intake.

## 6. Solution

### 6.1 DDT-498: fall back to the Account's case manager

Applied directly in production as an urgent hotfix on 2026-07-24, then recorded in git the same day and merged 2026-07-28.

```
IF(
    RecordType.Name = 'Referring Attorney',
    INTAKE_EXT,
    IF(
        NOT(ISBLANK(Owner_Extension__c)),
        Owner_Extension__c,
        IF(
            NOT(ISBLANK(Account.Client_Owner_Extension__c)),
            Account.Client_Owner_Extension__c,
            INTAKE_EXT
        )
    )
)
```

`INTAKE_EXT` stands for the Intake queue's extension number. The formula has since been extended by DDT-755 and DDT-945 to consult related records as well; the order above is the DDT-498 version.

Why this order: a Contact's own owner extension is the most specific answer; the Account's case manager is the next best; Intake is now only the last resort. Referring Attorneys keep going to Intake on purpose.

### 6.2 DDT-498: stop pushing converted Leads (PR #83)

```apex
// BatchFlowyteLeadRouting.start()
'SELECT Id, Phone, Lead_Owner_Extension__c ' +
'FROM Lead ' +
'WHERE Lead_Owner_Extension__c != null ' +
'AND Phone != null ' +
'AND IsConverted = false';
```

The same `AND IsConverted = false` was added to the query in `FlowyteRoutingService.executeFlowByteLead`. A converted Lead's caller is a client; the Account and Contact rows are the ones that should decide.

### 6.3 DDT-784: measure first, then shorten the decision

The router service (a separate repo) got two changes:

- **PR #18 (option A):** the failure path now logs the decision and forward latency and the party status, which before were only logged on success. Without this, "the router was late" was an inference from RingCentral leg timing, not a measured number.
- **PR #19:** a warm database connection pool, to cut the time the router takes to decide.

A larger option was proposed and probed: catch the call during the 5-second recording announcement instead of at the router extension, which a read-only probe showed was possible because the caller's leg is never answered during the announcement. Whether it was built is not recorded in the ticket.

Options considered and set aside: registering a non-answering device on the router extension to widen the window (it would ring on every routed call, and a dropped registration would silently undo it), and retrying on `409` (the call has already gone by the time the error comes back).

### 6.4 DDT-841: closed-lost to Intake (PR #265), and one RingCentral setting

`Account.Client_Owner_Extension__c` was changed so the three Lost client types return the Intake queue instead of the intake owner. Its companion `Client_Owner_Fallback_Extension__c` returns blank for them, because a queue has no per-user fallback. Contacts inherit the Account's value, so the change flows through to them.

An early draft also sent "empty" Accounts to Intake, using `Primary_Client_Type__c = "Potential Defense Client"` with no active potential cases. Review removed it: that label is also the catch-all for Accounts that match none of the other conditions, and **136 of 1,179 matching Accounts held hired or closed matters**. It would also have made every new Account push to the router on insert.

Separately, the case manager's department was changed from Intake to CMT in RingCentral admin and confirmed through the RingCentral API.

### 6.5 DDT-929: push the secondary number too (PR #307)

```apex
String secondaryPhone = normalizeToE164(acc.Phone_Secondary__c);
if (String.isNotBlank(secondaryPhone) && secondaryPhone != phone) {
    Map<String, String> secRule = new Map<String, String>();
    secRule.put('inbound_number', secondaryPhone);
    secRule.put('destination',    String.valueOf(extVal));
    secRule.put('source_object',        'Account');
    secRule.put('salesforce_record_id', acc.Id);
    secRule.put('fallback_extension', fbVal == null ? '' : String.valueOf(fbVal));
    rules.add(secRule);
}
```

Why it's written this way:

- **Same normalization as the primary.** No separate, stricter helper. The router's own E.164 check drops junk values (emails, addresses, notes).
- **Same `source_object` of `Account`.** A separate lower-priority tier for secondaries was considered and rejected: when a secondary number is shared with an emergency contact's Contact, the client's Account should still win.
- **Skipped when blank or equal to the primary**, so there are no duplicate rows.

Scale measured for the PR: 2,479 Accounts carried a secondary number; about 98% normalized to a valid number, about 2% were junk; at least 184 valid secondaries also matched some other record's primary.

## 7. Testing

| Change | Test | What it proves |
|---|---|---|
| PR #83 | `FlowyteRoutingServiceTest`, `BatchFlowyteLeadRoutingTest`, 15 of 15 passing at deploy | The Lead push still works for unconverted Leads |
| PR #265 | `FlowyteRoutingServiceTest`, `BatchFlowyteAccountRoutingTest`, 23 of 23 at validation | No regression in the Account push. No formula-outcome test was added; it was deferred with the "empty account" follow-up. |
| PR #307 | `BatchFlowyteAccountRoutingTest` plus `FlowyteRoutingServiceTest`, 26 of 26 at validation | Three new cases: secondary pushed as a second row, a no-digit secondary skipped, a secondary equal to the primary deduped |

**Live checks recorded on the tickets:**

- DDT-498: after the formula fix and a batch re-run, the affected Contact returned the case manager's extension and the router's table mapped the number to the case manager.
- DDT-841: the RingCentral department change was confirmed through the RingCentral API.
- DDT-929: the router's data was checked and the client's secondary number mapped to their case manager.

**Not covered:**

- No automated test can place a real call. Live test calls were left to the requesters on DDT-841 and DDT-929, and their outcome is not recorded in the tickets.
- The formula changes (6.1, 6.4) have no direct Apex assertion of their output.
- Whether the DDT-784 changes lowered the 10.3% forward failure rate is not recorded in the ticket.

## 8. Step by step

When a known client rings Intake:

1. **Find the router's decision for that call.** Note whether it matched, which record and extension it matched, the presence result, and whether the forward succeeded.
2. **If it matched the wrong extension**, list every Salesforce record carrying that phone number (Account, Contact, Lead, emergency contacts) and what each pushes. Check which one wins under the router's priority.
3. **Fix the source, not the router.** Correct the formula, exclude records that should not push (as with converted Leads), or add the missing number (as with secondary phones).
4. **Check the person's RingCentral department and presence**, if the match was right but the call still fell to a queue.
5. **If the decision was right and the forward failed**, look at the RingCentral leg timings for that session: how long the call sat on the router extension before rolling over, against when the router sent its forward.
6. **Validate any Salesforce change against production** with `FlowyteRoutingServiceTest` and the relevant `Batch*RoutingTest`, then get approval and deploy.
7. **Re-run the routing batches** (`BatchFlowyteLeadRouting`, `BatchFlowyteContactRouting`, `BatchFlowyteAccountRouting`) outside the 2 AM Central window. The router does not pick up a formula change on its own.
8. **Confirm in the router's data** that the number now maps to the right extension, then ask the requester to place a live call.

## 9. Rollback

- **Formula fields:** redeploy the previous version. That alone does not undo anything in the router. Re-run the routing batches afterwards to overwrite the rows that were pushed.
- **PR #83:** revert the two queries and re-run the Lead batch. Converted Leads would start overwriting client rows again.
- **PR #307:** revert the class. The roughly 2,400 secondary rows already pushed stay in the router until they are cleared or the batch is re-run after the revert.
- **RingCentral department:** change it back in RingCentral admin.

## 10. Known limits

- **Shared numbers still resolve to one record.** Account beats Contact beats Lead. A number that is a Contact's own primary and also an Account's secondary goes to the Account; that was accepted.
- **The router's window is short.** Any added latency in the router's decision, or a change to how long RingCentral holds the call on the router extension, brings the `409` failures back.
- **"No live matters" callers are not sent to Intake.** That needs a reliable empty-account signal and was left to a separate ticket.
- **Client overflow** when the case manager and the CMT queue are all busy is RingCentral queue configuration, not covered here.
- **Lead and Contact have no secondary phone field**, so only Account secondaries are routed.

## 11. Reuse for similar issues

**How to recognise one:** a caller who should match a specific person rings a general queue, or a fix in Salesforce "didn't work".

- **Read the router's decision log before assuming the lookup failed.** On DDT-784 the lookup was right and the forward was late. On DDT-743 the match was right and presence said unavailable.
- **Find every record that pushes the same phone number.** Most misroutes here were two records disagreeing, with the wrong one winning.
- **Never let a formula's catch-all branch be a real destination** unless that is a deliberate business rule. A blank record type is a common way to land in it.
- **Records that stop representing the caller should stop pushing** (converted Leads, closed-lost potentials under the firm's rule).
- **Re-push after every Salesforce-side fix.** The router's table is a copy.
- **Measure timing from both sides.** RingCentral's leg timings showed a 1.25-second window that everyone had believed was 10 seconds.
- **Check the person's setup in RingCentral too.** A department label in RingCentral admin can send calls the wrong way when Salesforce is correct.
