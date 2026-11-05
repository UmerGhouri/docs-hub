# Dochly signature texts reworded, with a short link

| | |
|---|---|
| **Ticket** | DDT-1282 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-09 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-10-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / Flow / Visualforce), Dochly e-signature, Heymarket |
| **Area** | Text messages sent to clients when a signature is requested, reminded or completed |
| **Related** | DDT-1159 (Dochly text inbox routing) |
| **Code** | PR #671 in the `ddt-salesforce` repo |

---

## 1. Summary

Dochly's signature-request texts named the sender and the document, and carried a 238-character signing link that looked like spam. Dochly's wording can't be configured, but Dochly only drafts the text: our own flow sends it through Heymarket. So the flow now rewrites the draft into fixed DDT Law Group wording and swaps the long link for a short one that redirects to the same signing page.

## 2. What was reported

A typical invite read: `<sender name> has requested your signature on <document title> - <client name> - <timestamp>. Review & sign: <238-character URL>`.

The request was:

- every Dochly signature-request text reads exactly `DDT Law Group has requested your signature on this document: [LINK]`;
- `[LINK]` is a short link, not the raw signing URL with its full token;
- the short link still opens the correct signing page.

A follow-up question asked why the texts take so long to arrive.

While checking production, a third problem turned up: the completion text was broken. Dochly fills one slot in it with the email subject, so production was sending sentences such as "Please sign the attached. is now complete."

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Test_Dochly_SMS` | Flow | **Edited**: builds the message before sending, records what was sent, saves the short-link code. Version 15 activated |
| `DochlySmsMessageBuilder` + test | Apex class | New. Picks the wording and makes the short-link code |
| `DochlyShortLinkController` + test | Apex class | New. Turns a short-link code back into the full signing URL |
| A new Visualforce page on the Dochly signing site | Visualforce page | New. Serves the short link |
| `dochly__DH_Signer__c.Short_Link_Code__c` | Field (Text 16, unique, case-sensitive) | New |
| `dochly__DH_Signer__c.Short_Link_Target__c` | Field (Long Text) | New. The full URL the code redirects to |
| `dochly__DH_Signer__c.SMS_Sent_Body__c` | Field (Long Text) | New. The text actually sent |
| `Dochly_Short_Link_Guest` | Permission set | New. Lets the signing site open the page |
| `Dochly_SMS_Short_Link_Fields` | Permission set | New. Field access for staff who troubleshoot client texts |

The flow was the only existing component changed. `dochly__DH_Signer__c` is a managed object, and it reports `IsCustomizable = true`, so subscriber fields can be added to it.

**Out of scope**

- **Inbox routing** (which Heymarket inbox a text goes out from). Unchanged from DDT-1159.
- **Renaming the flow.** It is still called `Test_Dochly_SMS`. A rename creates a second flow and a double text.
- **A branded domain** for the link. That needs DNS and a Salesforce custom domain; the page and codes would carry over unchanged.
- **Delivery speed.** See section 5.

## 4. How it works (background)

```
Dochly creates a signature request
  -> Dochly drafts the text into dochly__SMS_Outbound_Body__c and marks it Queued
  -> flow Test_Dochly_SMS fires
  -> HeyMarket SMS action sends the draft, unchanged
  -> Mark Sent
