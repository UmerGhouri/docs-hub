# PI contract texts came from the wrong number, and the send prompt still said DocuSign

| | |
|---|---|
| **Ticket** | DDT-1159 |
| **Project** | DocuSign Template Migration (replace with Dochly) |
| **Status** | Live since 2026-10-01 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-09-30 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow / LWC), Dochly e-signature, Heymarket texting |
| **Area** | PI intake: texting the signing link to the client |
| **Related** | DDT-1070 (PI contract sender moved to Dochly), DDT-1282 (wording and short link on the same texts) |
| **Code** | PR #583 in the `ddt-salesforce` repo |

---

## 1. Summary

After the PI contract moved to Dochly, firm operations tested a send and found two problems. The signing-link text reached the client from the firm's main texting number instead of the PI texting number that PI clients know and reply to. And the confirmation prompt in the send wizard still said "DocuSign", with a missing space before the phone number.

The text is sent by an existing Salesforce flow that Dochly hands its messages to. That flow now picks the PI number for PI documents and keeps the main number for everything else. The prompt now says Dochly and the spacing is fixed. No Apex was needed.

## 2. What was reported

From the firm's test of the Dochly contract send on 2026-09-29:

1. **The text came from the wrong number.** The ticket believed Dochly sent the text itself, from a number the firm does not monitor. It asked for the link to go out through Heymarket from the PI inbox, so it lands in the thread the PI team already uses with that client.
2. **The prompt said** "This sends the retainer to ... through DocuSign." It also rendered "mobile number" run straight into the digits.

Requirements kept from the ticket: a bad or missing phone must never block the contract, and no new scheduled job.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Test_Dochly_SMS` | Record-triggered flow | First committed to git exactly as production ran it (v13), then changed: the Heymarket inbox is chosen by a new `smsInbox` formula instead of a fixed value |
| `Test_Dochly_SMS` | FlowDefinition | Activates version 14 |
| `convertToHiredMatter.html` | LWC template | "DocuSign" changed to "Dochly"; the missing space fixed |

**Out of scope**

- **Renaming the flow.** It is called `Test_Dochly_SMS` but is the production SMS path for every Dochly signature. A flow's API name cannot be renamed in place: deploying under a new name creates a second flow, and until the old one is deactivated the client gets two texts. Left for its own ticket.
- **Non-US mobile numbers** (see section 10).
- **The email from Dochly,** which stays as it was.

## 4. How it works (background)

The ticket assumed Dochly texted the client itself. It does not. Dochly **queues** a text, and a flow in the org sends it:

```
Dochly marks a signer for SMS
  -> sets dochly__SMS_Send_Token__c and dochly__SMS_Status__c = 'Queued' on the Signer
  -> puts the message in dochly__SMS_Outbound_Body__c
Test_Dochly_SMS  (record-triggered on the Signer)
  -> sends through heymarket__SendUpto5KMessagesAction  (inbox: fixed value)
  -> reports back through dochly__DochlyLogSmsResult
```

That flow was built directly in production, was 13 versions deep, sent real client texts, and **existed in no commit on any branch** before this ticket.

## 5. Root cause

`Test_Dochly_SMS` had the Heymarket inbox hard-coded to the firm's main texting number and no branching. So **every** Dochly text in the org went out from the main number: criminal defense retainers, court appearance documents, FOIA consents and PI contracts alike.

**The ticket's first guess was wrong.** It was written around building a Heymarket callout in `DochlyContractService`, with warnings about callouts after DML. None of that applied: the bridge flow already existed and Dochly already queued into it.

The prompt was simply never updated when sending moved from DocuSign. The spacing bug: three of the four gaps in that sentence used `&nbsp;`, and the fourth relied on a line break, which the LWC template compiler trims.

## 6. Solution

### 6.1 Choose the inbox from the document's parent object

The flow now reads a formula instead of a fixed value:

```
IF(OR(
    {!$Record.dochly__Signature_Request__r.dochly__Document__r.dochly__Object_API_Name__c} = "Lead",
    ... = "Matter__c",
    ... = "Record_Request__c",
    ... = "Damages__c",
    ... = "Lien__c"),
  "<PI texting number>",
  "<main texting number>")
