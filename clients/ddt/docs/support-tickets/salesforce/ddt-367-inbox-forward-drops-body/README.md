# Shared inbox forwards drop the email body and attachments

| | |
|---|---|
| **Ticket** | DDT-367 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-27 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-16 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Shared inbox: forwarding an email to a colleague |
| **Related** | None recorded |
| **Code** | PR #64 in the `ddt-salesforce` repo |

---

## 1. Summary

Forwarding an email from the firm's shared inbox in Salesforce sent only the forwarder's own note. The original message and its attachments were missing, so the recipient had to search Salesforce for the subject line to find the real email.

The thread view already built the quoted original and the attachment list for a forward, but never passed them to the composer, and the composer had nowhere to receive them. PR #64 wired them through.

## 2. What was reported

On 2026-07-16 a staff member received a forward from the shared office inbox containing only "please see the email below". There was no quoted original and no attachment. The example was a monthly statement email whose attachment did not come through. The ticket asked for the full original body and all attachments on every forward.

The ticket guessed the cause might be Email-to-Case, Email Relay or Gmail settings. It was none of these (see 5).

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `emailReplyComposer` | LWC | Two new public properties, `defaultBody` and `initialAttachments`, used to seed the body and attachment list when the composer opens |
| `emailThreadView` | LWC (template) | Passes its existing forward body and forward attachment list into the composer |

**Out of scope**

- **Replies.** Both new properties are empty for a reply, so reply behaviour is unchanged.
- **Apex and email settings.** The change is front-end only; nothing on the server or in Email-to-Case was touched.
- **Gmail-side forwards.** The fix is in the Salesforce shared inbox screen. Nothing in the sources says Gmail forwards were changed.

## 4. How it works (background)

The shared inbox is a custom Lightning screen. Opening a thread shows `emailThreadView`; pressing **Forward** sets the mode to `forward` and opens `emailReplyComposer` inside it.

```
Forward clicked
  -> emailThreadView: replyMode = 'forward', showComposer = true
  -> emailReplyComposer opens with To, Cc, Subject pre-filled
  -> user writes a note, presses Send
```

`emailThreadView` already had two getters for forwarding:

- `defaultBody` builds a `---------- Forwarded message ---------` block with the From, Date, Subject, To and Cc lines and the original HTML (or escaped plain text) body.
- `forwardAttachments` lists the original message's attachments by file Id, name and size.

## 5. Root cause

The two getters existed but were **never bound to the composer**. The template passed only `default-to`, `default-cc` and `default-subject`, and `emailReplyComposer` declared no property to receive a body or attachments. So the composer always opened empty apart from the address lines and subject, and whatever the user typed was all that was sent.

The ticket's guesses (Email-to-Case, Email Relay, attachment size limits) were wrong. The forward was being built correctly in one component and dropped at the hand-off to the next.

**Evidence:** the template before PR #64 binds three `default-*` attributes and nothing else, while `emailThreadView.js` at the same commit already contains `get defaultBody()` and `get forwardAttachments()`.

## 6. Solution

**Composer: accept a body and attachments.**

```js
// Forward pre-fill: quoted original body + the original message's attachments.
// Empty for replies, so reply behavior is unchanged.
@api defaultBody = '';
@api initialAttachments = [];

connectedCallback() {
    this.toAddress = this.defaultTo || '';
    this.ccAddress = this.defaultCc || '';
    this.subject = this.defaultSubject || '';
    // Seed the quoted original + forwarded attachments (forward mode only).
    this.body = this.defaultBody || '';
    this.attachments = Array.isArray(this.initialAttachments)
        ? this.initialAttachments.map((a) => ({ id: a.id, name: a.name }))
        : [];
    // ...
}
```

**Thread view: pass them in.**

```html
<c-email-reply-composer
    default-to={defaultTo}
    default-cc={defaultCc}
    default-subject={defaultSubject}
    default-body={defaultBody}
    initial-attachments={forwardAttachments}
    oncancel={handleComposerCancel}
    onsent={handleComposerSent}
></c-email-reply-composer>
```

Why it's done this way:

- **Pre-fill in the composer, not append on send.** The user sees the quoted original and the attachments before sending, and can remove an attachment or trim the quote.
- **The getters already return empty for replies**, so one code path serves both modes with no branching in the composer.
- **Seeded once, in `connectedCallback`.** After that the body belongs to the user; it is not overwritten if the parent re-renders.

## 7. Testing

**Automated tests:** none recorded for this change.

**Live check:** on 2026-07-17 the requester was asked to refresh, forward an email with an attachment, and confirm the quoted original and the attachment arrived. The ticket was closed on 2026-07-27; the confirmation itself is not recorded in the ticket.

**Not covered:** no Jest test asserts that a forward carries the body and attachments.

## 8. Step by step

1. **Reproduce:** open a thread with an attachment in the shared inbox, press **Forward**, and note the composer body is empty and no attachment is listed.
2. **Check the hand-off:** in `emailThreadView.html`, see which attributes are passed to `c-email-reply-composer`. Compare with the getters in `emailThreadView.js`.
3. **Add `@api defaultBody` and `@api initialAttachments`** to `emailReplyComposer.js` and seed them in `connectedCallback`.
4. **Bind `default-body` and `initial-attachments`** in `emailThreadView.html`.
5. **Deploy both bundles** together.
6. **Open the screen and test:** forward an email with an attachment to yourself. The sent message must contain the forwarded-message header, the original body and the attachment. Then reply to a thread and confirm the reply still opens empty.

## 9. Rollback

Redeploy the previous versions of both bundles. It is safe (front-end only, no data) but brings the empty forwards back.

## 10. Known limits

- **Forwarded attachments arrived without sizes.** A later change to the composer fetches their sizes so the attachment size guard still applies; the original fix did not.
- **Attachments whose Salesforce copy has since been moved to Google Drive** need separate handling; the current composer carries a Drive file reference for those. That was added after this ticket.
- **Forwarding from Gmail directly** is outside this screen and was not changed.

## 11. Reuse for similar issues

**How to recognise one:** data is clearly computed somewhere (a getter, a wire, a variable) but never shows up on screen or in the sent result.

- **Trace the hand-off between components.** A parent can compute a value correctly and simply never pass it. Compare the child's `@api` properties with the attributes in the parent template.
- **Don't trust the ticket's guess at the layer.** Forward problems sound like email-server configuration; this one was two missing template attributes.
- **Seed editable content once** in `connectedCallback`, so the user's edits are not overwritten.
- **On an LWC change, open the screen.** Apex validation and CI do not render the component.
