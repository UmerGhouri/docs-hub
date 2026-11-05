# PI inbox spam folder not synced

| | |
|---|---|
| **Ticket** | DDT-1463 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-29 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-10-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Shared Inbox), Gmail |
| **Area** | PI shared inbox: which Gmail email reaches Salesforce |
| **Related** | DDT-1465 (SOS inbox search, also a shared-inbox question) |
| **Code** | Not applicable: answered as designed, no change made |

---

## 1. Summary

A staff member expected the Spam filter in the Salesforce PI shared inbox to show the PI Gmail account's spam folder. It does not, by design: Gmail's spam folder is not synced into Salesforce. The Spam filter in Salesforce only shows emails marked as spam inside Salesforce. Legitimate senders that land in Gmail spam are fixed on the Gmail side.

## 2. What was reported

On 2026-10-27 a staff member reported that the Spam filter in the Salesforce "Email Inbox — Injury" showed "No emails match your filters", while the PI Gmail account's spam folder held 13 messages from 1 to 22 October. Some looked like real business email, such as case status requests and medical billing statements. They asked whether that was intended, noting that Gmail deletes spam after 30 days, so anything there is lost if nobody checks Gmail directly.

## 3. Scope

**In scope:** answering whether Gmail spam should appear in Salesforce, and how to stop legitimate senders landing in spam.

**Out of scope:** syncing the Gmail spam folder into Salesforce. That would be a larger custom change needing the firm owner's approval, and the requester did not ask for it once the design was explained.

## 4. How it works (background)

```
Email arrives at the PI Gmail account
  -> Gmail filters it: Inbox, or Spam
  -> Inbox mail is synced into the Salesforce Shared Inbox   (Gmail spam is not)
  -> in Salesforce, staff set each email's Status: Open, Resolved, or Spam
```

The Salesforce Spam filter lists emails whose Salesforce Status is Spam, that is, emails a user marked as spam inside Salesforce. It is not a mirror of Gmail's spam folder.

## 5. Root cause

Not a fault. The Spam status in Salesforce and Gmail's spam folder are two different things with the same name. Gmail spam is left out of Salesforce on purpose: it is mostly junk and phishing, some of it unsafe to open, and syncing it would bury real client email in the PI inbox.

## 6. Solution

The requester was given the explanation on 2026-10-28, with two options:

- **A real sender keeps landing in spam:** fix it in Gmail. Whoever has access to the team inbox, or a Workspace admin, can mark the message "Not spam" or add a filter for that sender so it goes to the inbox. This takes a couple of minutes.
- **Syncing the spam folder into Salesforce:** a bigger custom change, needing the firm owner's approval first.

The requester already had access to the Gmail account, had been marking real emails as not spam there, and was satisfied with the answer.

## 7. Testing

Not applicable: no change was made.

## 8. Step by step

When a staff member asks why Gmail spam is not in the Salesforce inbox:

1. **Explain the two meanings of "Spam"**: the Salesforce Status set by users, and Gmail's spam folder, which is not synced.
2. **Look at what is in Gmail spam.** If real senders are there, list them.
3. **In Gmail, mark each real message "Not spam"**, or create a filter for the sender with "Never send it to Spam".
4. **Check the next email from that sender** reaches the Salesforce inbox.
5. **If the team wants Gmail spam in Salesforce anyway**, raise it as a separate change for the firm owner to approve.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **Gmail deletes spam after 30 days.** A real email that lands in spam and is not noticed in Gmail is lost, and never reaches Salesforce.
- **Someone has to check Gmail spam** now and then, since Salesforce gives no sign it holds anything.

## 11. Reuse for similar issues

**How to recognise one:** "this filter in Salesforce doesn't show what I see in Gmail".

- **Check what the Salesforce filter actually reads.** A status with the same name as a Gmail folder is not necessarily synced from it.
- **Fix sender problems at the source.** Gmail filters and "Not spam" fix delivery for every future email; nothing in Salesforce can.
- **Treat syncing junk folders as a deliberate decision**, given the phishing risk and the noise it adds.