```

Why this way:

- **Flipping the fixed value would have moved every Dochly text** in the org onto the PI line. Last 7 days in production: Opportunity 419, `Court_Appearance__c` 146, `Damages__c` 27, `Matter__c` 6, Lead 5. PI documents are a small minority.
- **The Signer does not carry its parent object,** so the formula hops Signer to Signature Request to Document to `dochly__Object_API_Name__c`, the same hop the existing `Dochly_Signature_Log_Activity` flow uses.
- **A blank or broken link falls to the main number,** so an unresolvable parent keeps the old behaviour rather than silently moving a text to the PI line.
- **The numbers stay in the formula, not custom metadata.** The value replaced was already a fixed value in the same element; moving two numbers into metadata would add an object and a deploy dependency nobody asked for.

### 6.2 Commit production's version first

Commit 1 is the flow retrieved from production and committed unchanged, so there is something to go back to. Commit 2 is the behaviour change.

### 6.3 Activate through a FlowDefinition

A flow deployed with status Active still lands as Draft when a new version is created. Without the `FlowDefinition`, the deploy would go green and v13 would keep sending from the old number. The `FlowDefinition` names version 14, which is only correct if production is still on v13 at deploy time, so the version is re-read just before deploying.

### 6.4 The prompt

Only the vendor name changed in the wording, and the fourth gap now uses `&nbsp;` like its neighbours. The prompt is read only by the staff member sending; nothing in it reaches the client. Checked: none of the 29 SMS bodies Dochly had generated and none of 200 delivery records contained "DocuSign".

## 7. Testing

**Sandbox (2026-10-01).** Proved:

- the Apex puts the mobile on the signer and sets the notification channel to both email and SMS
- Dochly queues the text, with the body generated
- the bridge flow fires on the right trigger and runs to the end
- the formula's input resolved correctly to `Lead`

**Not proved in the sandbox:** which inbox was used. Heymarket does not transmit from the sandbox (no message rows on the test day, newest row 2026-09-10), and the provider message Id came back blank. So "Sent" there only meant the action call did not throw.

Getting that far took two vendor-side fixes: the sandbox's Dochly package was upgraded to match production, and Dochly had to enable SMS in a protected setting only they can change.

**Production.** Deployed 2026-10-01. The ticket's closing comment records that PI documents text from the PI number, everything else stays on the main number, and the prompt says Dochly. **Still active (2026-11-05):** `Test_Dochly_SMS` reads as active in production.

**Not covered:** `convertToHiredMatter` has no LWC tests, so nothing in CI reads the prompt; it has to be checked on screen. The ticket's check that a Lead with no mobile still sends by email is not separately recorded.

## 8. Step by step

1. **Find what actually sends the text.** Look at what Dochly writes on the Signer and which flow reacts to it.
2. **Retrieve that flow from production and commit it unchanged.**
3. **Size the traffic by parent object** over a recent window, so you know what a change would move.
4. **Replace the fixed inbox with a formula** on the document's parent object, defaulting to the old number.
5. **Fix the prompt wording and spacing** in `convertToHiredMatter.html`.
6. **Re-read production's current flow version,** and set the `FlowDefinition` to the next one.
7. **Deploy** the flow, the `FlowDefinition` and the LWC.
8. **Verify:** Setup shows the new version active; the prompt reads correctly on screen; one PI contract text arrives from the PI number; one non-PI document still texts from the main number. Check with:

```sql
SELECT heymarket__Targeted_Phone__c, heymarket__Inbox_Number__c, heymarket__Status__c
FROM heymarket__Message__c WHERE CreatedDate = TODAY ORDER BY CreatedDate DESC LIMIT 5
```

Initial-send texts are rare (of the 29 texts Dochly had queued in production by then, 2 were invites, 15 reminders and 12 completions), so if the first contract produces no text, wait for the reminder before concluding the routing is broken.

## 9. Rollback

Not recorded in the ticket. Version 13 is the original, committed unchanged before the edit; re-activating it with a `FlowDefinition` (`activeVersionNumber` 13) would return all Dochly texts to the main number. The LWC change is cosmetic and need not be rolled back.

## 10. Known limits

- **Non-US mobiles get email only, silently.** The phone normaliser accepts only 10 digits, or 11 starting with 1. Anything else sends by email with no text, while the prompt still says a text went out. Widening it is a product decision; none of about 50,000 outbound Heymarket messages over 90 days had gone to a non-US number.
- **A Contact-parented document would not route to the PI number.** There were none at the time.
- **Adding a new PI object** (a new kind of PI document) means adding it to the formula, or its texts go out from the main number.
- **The flow is still named `Test_Dochly_SMS`.**

## 11. Reuse for similar issues

**How to recognise one:** a vendor's notifications come from the wrong sender, or a change request assumes the vendor sends something itself.

- **Find what really sends the message** before designing a fix. Here the ticket planned new Apex for a job an existing flow already did.
- **Commit a production-only flow unchanged before editing it,** so rollback has a target.
- **Route by the record the document belongs to,** with the old behaviour as the default, rather than flipping a global value.
- **Size the traffic by category first,** so the effect of a change is known.
- **Activate new flow versions with a `FlowDefinition`,** and re-read the live version number right before deploying.
- **In a sandbox, "Sent" may only mean "did not throw."** Look for the provider's message Id or the vendor's own record before calling it delivered.
- **Check UI text on screen.** A green pipeline does not render an LWC.