```

Dochly writes the wording itself, in the managed package. Our flow only hands it over.

## 5. Root cause

The ticket guessed the wording lived in "a custom Dochly SMS template / Apex class / flow". Checked against production:

- **Dochly's wording can't be configured.** The package ships no custom labels (`ExternalString` with `NamespacePrefix = 'dochly'` returns 0 rows), no template or config object has an SMS field, and `dochly__Dochly_Setting__mdt` has none.
- **Dochly doesn't send the text.** It drafts it into `dochly__SMS_Outbound_Body__c`; `Test_Dochly_SMS` passes that field to Heymarket unchanged.

So the only place to change the wording is our flow, between the draft and the send.

**Why not Bitly.** The link carries the signing token, so whoever holds it can sign. Bitly would put every client's live signing link with a third party, adds a paid API and a callout, and plain `bit.ly` links are often filtered by carriers, which is the spam problem the ticket wanted fixed. Dochly has a `Short_Code__c` field on the signer, but it was blank on every record and the package has nothing that serves it.

**The delay question.** On all three production invites checked, Dochly stamped `SMS_Notified_Date__c` within 1 to 3 seconds of creating the request. That is Dochly's handoff time, not a delivery receipt, so any delay is after the handoff, in Heymarket or the carrier. This change doesn't address it. The ticket's closing comment notes the texts arrived right away in testing.

## 6. Solution

### 6.1 The wording

| Text | After |
|---|---|
| Invite | `DDT Law Group has requested your signature on this document: [short link]` |
| Reminder | `Reminder: DDT Law Group is still waiting for your signature on this document: [short link]` |
| Completion | `Your document with DDT Law Group has been signed. View or download your signed copy: [short link]` |

The invite is the ticket's text, word for word. The reminder and completion lines follow it, as agreed on 2026-10-09. The short link is 63 characters against 238, and the whole invite is 124 characters, one SMS segment.

### 6.2 The flow

1. Dochly queues a text and the flow fires, as before.
2. **Build SMS** (`DochlySmsMessageBuilder`) picks the wording from `dochly__SMS_Message_Type__c`, takes Dochly's URL out of the draft, and gives the signer a code, reusing the one they already have.
3. **HeyMarket SMS** sends that message instead of the draft.
4. **Mark Sent** also records the text actually sent in `SMS_Sent_Body__c`. Dochly's own field keeps Dochly's draft.
5. **Has Short Link → Save Short Link** stores the code and the full URL on the signer.
6. When the client taps the link, `DochlyShortLinkController` looks up the code and redirects to the stored URL. The client lands on exactly the page the long link opened, with the same token, so Dochly's expiry, void and already-signed handling all still apply.

### 6.3 Choices a reviewer will ask about

- **The builder never throws and never writes.** The completion text is queued inside Dochly's signing transaction. An unhandled error would roll back the client's signature. Anything unrecognised (a new message type, a draft with no Dochly URL) is sent exactly as Dochly drafted it.
- **The code is saved after the send, not before.** A database write before Heymarket's callout would leave uncommitted work ahead of a callout, which fails.
- **Has Short Link** exists so a fallback text, which returns no code, never blanks a code the signer already holds and breaks the links in their earlier texts.
- **The code is the security boundary.** It is a random 10-character code from `Crypto.getRandomInteger`, made once per signer, using an alphabet without look-alike characters (`0 o O 1 l I`). Codes are case-sensitive. SOQL compares text without case, so generation rejects a code that matches an existing one ignoring case, and the redirect confirms the exact match in Apex.
- **No open redirect.** The redirect follows only an `https` URL on the same host the short-link request arrived on, pointing at Dochly's signing page. The first version checked only the path; the AI review on PR #671 caught that a lookalike URL on another host would have passed, and the host check was added before merge.
- **The link follows the org.** It is built from the host in Dochly's own URL, so a sandbox text points at the sandbox.

```apex
@TestVisible
static final String INVITE_PREFIX = 'DDT Law Group has requested your signature on this document: ';
@TestVisible
static final String REMINDER_PREFIX = 'Reminder: DDT Law Group is still waiting for your signature on this document: ';
@TestVisible
static final String COMPLETION_PREFIX = 'Your document with DDT Law Group has been signed. View or download your signed copy: ';
```

## 7. Testing

**Apex tests.** 14 of 14 passed in the sandbox after the host check was added.

| Test class | What the tests prove |
|---|---|
| `DochlySmsMessageBuilderTest` | Invite uses the ticket's exact text; reminder and completion get their own wording; a signer with no code gets a fresh 10-character one; codes in one batch all differ; a code already held is never handed out again; anything unrecognised is sent exactly as drafted; text after the URL doesn't leak into the redirect; the link keeps the site it came from |
| `DochlyShortLinkControllerTest` | A known code redirects to the full URL; the code is case-sensitive; unknown, missing and malformed codes show "This link is not valid"; a target that isn't Dochly's signing page is never followed; Dochly's path on another host is never followed |

**Sandbox, end to end, on a real Dochly send:**

- An invite went to Heymarket in the new wording with a short link, and the code and full URL were saved on the signer.
- A reminder used the reminder wording with the **same** code.
- Opened logged out, the short link redirected to Dochly's URL with the token byte-identical, and the signing page loaded.
- An unknown code, and the real code in the wrong case, both showed "This link is not valid".
- A test through the UI reached `Viewed`, so the short link opened the document, and the text arrived at the same time.
- A signer temporarily pointed at another host showed "This link is not valid", and was restored afterwards.

**Live check (2026-11-05):** in production, `DochlySmsMessageBuilder` and `DochlyShortLinkController` were created on 2026-10-09, and `Test_Dochly_SMS` is on **version 15, Active**.

**Not covered:**

- A text arriving from production's own inboxes. The sandbox test sent from one inbox only.
- Signing all the way through after the redirect. The sandbox test reached `Viewed`, not `Completed`.
- The completion text end to end. It needs a real signature; unit tests cover its wording.
- Link previews. The redirect is a Visualforce JavaScript redirect, not an HTTP 302, so a link-preview bot won't follow it and a phone preview may show no page title.
- The production deploy's test run is not recorded in the ticket.

## 8. Step by step

1. **Re-read the production flow version.** The FlowDefinition activates version 15 because production was on 14. If someone has edited the flow since, the deploy lands as a different version and activates the wrong one while still reporting success.

    ```
    sf data query -o <prod-alias> --use-tooling-api \
      -q "SELECT VersionNumber, Status FROM Flow WHERE Definition.DeveloperName = 'Test_Dochly_SMS' ORDER BY VersionNumber DESC LIMIT 1"
    ```

2. **Retrieve the production flow** and confirm it matches the copy in git before the change.
3. **Deploy** the three signer fields, both classes and their tests, the page, both permission sets, the flow and its FlowDefinition.
4. **Immediately assign the permission sets**: `Dochly_Short_Link_Guest` to the Dochly signing site's guest user, and `Dochly_SMS_Short_Link_Fields` to the deployer. Use `Database.insert(rows, false)` and print each result. Until the guest has the page, every short link shows "Authorization Required".
5. **If step 4 was delayed**, find signers that got a code in the gap and resend from Dochly.
6. **Requester's check:** send one document to your own mobile, confirm the exact wording, tap the link, sign, and confirm the completion text uses the same code.

## 9. Rollback

Reactivate flow version 14 in Setup, or redeploy the previous flow with a FlowDefinition pointing at it. Texts go back to Dochly's wording and long URL. **Leave the fields, page and classes in place**, so links already sent keep working.

## 10. Known limits

- **A new Dochly message type** is sent in Dochly's own wording until the builder learns it. That is deliberate: it can't fail a signature.
- **If a Dochly upgrade changes its signing page name or URL shape**, the builder stops recognising the URL and every text falls back to Dochly's draft and long link.
- **Whoever holds a short link can sign**, exactly as with the long link. The code is only as private as the text message.
- **Delivery delay** after the handoff to Heymarket is not addressed.

## 11. Reuse for similar issues

**How to recognise one:** a managed package's wording or behaviour can't be configured, but the package hands its output to something we own (a flow, a trigger, an integration) before it reaches the customer.

- **Look for the handoff.** Rewrite at the point where our automation picks up the package's output, rather than fighting the package.
- **Never throw inside someone else's transaction.** If our code runs inside a signing or payment transaction, fall back to the original output on any error.
- **No DML before a callout** in the same transaction. Save afterwards.
- **For anything that stands in for a token**, use a random code, compare it exactly, and check the redirect target's host, not just its path.
- **Keep old links working on rollback**: switch the sending logic back, leave the redirect in place.
