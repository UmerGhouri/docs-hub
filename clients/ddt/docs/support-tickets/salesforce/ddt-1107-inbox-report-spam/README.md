# Report spam button in the shared inbox

| | |
|---|---|
| **Ticket** | DDT-1107 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-28 |
| **Priority** | No priority |
| **Reported by** | Firm operations, 2026-09-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, trigger, LWC, custom object, custom metadata, permission set) |
| **Area** | Shared email inbox (Email-to-Case) |
| **Related** | None recorded |
| **Code** | PR #483 (feature) and PR #507 (protected-sender list) in the ddt-salesforce repo |

---

## 1. Summary

The shared inbox had no way to report spam or block a sender, and about 1 in 3 of the roughly 6,500 email Cases a month came from automated noreply, notification or newsletter senders. This added a **Report spam** button (single and bulk), a **per-inbox sender block**, a **Spam** filter, and a **Not spam** undo. Senders that must never be blocked, such as website-form notifiers, lead vendors, courts and clients, are protected by an admin-editable list. The feature went live on 2026-09-24; the protected list was widened from 12 to 35 entries on 2026-09-28.

## 2. What was reported

- **2026-09-24:** firm leadership asked for a Report spam button in the inbox. Decisions in the ticket: block **per inbox**, not firm-wide; block exact addresses only, never whole domains; never delete anything; and never block form or notification senders, because reporting one spam form submission must not stop every future real lead.
- Measured in production on 2026-09-24: **6,536** email Cases in the last 30 days, confirming the ticket's estimate.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Case.Is_Spam__c` | Custom field (checkbox) | New |
| `Blocked_Sender__c` | Custom object | New: `Email__c`, `Inbox__c`, `Active__c`, `Blocked_By__c`, `Reported_Case__c`, list view `All` |
| `Spam_Protected_Sender__mdt` | Custom metadata type | New, with field `Address__c`; 12 records, later 35 |
| `InboxSpamService` | Apex class | New: report, block, protect, auto-spam, keep closed, Not spam |
| `InboxSpamServiceTest` | Apex test | New, 21 test methods |
| `CaseInboxTrigger` | Trigger | Adds `before update` |
| `CaseInboxTriggerHandler` | Apex class | Applies blocks on insert; new `beforeUpdate`; logs Auto-spam |
| `EmailInboxController` | Apex class | `reportSpam`, `bulkReportSpam`, `markNotSpam`, Spam status filter |
| `emailInbox`, `emailThreadView` | LWC | Spam segment, bulk Spam button, Spam badge, Report spam and Not spam buttons |
| `Email_Inbox_User` | Permission set | Access to `Blocked_Sender__c` (no delete) and field access |

**Out of scope** (from the ticket): blocking at the Gmail level, domain blocking, sweeping older mail from a blocked sender, unsubscribing, and spam Leads, which are handled elsewhere.

## 4. How it works (background)

```
Email arrives at a shared inbox
  -> Email-to-Case inserts the Case
       CaseInboxTrigger before insert -> InboxSpamService.applyBlocks
         sender + inbox match an active Blocked_Sender__c row,
         and the sender is not protected and not a client?
           -> Is_Spam__c = true, Status = Closed, "Auto-spam" on the timeline
  -> Email-to-Case inserts the EmailMessage
       active flow Reopen_Case_Status_Only: Closed Case + new email -> Status = New
  -> CaseInboxTrigger before update -> InboxSpamService.keepSpamClosed
       spam Case stays Closed
```

**Report spam** sets `Is_Spam__c`, closes the Case, logs a "Marked spam" activity and creates a `Blocked_Sender__c` row for that sender and that inbox only. **Not spam** clears the flag, reopens the Case to New, and sets the block's `Active__c` to false.

## 5. Root cause

Not applicable: a new feature. Three production findings shaped it:

1. **An active flow would have undone every auto-spam.** `Reopen_Case_Status_Only` (active in production, not in git) flips a Closed Case back to New when an inbound email lands on it. Email-to-Case inserts the Case first and the email second, so a Case closed as spam on insert would be reopened a moment later. It would have passed a unit test and done nothing in production.
2. **Another active flow still emails Intake staff about every Intake email**, including auto-spam. It was not changed here (see 10).
3. **The 12 seed protected entries missed about 800 emails a month** that must never be blocked, including a paid lead feed, the office scanner, court e-filing notices and senders of portal verification codes.

## 6. Solution

### 6.1 Blocking, per inbox

`applyBlocks` runs in `before insert`, with one query for the whole batch:

```apex
for (Blocked_Sender__c b : [
    SELECT Email__c, Inbox__c FROM Blocked_Sender__c
    WHERE Active__c = true AND Email__c IN :emails
    LIMIT 2000
]) {
    activeKeys.add(key(b.Email__c, b.Inbox__c));
}
```

The key is address plus inbox, so a sender blocked in one inbox still reaches the others.

### 6.2 Keeping spam closed

Rather than editing a flow that lives only in production, a `before update` branch was added to the **existing** `CaseInboxTrigger` (the repo forbids a second trigger on an object):

```apex
public static void keepSpamClosed(List<Case> newCases, Map<Id, Case> oldMap) {
    for (Case c : newCases) {
        Case old = oldMap.get(c.Id);
        if (old == null) continue;
        if (c.Is_Spam__c == true && old.Status == 'Closed' && c.Status != 'Closed') {
            c.Status = 'Closed';
        }
    }
}
```

The only way to reopen a spam Case is to clear `Is_Spam__c` in the same update, which is what Not spam does.

### 6.3 Senders that are never blocked

Reporting one of these marks **that one email** as spam and creates **no** block. The toast says so.

