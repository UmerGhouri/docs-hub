# Shared inbox replies missing prior thread history

| | |
|---|---|
| **Ticket** | DDT-605 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-04 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC), shared email inbox |
| **Area** | Shared inbox: the Reply and Reply All composer |
| **Related** | DDT-712 (BCC in the inbox composers), DDT-663 (attachment size warnings) |
| **Code** | PR #107 in the ddt-salesforce repo |

---

## 1. Summary

Replies sent from the Salesforce shared inbox went out as standalone emails, with no copy of the message being answered. Forward already quoted the original; Reply and Reply All never had. The composer's starting text now includes an "On `date`, `sender` wrote:" line and the original message in a quote block, the same as a standard email client.

## 2. What was reported

- **2026-07-31:** Reply and Reply All from the shared inbox sent only the new text. Recipients (clients, opposing counsel, courts) could not see what was being answered, and anyone forwarding the thread later lost the context.
- It affected every user replying from the shared inbox. The ticket asked whether it was a setting, a bug or how the composer was built.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `emailThreadView` (`emailThreadView.js`) | LWC | The `defaultBody` getter now builds a quoted block for Reply and Reply All, not just Forward |
| `.prettierrc` | Repo config | Aligned to the repo's 4-space, single-quote, 120-column style so the formatting check passed |

**Out of scope**

- **Forward.** Its behaviour is unchanged.
- **Apex and email templates.** None were involved.
- **Quoting the whole thread.** Only the message being replied to is quoted, as most email clients do; earlier messages appear only if that message already quoted them.

## 4. How it works (background)

```
User opens a thread in the shared inbox and clicks Reply / Reply All / Forward
  -> emailThreadView picks the source message (sourceMessage)
  -> defaultBody builds the composer's starting HTML
  -> the composer opens with that HTML; the user types above it
  -> the email is sent with whatever the body contains
```

Before the fix, `defaultBody` began with:

```js
if (this.replyMode !== 'forward') return '';
```

so for Reply and Reply All the composer started empty, and nothing below the user's text was ever sent.

## 5. Root cause

The composer was built that way. It was neither a configuration setting nor a template: `defaultBody` returned an empty string for every mode except Forward. The composer already seeded its body from `defaultBody`, so only that getter had to change.

## 6. Solution

The early return was removed. Forward keeps its existing header block, and Reply and Reply All get an attribution line and a quote block:

```js
const fromName = source.fromName ? this.escapeHtml(source.fromName) : '';
const fromAddr = source.fromAddress
    ? `${fromName ? ' ' : ''}&lt;${this.escapeHtml(source.fromAddress)}&gt;`
    : '';
const attribution = `On ${this.escapeHtml(this.formatDate(source.messageDate))}, ${fromName}${fromAddr} wrote:`;
return (
    `<br><br>${attribution}<br>` +
    `<blockquote style="margin:0 0 0 0.8ex;border-left:1px solid #ccc;padding-left:1ex;">${originalBody}</blockquote>`
);
```

Why it's done this way:

- **The original body is reused as-is.** The HTML body is used when present; otherwise the plain-text body is escaped and its line breaks kept. This is the same rule Forward already used.
- **Sender name, address and date are escaped** before going into HTML, so an address like `name <address>` displays rather than being read as a tag.
- **Inline style on the `blockquote`.** Email clients drop most stylesheet rules, so the left border that marks the quote has to be inline.
- **Two leading line breaks** leave room for the user to type above the quote.
- **Formatting was a separate commit.** The file had never been run through the formatter, and the new formatting check failed on it. Reformatting in its own commit kept the logic change readable on its own.

## 7. Testing

- The resolution comment (2026-08-03) confirms Reply and Reply All now include the prior email quoted below the new message, and that the change was live.
- No automated test is recorded. The ticket's acceptance criteria asked for a check against the scenario in the original screen recording and confirmation with the requester; whether those were done is not recorded in the ticket.

## 8. Step by step

1. Reproduce: open a thread in the shared inbox, click Reply, and confirm the composer opens empty.
2. In `emailThreadView.js`, find `get defaultBody()` and remove the early return for non-forward modes.
3. Build the original body once (HTML body, or escaped plain text).
4. Keep the existing Forward branch.
5. For Reply and Reply All, return an attribution line and the original body inside an inline-styled `blockquote`, escaping every value that goes into the HTML.
6. Deploy the LWC. Then **open the screen**: click Reply and Reply All on a real thread and confirm the quote appears below the cursor. No CI check renders an LWC.
7. Send a test reply to an outside mailbox and confirm the quote arrives intact.

## 9. Rollback

Redeploy the previous `emailThreadView.js`. Replies go back to having no quoted history. Safe, but it brings the original problem back.

## 10. Known limits

- **Only the replied-to message is quoted.** If that message didn't quote its own history, older messages aren't included.
- **Long threads grow.** Every reply carries the previous message, so a long back-and-forth makes large emails. Users can delete the quote before sending.
- **The quote is editable.** Users can change or remove it, and nothing records that they did.

## 11. Reuse for similar issues

**How to recognise one:** a feature works for one composer action (Forward) and not for its siblings (Reply, Reply All).

- **Look for an early return or a mode check** in the code that seeds the composer. A one-line guard is often the whole cause.
- **Escape anything that goes into HTML**, including names and email addresses.
- **Use inline styles in email HTML.**
- **Keep reformatting out of the logic commit**, so a reviewer can see what actually changed.
