# Shared inbox Reply All missing recipients and wrong signature address

| | |
|---|---|
| **Ticket** | DDT-285 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-27 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-07 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Apex), email templates |
| **Area** | Shared email inbox: replying to client emails |
| **Related** | DDT-712 (BCC in the shared-inbox composers), DDT-919 (reply from the newest message) |
| **Code** | PR #53 in the ddt-salesforce repo |

---

## 1. Summary

In the Salesforce shared inbox, Reply All only addressed the original sender; everyone else on the email was dropped. The reply composer could already work out the Cc list, but the thread view never passed it in. The fix wires the Cc list through and also leaves the inbox's own address out of it, so the firm doesn't Cc itself. The same ticket reported an old office address in the email signature, which was corrected in the email templates.

## 2. What was reported

1. **Reply All dropped recipients.** Only the original sender appeared; the other people the email was addressed to were not carried into the reply.
2. **Wrong office address in the signature.** The general signature footer still showed the firm's previous office address.

Why it matters: dropping recipients breaks threads and silently leaves people out of replies. A wrong address on outbound email is client-facing.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `emailThreadView` | LWC | Passes `default-cc` to the reply composer; leaves the inbox's own address out of the Reply All Cc |
| `EmailInboxController` | Apex class | New `ThreadDTO.replyFromAddress`, filled by a new `resolveInboxAddress` method from the Case's source inbox queue |
| Email templates | Templates | Office address in the signature corrected (per the resolution note, "in the Dochly email templates") |

**Out of scope**

- **The reply composer itself (`emailReplyComposer`).** It already accepted a Cc value; it was never given one.
- **Plain Reply.** Unchanged: it still addresses only the sender.

## 4. How it works (background)

```
User opens a thread in the shared inbox (emailThreadView)
  -> EmailInboxController.getEmailThread(caseId) returns a ThreadDTO
  -> user clicks Reply All
  -> emailThreadView works out To (the sender) and Cc (everyone else)
  -> it renders c-email-reply-composer with the defaults
```

The thread view had a `defaultCc` getter that built the Cc list from the original message's To and Cc. The composer had a `default-cc` property to receive it. The two were never connected in the template.

## 5. Root cause

`emailThreadView.html` rendered the composer with `default-to` and `default-subject`, but **no `default-cc` binding**. So the Cc list was computed and thrown away, and every Reply All went out with an empty Cc.

Once the Cc was wired through, a second problem would have appeared: the inbox's own address is usually on the original email's To line, so Reply All would have Cc'd the inbox itself. The fix handles that at the same time.

## 6. Solution

### 6.1 Pass the Cc list to the composer

```html
<c-email-reply-composer
    case-id={thread.caseId}
    default-to={defaultTo}
    default-cc={defaultCc}
    default-subject={defaultSubject}
    ...
```

### 6.2 Tell the thread view its own inbox address

```apex
/**
 * The shared-inbox mailbox address for a Case's source queue, used by the
 * reply composer to drop this inbox's own address from a Reply All Cc so
 * we never CC ourselves. Null when the queue or its email isn't resolvable.
 */
private static String resolveInboxAddress(String inboxQueueName) {
    if (String.isBlank(inboxQueueName)) return null;
    List<Group> queues = [
        SELECT Email FROM Group
        WHERE Type = 'Queue' AND DeveloperName = :inboxQueueName
        LIMIT 1
    ];
    return queues.isEmpty() ? null : queues[0].Email;
}
```

`getEmailThread` sets `dto.replyFromAddress = resolveInboxAddress(c.Source_Inbox_Queue__c)`.

Why: each shared inbox is a queue with its own email address, and the Case records which queue it came from. Reading the address from the queue means no address is hardcoded, and a new inbox works without a code change.

### 6.3 Leave the inbox out of the Cc

```js
// Exclude this inbox's own reply-from address so we don't CC ourselves
// on a Reply All (the inbox address is typically the original "To").
const selfAddr = (this.defaultFrom || '').trim().toLowerCase();
if (selfAddr) seen.add(selfAddr);
```

The sender's address was already in the `seen` set, so they go in To and not again in Cc.

### 6.4 Signature address

Corrected in the email templates. The ticket records only that it was done; the template names are not recorded.

## 7. Testing

- **Sandbox check (from the commit):** in the partial sandbox, Reply All pre-filled To with the sender and Cc with every other original recipient.
- **Resolution note (2026-07-27):** Reply All puts the original sender in To and all other recipients in Cc, minus the inbox's own address.
- **Not covered:** no automated test is recorded for the Cc wiring.

## 8. Step by step

1. **Reproduce.** Open a shared-inbox thread whose last email had several recipients. Click Reply All and note that Cc is empty.
2. **Check the binding.** In `emailThreadView.html`, find `c-email-reply-composer` and confirm whether `default-cc` is passed.
3. **Add `default-cc={defaultCc}`** (6.1).
4. **Add `replyFromAddress` to `ThreadDTO`** and fill it from the source queue's email (6.2).
5. **Exclude it in the Cc builder** (6.3).
6. **Validate and deploy** `EmailInboxController` and `emailThreadView` together, since the LWC reads the new DTO field.
7. **Test in a sandbox:** Reply All on a multi-recipient email shows the sender in To, everyone else in Cc, and not the inbox address.

## 9. Rollback

Redeploy the previous `emailThreadView` and `EmailInboxController`. That brings back the empty Cc. The extra DTO field is harmless on its own, so the Apex can stay if only the LWC is rolled back.

## 10. Known limits

- **Only the inbox's main address is excluded** by this change. An inbox with older or alias addresses would still Cc those. That was handled later, in PR #641, which also excludes an inbox's other addresses.
- **Which message Reply All works from** was changed later under DDT-919, which makes replies work from the newest message.

## 11. Reuse for similar issues

**How to recognise one:** a value is clearly computed in a component (a getter exists) but never shows up in the child it was meant for.

- **Check the template binding first.** An unbound `@api` property fails silently: no error, just the default.
- **When you fix Reply All, think about who should *not* be on it**: the sender (already in To), and the mailbox you are replying from.
- **Read addresses from configuration** (the queue's email), not from code.