- **`Spam_Protected_Sender__mdt`:** an exact address, or a domain starting with `@`. A domain entry also covers its subdomains but not look-alikes. Matching ignores case. Admins add records under Setup, Custom Metadata Types, Spam Protected Sender, Manage Records. **No deploy needed.**
- **Clients:** any address that is a Person Account's email.

Both checks run **again when mail arrives**, so adding a protected entry later rescues an address already blocked, and a blocked sender who later becomes a client gets through.

### 6.4 Widening the protected list (PR #507)

Firm leadership built the list from production: every inbound email over 60 days (13,647), grouped by sender, with subject lines read for any sender an address alone did not identify. **23 entries** were added on top of the 12 seed records, making **35**:

| Group | Examples (by type) |
|---|---|
| Leads and referrals | Paid lead feed, a case-booking service, a human referral source |
| Courts and e-filing | E-filing providers, county court systems, a court announcement list |
| Our own systems | Office scanner, outbound mail service (including its login codes), bank payment alerts, card payments |
| Records and PI vendors | Medical records vendors, an insurer (whole domain, because adjusters write from all over it) |

Six newsletter senders were **deliberately left blockable**, so the button still does its job. One court announcement list had to be an **exact address**, because its envelope sender is a mailing service, not the court's domain.

## 7. Testing

**Apex, `InboxSpamServiceTest` (21 test methods):** single and bulk report at 251 records, a teammate's claimed email refused, duplicates, per-inbox scoping, protected exact and domain entries, a protected entry added after a block, a client sender at report time and on arrival, auto-spam on insert at 251 records, the reopen guard, Not spam on reported and auto-spam emails, the list filters, and null paths.

`aSpamEmailStaysClosedWhenAnInboundReplyOrAutomationTriesToReopenIt` inserts a real inbound email, so in production it runs against the real flow. It also does a direct `Status = 'New'` update, so it proves the guard even where the flow is missing.

**Against production, check-only, 2026-09-24:**

| Run | Result |
|---|---|
| `InboxSpamServiceTest` only | 21 of 21 passing |
| `RunLocalTests` full sweep | 2,949 of 2,949 passing |
| The three covering test classes | 92 of 92 passing (the deployed validation) |

Naming only `InboxSpamServiceTest` fails per-class coverage (`CaseInboxTriggerHandler` 65.6%, `EmailInboxController` 29.4%), so `EmailInboxControllerTest` and `CaseInboxTriggerHandlerTest` must be named too.

**Jest:** 457 of 457 locally, including 4 new `emailInbox` tests and 3 new `emailThreadView` tests.

**Sandbox:** all five acceptance criteria passed end to end.

**PR #507:** every one of the 23 entries matched real production senders (a typo would look protected and not be), and the six deliberately blockable senders were confirmed still blockable against all 35.

**In production, 2026-09-24 to 2026-09-28:** 40 senders blocked and 64 Cases marked spam, all in the Defense inbox. None of the blocked senders was on the protected list.

**Not covered:** a validation never renders an LWC. Whether the buttons were clicked in production by a real inbox user, rather than only in the sandbox, was raised as an open question and is not recorded.

## 8. Step by step

1. **Check what else changes Case status.** Query active flows on Case and EmailMessage. Here, `Reopen_Case_Status_Only` would have reopened every auto-spam.
2. **Retrieve the existing inbox classes, trigger and LWCs from production** and confirm they match the branch before editing them.
3. **Add the new field, object and metadata type**, and seed the protected list.
4. **Add `InboxSpamService`** and call it from the existing trigger handler: `applyBlocks` in before insert, `keepSpamClosed` in before update.
5. **Add the controller methods and LWC buttons,** and exclude spam from the Resolved and See All views.
6. **Validate against production** naming all three covering test classes, then quick-deploy as the user who validated.
7. **Measure real senders before go-live:** inbound email over 30 to 60 days, grouped by sender. `FromAddress` cannot be grouped in SOQL, so pull rows and count outside.
8. **Add the senders that must never be blocked** to `Spam_Protected_Sender__mdt`, checking each one against real traffic.
9. **Live check in the inbox:** report a test sender, send again to the same inbox (lands in Spam) and to another inbox (arrives normally), then Not spam.

## 9. Rollback

- **Stop all blocking at once:** set `Active__c = false` on every `Blocked_Sender__c` row. New mail stops being filtered immediately, with no code change.
- **Restore spam mail:** press Not spam, or clear `Is_Spam__c` and set Status to New in the same update.
- **Code:** redeploy the previous `CaseInboxTrigger` **first** (without `before update`), then the handler, controller and LWCs. The trigger calling a method that no longer exists will not compile.
- Leave the new field, object and metadata type in place; deleting them destroys the block history.

## 10. Known limits

- **The protected list is a 60-day snapshot.** A sender that writes rarely (a court, an annual vendor) may be missing. When someone says "we stopped getting X", check first whether X was reported as spam. Re-run the sender count every 90 days.
- **Intake staff still get personal emails about auto-spam** in the Intake inbox, from a separate production-only flow. Adding a not-spam condition to it was left as a separate decision.
- **Only exact addresses are blocked.** A spammer who changes address gets through.
- **Older mail from a newly blocked sender is not swept.**

## 11. Reuse for similar issues

- **Before closing records in a trigger, look for flows that reopen them.** A unit test will not show it, because the flow only runs in the org that has it.
- **Guard in the existing trigger's handler** instead of editing a flow that only exists in production.
- **Any "block" feature needs a never-block list** built from real traffic, checked in both directions: everything that must survive is protected, and the noise is still blockable.
- **Make blocks reversible** (an active flag, not a delete) and re-check protections on arrival, so fixing the list repairs past mistakes.
