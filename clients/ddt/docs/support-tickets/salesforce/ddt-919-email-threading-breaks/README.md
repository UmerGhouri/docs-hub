# Email threading breaks on long or mixed chains

| | |
|---|---|
| **Ticket** | DDT-919, DDT-970 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-07 (DDT-919) and 2026-09-14 (DDT-970) |
| **Priority** | High |
| **Reported by** | Case Manager, 2026-09-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC), Email-to-Case |
| **Area** | The shared email inbox: reading and replying to a Case's email thread |
| **Related** | None recorded |
| **Code** | PR #310 and PR #314 (DDT-919), PR #354 (DDT-970) in the `ddt-salesforce` repo |

---

## 1. Summary

Two linked problems in the shared email inbox, reported by the same PI case manager a week apart.

- **DDT-919:** on long chains, emails sent from the Matter were missing from the inbox thread, Reply quoted an old received email instead of the newest message, and every reply opened a brand new Case.
- **DDT-970:** the DDT-919 fix pulled every Matter email into every Case thread on that Matter, so unrelated correspondents (an adjuster, a second carrier, a lienholder) ended up in one huge chain.

The final design threads emails **by correspondent**: a Case's thread shows the messages exchanged with the party that Case is about, gathered from every Case on the Matter.

## 2. What was reported

