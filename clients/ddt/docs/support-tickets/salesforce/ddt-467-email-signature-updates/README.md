# Updating email signatures and sender details

| | |
|---|---|
| **Ticket** | DDT-467, DDT-637, DDT-760, DDT-1495, DDT-1515 |
| **Project** | Support tickets |
| **Status** | Live since 2026-11-05 (latest change) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-20 (first ticket) to 2026-11-04 (last ticket) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom metadata, user email settings, org-wide Email Footer), Google Workspace (Gmail signatures, mentioned only) |
| **Area** | Outbound email from the Salesforce shared inboxes: who sent it, and the address and phone number shown under it |
| **Related** | DDT-702 (governor limits on the same send path), DDT-1513 (settlement statement footer address, separate scope) |
| **Code** | PR #67, PR #68 and PR #225 in the `ddt-salesforce` repo; the rest was done in Setup and in user email settings |

---

## 1. Summary

Over four months, five tickets asked for the same kind of fix: an outbound email from Salesforce showed the wrong sender, address, phone number or email address. They kept coming back because the text under a Salesforce email is assembled from **four different places**, and each ticket was about whichever one happened to be wrong that week.

Two code changes made the shared inboxes always name the actual sender (PR #67, PR #68) and state each inbox's own callback number (PR #225). Everything else was configuration: individual users' signatures in My Email Settings, and the **org-wide Email Footer** in Setup, which turned out to be the source of the address everyone saw.

## 2. What was reported

- **DDT-467 (2026-07-20).** Emails sent from the office shared inbox gave no way to tell which team member sent them, and the default signature still showed the firm's original office address. Filed as a Google Workspace issue.
- **DDT-637 (2026-08-04).** One staff member's outbound email showed the original office address and the firm's main line instead of the new address and the PI line.
- **DDT-760 (2026-08-18).** Emails from the injury (PI) shared inbox showed the firm's main line under a PI team member's name. PI does not use that number, so clients were calling the wrong line.
- **DDT-1495 (2026-11-02).** A PI attorney's signature showed the old PI email address and the previous office address, in both the signature line and a "Note" paragraph telling clients where to send medical and insurance updates.
- **DDT-1515 (2026-11-04).** Every team member's signature in the shared inbox showed the previous office address after the firm moved again. The requester also asked how staff can change their own signatures.

## 3. Scope

**In scope**

| Component | Type | Change | Ticket |
|---|---|---|---|
| `EmailComposeService` | Apex class | Appends the sender's signature, or a fallback name line, to every new shared-inbox email; then the inbox's callback number | DDT-467, DDT-760 |
| `EmailReplyService` | Apex class | The same for replies and forwards | DDT-467, DDT-760 |
| `EmailComposeServiceTest`, `EmailReplyServiceTest` | Apex tests | Sender-line and callback-line tests | DDT-467, DDT-760 |
| `Inbox_Config__mdt.Callback_Phone__c` | Custom metadata field (new) | The number stated under the sender, per inbox | DDT-760 |
| `Inbox_Config` records `Intake_Team`, `Office_Inbox_Queue`, `PI_Team`, `SOS_Team` | Custom metadata records | Each given its callback number | DDT-760 |
| User Email Signature (My Email Settings) | User setting | Edited for individual users | DDT-637, DDT-1495, DDT-1515 |
| Org-wide Email Footer | Setup | Address updated (August, then November) | DDT-637, DDT-1515 |

**Out of scope**

- **Gmail signatures.** Staff have a Google Workspace signature as well as a Salesforce one. The Google side was updated for everyone in bulk by the firm, outside these tickets.
- **A bulk update of Salesforce user signatures.** Asked for on DDT-637 and declined: of 61 active users, only 11 had a signature saved, and each one was personalised (name, title, extension). Admins edit them one user at a time instead.
- **Document templates.** The address question in DDT-467 was answered for letters and documents by moving template generation to Dochly rather than editing the old S-Docs templates.
- **The settlement statement footer.** That is DDT-1513.

## 4. How it works (background)

What a client sees under a shared-inbox email is built in this order:

```
Staff member writes the email in the shared inbox (Salesforce)
  -> Apex adds the sender block:
       the user's own Email Signature            (if "include signature" is on and one is saved)
       otherwise a fallback line: "— <Name> / Driver Defense Team"   (PR #67)
  -> Apex adds the inbox's callback number from Inbox_Config__mdt    (PR #225)
  -> Salesforce sends it from the inbox's org-wide email address
  -> Salesforce appends the org-wide Email Footer                   (Setup; not stored on the EmailMessage)
```

So there are four sources: the **user's signature**, the **Apex fallback line**, the **inbox callback number**, and the **org-wide footer**. Only the first can be edited by the user.

**The detail that makes this hard to debug:** the org-wide footer is added *after* Apex builds the body. It does not appear in `EmailMessage.HtmlBody`, only in the delivered email. Searching the stored message, the user's profile or the email templates for the wrong text finds nothing.

## 5. Root cause

### Sender identity (DDT-467)

Every shared-inbox email goes out from the inbox's org-wide address, so the "From" name was the firm's, never the person's. The Apex only appended a signature when the user ticked "include signature" **and** had one saved; otherwise nothing identified the sender. The ticket assumed this was a Google Workspace delegation setting. It was not: these emails are sent by Salesforce, so the fix belonged in the Salesforce send code.

### The wrong phone number (DDT-760)

The ticket assumed the PI team member's signature held the wrong number. It did not: her signature was **blank**. The Apex fallback wrote her name and the firm line, and the **org-wide Email Footer** supplied the only phone number in the message, the firm's main line. That number was not in her profile, in any template, or in the code (recorded in the PR #225 commit message).

### The wrong address (DDT-637, DDT-1495, DDT-1515)

Two different sources, depending on the ticket:

- **DDT-1495** was a **user-level** signature: the text was in that user's My Email Settings, not a template.
- **DDT-1515** was the **org-wide Email Footer**. That is why the old address was identical for everyone: it was never in their personal signatures. The footer had been changed to the previous address during DDT-637 and still held it when the firm moved again.
- **DDT-637** was both: the user had no Salesforce signature at all, so one was created for him, and the org-wide address was updated at the same time.

**The lesson.** Every ticket named a person's signature as the cause, and in three of five the real source was something shared (the footer or the Apex fallback). Before editing a signature, find out which of the four sources the wrong text comes from.

## 6. Solution

### 6.1 Always name the sender (DDT-467, PR #67 and PR #68)

Both send paths now append a sender block every time, not only when a signature is saved:

```apex
// Identify the sender on every outbound shared-inbox email. The OWA "From"
// only shows the firm's shared identity, so without this nobody can tell who sent it.
User sender = [SELECT Name, Signature FROM User WHERE Id = :UserInfo.getUserId() LIMIT 1];
String finalBody;
if (includeSignature == true && String.isNotBlank(sender.Signature)) {
    finalBody = htmlBody + '<br><br>' + sender.Signature;
} else {
    finalBody = htmlBody + '<br><br>&mdash;<br>' + sender.Name.escapeHtml4() + '<br>Driver Defense Team';
}
```

Why it's done this way:

- **The signature is appended server-side as raw HTML**, so the rich-text editor cannot mangle table layouts in users' signatures.
- **The name is HTML-escaped**, since it goes straight into an HTML body.
- **The fallback means the sender is never blank**, which is what the ticket's audit need actually required.

PR #68 also made the reply service skip the real send under test, so the reply tests pass in an org with org-wide addresses.

### 6.2 State each inbox's own callback number (DDT-760, PR #225)

A new text field, `Callback_Phone__c`, on `Inbox_Config__mdt`, filled for all four inboxes: the PI inbox gets the PI line, the other three get the firm's main line (the same number the footer already gave them, so nothing changed for them). Both services append it after the sender block:

```apex
@TestVisible
private static String callbackLine(String inboxDevName, String bodySoFar) {
    if (String.isBlank(inboxDevName)) return '';
    Inbox_Config__mdt cfg = Inbox_Config__mdt.getInstance(inboxDevName);
    if (cfg == null || String.isBlank(cfg.Callback_Phone__c)) return '';
    String phone = cfg.Callback_Phone__c.trim();
    if (bodySoFar != null && bodySoFar.contains(phone)) return '';
    return '<br>' + phone.escapeHtml4();
}
```

Why it's done this way:

- **Custom metadata, not a constant.** A hardcoded number is what caused this. Ops can correct a line in Setup without a deploy.
- **No extra SOQL.** `getInstance` on custom metadata costs no query, which matters on a send path that has hit governor limits before (DDT-702). In the reply service the Case lookup was split out so one query serves both the callback line and the "From" address.
- **Not stated twice.** If the sender's own signature already contains the number, nothing is added.
- **Blank means nothing.** An inbox with no record, or a blank field, appends nothing.

The commit message notes that this makes it safe to remove the phone line from the org-wide footer, a Setup change. Whether that was done is not recorded in the ticket.

### 6.3 Fix the signatures and the footer (DDT-637, DDT-1495, DDT-1515)

- **DDT-637:** created a Salesforce signature for the user (name, team, PI line, new address) and updated the org-wide address. Admins can edit a user's signature on their behalf.
- **DDT-1495:** edited the user's Email Signature in My Email Settings, in both places the old email and address appeared (the signature line and the "Note" paragraph), and checked it in the email composer.
- **DDT-1515:** updated the **org-wide Email Footer** to the new address, and fixed the old address in two users' personal signatures.

## 7. Testing

**Apex tests**

| Test | What it proves |
|---|---|
| `testSendNewEmail_AppendsSenderNameWhenNoSignature` | With no signature, the body still carries the separator, the sender's name and the firm line, and the original body is preserved |
| Sender-line test in `EmailReplyServiceTest` (PR #68) | The same for replies |
| `testCallbackLine_BlankInboxAppendsNothing` | No inbox, no number |
| `testCallbackLine_UnconfiguredInboxAppendsNothing` | An inbox with no config record adds nothing |
| `testCallbackLine_StatesTheInboxsConfiguredNumber` | The configured number is added, escaped |
| `testCallbackLine_NotRestatedWhenSignatureAlreadyHasIt` | No duplicate when the signature already states it |
| `testSendReply_StampsTheSendingInboxsNumber`, `testSendReply_NoInboxOnCaseStampsNoNumber` | The reply path end to end, with and without an inbox on the Case |

PR #225 also made eight existing negative tests in these two classes assert that the thrown exception carries a message, instead of an empty `catch`.

**Configuration changes** were checked by eye in the email composer (DDT-1495) and by the requester on a live send (DDT-1515). No test covers them, and none can cover the org-wide footer, because it is not in the stored message.

**Live check (2026-11-05)**, active users only:

| Query | Result |
|---|---|
| `SELECT COUNT(Id) FROM User WHERE IsActive = true AND Signature != null` | **11** |
| ... and the signature contains the previous office address | **0** |
| ... and the signature contains the current office address | **5** |
| ... and the signature contains the original (oldest) office address | **5** |
| ... and the signature contains the old PI email address | **0** |

**Not covered:** the org-wide footer's contents (not readable this way), and whether the phone line was removed from it.

## 8. Step by step

**When someone reports a wrong address, phone or sender on a Salesforce email:**

1. **Get a delivered copy** of the email, not the record in Salesforce. The footer only appears in delivered mail.
2. **Find which source holds the wrong text:**
   - Compare the delivered email with the stored `EmailMessage.HtmlBody`. Text that is in the delivered email and not the stored body comes from the **org-wide Email Footer**.
   - Text under the sender that matches their personal signature is the **user's signature**.
   - A bare "— Name / Driver Defense Team" line means the user has **no signature** (or did not include it), and the fallback ran.
   - A phone number on its own line under the sender block is the inbox's **`Callback_Phone__c`**.
3. **Fix it in that source:**
   - **User signature:** log in as, or edit for, the user: Settings (profile icon) → My Email Settings → Email Signature. Replace every occurrence, including any "Note" paragraph.
   - **Org-wide footer:** Setup → Email Footers. Admin only. It applies to everyone, so check the change is right for every team.
   - **Callback number:** Setup → Custom Metadata Types → `Inbox_Config` → the inbox's record → Callback Phone. No deploy needed.
4. **Find everyone else affected.** Run a count of active users whose `Signature` contains the old text (section 7) and decide with the requester whether to fix them too. Stay within the ticket unless they say yes.
5. **Verify** by composing an email from the shared inbox and checking the preview, then ask the requester to confirm on a live send.
6. **Tell the requester how to change their own signature** (step 3, first bullet) and that the shared footer is admin-only.

## 9. Rollback

- **Code (PR #67, #68, #225):** redeploy the previous versions of `EmailComposeService` and `EmailReplyService`. That removes the sender line and the callback number. To change only a number, edit the custom metadata record instead.
- **Signatures and footer:** edit the text back. Neither keeps a history, so note the old text before changing it if you might need it.

## 10. Known limits

- **Five active users still had the original (oldest) office address in their personal signature** on 2026-11-05. DDT-1515 fixed the footer and two signatures; these five were not part of any ticket's scope. Raise a ticket before changing them.
- **Personal signatures are per user.** The next address or phone change needs the footer, the custom metadata and every saved signature checked again. A bulk update was ruled out because each one is personalised.
- **The footer is invisible to the record.** Anyone debugging from `EmailMessage` alone will not find it.
- **Gmail and Salesforce signatures are separate.** Updating one does not update the other.
- **The callback number is per inbox, not per person.** Someone with a direct line needs it in their own signature; the code then skips the inbox number only if the signature contains exactly the same text.

## 11. Reuse for similar issues

**How to recognise one:** a ticket says "my signature is wrong" or "the address on our emails is old".

- **Ask which of the four sources holds the text** before editing anything: user signature, Apex fallback line, inbox callback number, org-wide footer.
- **Same text for everyone means something shared**, usually the footer. Different text per person means personal signatures.
- **Compare delivered mail with the stored body** to tell footer text from everything else.
- **Put values staff may need to change in custom metadata**, not in code, so a phone or address change does not need a deploy.
- **When the firm moves or changes a number**, check all of these in one pass: org-wide footer, `Inbox_Config` callback numbers, every saved user signature (count them with a query), Gmail signatures, and document templates.
- **Answer the "how do I change it myself" question** in the ticket, so the next request can be self-served.
