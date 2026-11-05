# New e-signature template: PI Payment Authorization - Client Direct

| | |
|---|---|
| **Ticket** | DDT-1498 |
| **Project** | Dochly |
| **Status** | Live since 2026-11-05 |
| **Priority** | Medium |
| **Reported by** | Firm operations (PI team), 2026-11-03 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature (template editor and sender settings) |
| **Area** | PI settlements: authorising payment of settlement funds into the client's own bank account |
| **Related** | None recorded |
| **Code** | None. A Dochly template build, so nothing is in git |

---

## 1. Summary

The PI team's existing "PI Payment Authorization" document was turned into a Dochly e-signature template, "PI Payment Authorization - Client Direct". The client's details are merged from Salesforce, the client fills in their own bank details, and the signature field is already placed. After the requester's review, the sender name was changed to the PI team's brand, emails were set to go from the PI team's shared inbox, and the account-type checkboxes were fixed so the client can choose Checking or Savings.

## 2. What was reported

This was a request for a new template, not a bug. The PI team needed to send settlement disbursement authorizations for signature without re-entering client data or placing signature fields by hand each time.

What the ticket asked for:

- template name "PI Payment Authorization - Client Direct", type E-Signature Template;
- one signer, the client;
- client data pre-filled from Salesforce;
- the client's signature field pre-placed;
- generated directly from Salesforce.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `PI Payment Authorization - Client Direct` | Dochly e-signature template | Built from the existing `PI Payment Authorization` document |
| Same | Dochly template | Client-fillable bank detail fields, client signature, client name fields |
| Same | Dochly template | Account type checkboxes fixed so the client can choose Checking or Savings |
| Sender name and address | Dochly | Sender name changed to the PI team's brand; emails sent from the PI team's shared inbox. Where in Dochly this was set is not recorded |

**Out of scope**

- **Payments into a third party's account**, which need more than one signer. The ticket says that will be a separate template, requested later.
- **The email message the requester supplied.** On the ticket the requester shared the wording of a saved message template, "Payment Authorization", with the subject "To Sign - Payment Authorization". Whether it was set up as part of this ticket is not recorded.

## 4. How it works (background)

```
Staff generate "PI Payment Authorization - Client Direct" from Salesforce
  -> Dochly merges the client's name and details
  -> the envelope is emailed to the client from the PI team's shared inbox
  -> the client fills in their bank details, picks Checking or Savings, and signs
  -> the completion email goes to the client and to the PI team's shared inbox
```

## 5. Root cause

Not applicable: this was a new template. The follow-up changes came from the requester's test send (see 7).

## 6. Solution

### 6.1 The template

Built in Dochly from the existing payment authorization document. The builder asked whether the bank fields would be filled in by the client or by staff before sending. The requester answered that they are **client-fillable fields, because they are the client's bank information.** So staff never type a client's bank details; the client enters them while signing.

### 6.2 Changes after the requester's test send

| Requested | Outcome recorded on the ticket |
|---|---|
| Show the PI team's brand, "DDT Injury Team", instead of "DDT Law Group", as PI uses the Injury branding | Changed |
| Send from the PI team's address | Emails now sent from the PI team's shared inbox |
| Update the address to the firm's new office | Not recorded in the closing comment |
| Let the client choose either Checking or Savings | The checkbox issue was fixed |
| Send the completion email to the shared inbox instead of the tester's own | Explained, not changed (see 7) |

## 7. Testing

The requester sent the template to themselves as a test. They reported that everything else was "working and syncing properly", and raised the items in 6.2.

**The completion email.** The requester received the completion email at their own work address. That happened because they had used their own address as the recipient while testing. When the document is sent to a real client, the completion email goes to the client's address and to the PI team's shared inbox.

**Not covered:** a send to a real client, recorded on the ticket.

## 8. Step by step

1. **Get the source document** and the list of fields: which are merged from Salesforce, which the client fills in, which staff fill in.
2. **Ask who fills in sensitive fields** such as bank details. Here the client does.
3. **Build the e-signature template** in Dochly, named as requested, with the client as the only signer.
4. **Add the merge fields** for the client's details, the client-fillable fields, and the client's signature.
5. **Make a pair of exclusive choices** (Checking or Savings) selectable as either one.
6. **Set the sender name and sender address** to the team that owns the process, in its own branding.
7. **Send a test to yourself**, and expect the completion email to come back to you because you were the recipient.
8. **Have the requester run the same test** and work through what they find.

## 9. Rollback

Deactivate the template in Dochly. The original `PI Payment Authorization` document is unchanged.

## 10. Known limits

- **Single signer only.** A payment into anyone else's account needs the separate multi-signer template the ticket mentions.
- **The office address.** Whether it was updated on this template is not recorded in the ticket.

## 11. Reuse for similar issues

**How to recognise one:** a document staff send for signature is being filled in and signature-placed by hand each time.

- **Decide who fills in each field before building.** Data only the signer knows (bank details) should be signer-fillable, so staff never handle it.
- **Use the right brand for the practice area.** PI documents go out as the PI team, not the general firm name.
- **Test sends go back to the tester.** A completion email arriving at your own address after a test is expected, not a misrouting.
- **Check every checkbox group as the signer** to make sure each option can actually be chosen.