- **DDT-919, 2026-09-02.** Long email chains "break threading and throw errors", so the case manager could not reply to the most recent message. On the example Case, Reply always landed on an older received email rather than her own newer one. The inbox showed 4 messages where the record held 6.
- **DDT-970, 2026-09-09.** All the emails on a matter were bundled into one massive chain instead of separate threads per recipient or topic, so it was hard to see when a particular carrier, provider or adjuster had last replied. She flagged the risk directly: replying to a merged thread could send sensitive notes or attachments to the wrong recipient. On one matter, 5 emails were being rendered across 21 threads. On 2026-09-11 she reported it was still not resolved on her open tickets; it was rechecked the same day.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `EmailInboxController.getEmailThread` | Apex | Shows Matter emails with no Case parent (#310); scopes them to the correspondent and merges sibling Cases by party (#354); collapses duplicate Message-IDs (#354) |
| `EmailReplyService` | Apex | Sets `In-Reply-To` and `References` on outbound replies; stores `ReplyToEmailMessageId` (#314) |
| `emailThreadView` | LWC | Reply, Reply All and Forward build from the newest message (#314) |
| `EmailInboxControllerTest`, `EmailReplyServiceTest` | Apex tests | See section 7 |

**Out of scope**

- **Emails related to the Case's Account, Opportunity or Lead.** Only the Case's Matter is followed.
- **The Email-to-Case setting that opens a new Case per inbound email.** Unchanged; DDT-970 works around it by merging sibling Cases in the view.
- **`EmailThreadPrintController`.** Not changed, but it uses the same data, so printouts follow the new threading.

## 4. How it works (background)

```
Inbound email to a firm inbox
  → Email-to-Case opens a new Case and stores an EmailMessage with ParentId = that Case
Email sent from a Matter record
  → stored with RelatedToId = Matter and ParentId = null (no Case)
Shared inbox opens a Case
  → getEmailThread builds the thread
  → emailThreadView renders it; Reply / Reply All / Forward prefill from one message
Reply sent from the inbox
  → EmailReplyService sends with Messaging.sendEmail and inserts an EmailMessage
```

## 5. Root cause

### DDT-919

1. **Missing messages.** `getEmailThread` selected only `EmailMessage WHERE ParentId = :caseId`. An email sent from the Matter has `ParentId = null`, so it could never appear. It was correct on the record, absent from the inbox.
2. **Reply quoted the wrong message.** The composer built Reply, Reply All and Forward from `sourceMessage`, which returns the newest **inbound** message. When the newest message in the thread was one the firm sent, it was silently left out of the quoted body and the reply addressed the old sender.
3. **Every reply opened a new Case.** `Messaging.sendEmail` does not return the Message-ID it stamps on the outgoing email, so it was never stored. When the other side replied, Email-to-Case had nothing to match it to and opened a new Case. One client's conversation was spread over 12 Cases this way.

### DDT-970

Introduced by DDT-919's first fix (PR #310). It folded **every** Matter-scoped email into **every** Case thread on that Matter, with no notion of who the email was to. So a fax opened with an adjuster's email, a second carrier's and a lienholder's stacked underneath it. Separately, an email addressed to two firm inboxes is stored once per inbox, so such threads showed every message twice.

## 6. Solution

### DDT-919, PR #310: show Matter emails in the thread

```apex
Set<Id> relatedRecordIds = new Set<Id>();
if (c.Matter__c != null) relatedRecordIds.add(c.Matter__c);
// ... WHERE ParentId = :caseId
//        OR (ParentId = null AND RelatedToId IN :relatedRecordIds)
```

A **set** is used on purpose: a Case with no Matter yields an empty `IN`, which matches nothing. Binding the id directly would become `RelatedToId = null` and pull in unrelated orphan emails. Emails that belong to a different Case are left out, since they already appear as their own inbox row.

### DDT-919, PR #314: reply from the newest message, and keep the thread together

- **`replySource`** (newest message in the thread, whichever side sent it) now drives the prefilled To, Cc, subject, quoted body and forward attachments. `sourceMessage` was left alone because another feature (pull discovery) still uses it.
- **Recipient by direction:** replying to a message the firm sent goes to that message's To address, so the reply reaches the correspondent rather than the shared inbox itself.
- **Reply All** removes the resolved recipient from Cc address by address, so nobody appears in both To and Cc. (Review caught that the first version compared a joined multi-address string and duplicated every recipient.)
- **`replySource` prefers messages on this Case** and falls back to the newest overall, so a Matter email to a different party cannot silently redirect the reply.
- **Threading headers:** outbound replies now carry `In-Reply-To` and `References` set from the newest inbound message that has a Message-ID, and `ReplyToEmailMessageId` is stored. A Case with no such inbound behaves exactly as before.

### DDT-970, PR #354: thread by correspondent

- **Party identity** is each external address, plus its organisation domain. So a carrier's claims mailbox and an adjuster's own mailbox at the same carrier count as one party. Firm addresses are ignored.
- **Consumer mail domains** (Gmail, Yahoo, Hotmail and similar) are excluded from the domain rule: two unrelated people on the same free-mail service are not the same correspondent.
- **Matter emails fold in only for the party this Case is talking to,** and the conversation is gathered from **every Case on the Matter**. One carrier writing in four times used to be four threads; it now reads as one.
- **Sibling Cases merge only for parties the firm has written to.** Fax alerts, e-signature notices and scanner emails arrive from one address and never get a reply; they stay separate.
- **Duplicates collapse on the RFC Message-ID** (`MessageIdentifier`).

## 7. Testing

**PR #314 (Apex).** Four tests on the reply anchor: it is the newest inbound with a Message-ID; a newer inbound without one is skipped; an outbound message is never chosen; with no qualifying inbound the anchor stays null. 55 tests passing in the sandbox. `Messaging.sendEmail` is skipped under test, but the anchor query and the field write run exactly as in production.

**PR #354 (Apex).** Six tests: Matter emails are scoped to the correspondent; they fold in for the same party; a free-mail domain does not merge; repeated Message-IDs are collapsed; one chain per recipient across sibling Cases; inbound-only notifications stay separate.

**Live checks (read-only, against production).** For PR #310, the new query returned 6 messages on the reported Case where the old one returned 4. For PR #314, the prefilled To field was unchanged on both affected Cases.

**Not covered:** a real email round trip proving a counterparty's reply now lands on the same Case. The sandbox used could not send email at all.

## 8. Step by step

1. Open the reported Case and compare the inbox thread with the emails on the Case and its Matter. Missing messages with `ParentId = null` point at the thread query.
2. Check which message Reply is built from. If it is the newest inbound rather than the newest message, that is the composer.
3. Check whether the counterparty's replies are opening new Cases: look for several Cases on the same Matter from the same sender.
4. Change `getEmailThread` to include Matter emails, scoped to the correspondent's addresses and organisation domain, and gather sibling Cases on the Matter.
5. Change the LWC to build replies from the newest message, preferring this Case's own.
6. Set `In-Reply-To`, `References` and `ReplyToEmailMessageId` on outbound replies.
7. Add the tests in section 7.
8. Compare the thread on the reported Case before and after, read-only, then validate, get approval, deploy.

## 9. Rollback

Redeploy the previous `EmailInboxController`, `EmailReplyService` and `emailThreadView`. Data is not changed by any of the three PRs, so rollback is safe, but each problem above returns. Rolling back PR #354 alone brings back the mega-thread from DDT-970.

## 10. Known limits

- **A Matter email to a party that never wrote in has no Case to attach to,** so it no longer appears in the shared inbox. It is still on the Matter's activity timeline.
- **Domain grouping is a heuristic.** Two different organisations on one domain merge; a consumer domain missing from the list would merge unrelated people.
- **The new-Case-per-reply fix is unproven end to end** until a real round trip is checked.
- **The original "error on reply" text** was not captured in the ticket. A draft change on PR #310 targeted a Case publisher-layout error, but it was reverted before merge, so that path was not changed.

## 11. Reuse for similar issues

**How to recognise one:** emails visible on a record but missing from a thread view, replies that quote the wrong message, or one conversation spread over many Cases.

- Check both `ParentId` and `RelatedToId`. Emails sent from a related record often have no Case parent.
- When widening a thread query, decide what scopes it (here, the correspondent). Widening without a scope is exactly what produced DDT-970.
- Bind optional ids through a set, so "no id" matches nothing instead of matching nulls.
- `Messaging.sendEmail` does not return the Message-ID. If you need replies to thread back, set `In-Reply-To` and `References` yourself.
- Collapse duplicate messages on `MessageIdentifier` when an email reaches more than one firm inbox.
