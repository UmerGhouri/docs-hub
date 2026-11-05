# Some inbound HTML emails render blank

| | |
|---|---|
| **Ticket** | DDT-644 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-07 |
| **Priority** | High |
| **Reported by** | Firm staff (PI team), 2026-08-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Visualforce print page, LWC), shared inbox |
| **Area** | Shared inbox: reading and printing inbound emails on a Case |
| **Related** | None recorded |
| **Code** | PR #117 in the `ddt-salesforce` repo |

---

## 1. Summary

Emails from a medical-records vendor showed up completely blank in the Salesforce shared inbox, although they read fine in Gmail. The emails carry a large inline logo encoded inside the HTML. That pushed the HTML past Salesforce's **128 KB limit on `EmailMessage.HtmlBody`**, so the stored HTML was cut off partway through the image and contained no visible text. The inbox showed the HTML whenever it was not empty.

The fix checks whether the HTML has any visible text. If it doesn't, the inbox and the print view show the plain-text body instead, which Salesforce stores separately and in full, along with a short notice.

## 2. What was reported

A PI team member reported that an email from MRO Corp ("Your Medical Records Are Available - Payment Due") rendered as an empty body on the Case. In Gmail the same email showed the amount due, contact details and a "Pay Now" button. Without them, the PI team could miss that records were ready and a payment was due unless they checked Gmail. The ticket expected other vendor emails with heavy HTML to be affected too.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `emailThreadView` | LWC | New `htmlHasVisibleContent()`; falls back to the text body for display and for reply/forward quoting; shows a notice on fallback |
| `EmailThreadPrintController` | Apex class | Same visible-content check and fallback for the print view |
| `EmailThreadPrintControllerTest` | Apex test | Two new tests (see section 7) |

**Out of scope**

- **Recovering the lost HTML.** Salesforce truncates `HtmlBody` when the email is saved. The cut-off part, including the "Pay Now" button, is not stored anywhere in Salesforce. The fix shows what *is* stored.
- **How emails come in** (Email-to-Case). Nothing changed in intake.

## 4. How it works (background)

```
Vendor email arrives
  -> Salesforce saves an EmailMessage on the Case
       HtmlBody  (capped at 128 KB, truncated beyond that)
       TextBody  (separate field, plain text)
  -> shared inbox (emailThreadView) and print view (EmailThreadPrintController)
       before: show HtmlBody whenever it is not blank, else TextBody
```

A logo embedded as base64 text inside the HTML adds a large block of text to it. If that block comes before the message text, the 128 KB cut lands inside the image, and everything after it, which is all the visible text, is lost.

## 5. Root cause

1. The vendor's HTML carries a large inline base64 logo near the top.
2. The full HTML is over 128 KB, so Salesforce stored only the first 128 KB of `HtmlBody`, ending mid-image.
3. What remained was markup plus a broken, unclosed image tag, with no visible text.
4. The inbox checked only whether `HtmlBody` was **non-blank**. It was, so the inbox handed that markup to `lightning-formatted-rich-text`, which rendered nothing.
5. `TextBody` held the full readable message all along. It was never shown, because the HTML "existed".

The ticket listed sanitisation, multipart parsing, inline images and CSS as possible causes. It was none of them: it was the storage limit on the field.

## 6. Solution

The same check is written once in JavaScript (inbox) and once in Apex (print view), so both views agree:

```apex
@TestVisible
private static Boolean htmlHasVisibleContent(String html) {
    if (String.isBlank(html))
        return false;
    String stripped = html
        .replaceAll('(?is)<head.*?</head>', ' ')
        .replaceAll('(?is)<style.*?</style>', ' ')
        .replaceAll('(?is)<script.*?</script>', ' ')
        .replaceAll('(?s)<!--.*?-->', ' ')
        .replaceAll('<[^>]*>', ' ')   // closed tags
        .replaceAll('<[^>]*$', ' ')   // trailing unterminated tag (truncation point)
        .replaceAll('&nbsp;', ' ')
        .replaceAll('&[a-zA-Z#0-9]+;', ' ')
        .replaceAll('\\s+', ' ')
        .trim();
    return stripped.length() > 0;
}
```

The body to show becomes: the HTML if it has visible text, otherwise the plain-text body, escaped and with line breaks kept.

Why it is done this way:

- **"Has visible text", not "is not blank".** Truncated HTML is not blank, which is exactly why the old check passed it through.
- **Head, style and script are removed first.** A page title alone is not visible body content.
- **The trailing unclosed tag is removed separately.** The cut point leaves an unterminated tag that a normal tag pattern would not match.
- **The fallback notice shows only when there was HTML that could not be shown and text exists.** It reads: *"The formatted version was too large to store, so the plain-text version is shown below. Any buttons or links (e.g. a payment portal) may only be visible in the original mailbox."* Plain-text-only emails don't get a spurious notice.
- **Reply and forward quoting use the same rule**, so a reply doesn't quote an empty body.
- **Normal HTML emails are unchanged.** Any HTML with visible text is shown as before.

## 7. Testing

**Apex tests** in `EmailThreadPrintControllerTest`:

| Test | What it proves |
|---|---|
| `fallsBackToTextWhenHtmlHasNoVisibleContent` | An email whose HTML ends inside an unclosed base64 image renders the text body (containing the amount due), and the output does not contain the truncated HTML |
| `htmlHasVisibleContentHeuristic` | Null and whitespace are not visible; a head/title with only an image is not visible; HTML cut mid-base64 is not visible; a paragraph with text and a link is visible |

**Not recorded in the ticket:** a live check against the reported email after deploy, a check against a second vendor's email (an acceptance criterion), or tests for the LWC side.

**Not covered:** the "Pay Now" link. It was in the truncated part and cannot be recovered in Salesforce.

## 8. Step by step

1. Open the blank email's `EmailMessage` and check the length of `HtmlBody`. A value at the 128 KB cap means it was truncated.
2. Check that `TextBody` has the readable content.
3. Look at the end of `HtmlBody`. If it stops inside a tag or a base64 string, this is the same problem.
4. In the component that displays the email, replace any "HTML is not blank" check with a "HTML has visible text" check, as in section 6.
5. Fall back to the escaped `TextBody`, and tell the user that formatted content may only be in the original mailbox.
6. Apply the same rule everywhere the body is shown or quoted: display, print, reply and forward.
7. Add a test with HTML cut off mid-image and confirm it renders the text body.

## 9. Rollback

Redeploy the previous versions of `emailThreadView` and `EmailThreadPrintController`. Safe, with no data involved, but truncated vendor emails go back to rendering blank.

## 10. Known limits

- **Links and buttons in the cut-off part are lost** in Salesforce. Users have to open the original mailbox for them.
- **An email truncated after some visible text** still shows the partial HTML, not the text body, because it does have visible content. The tail of that email is still missing.
- **The check is a pattern match, not an HTML parser.** Unusual markup could fool it either way.
- **Two copies of the same logic** (JavaScript and Apex) must be changed together.

## 11. Reuse for similar issues

**How to recognise one:** an email reads fine in the mail client but is blank, or cut short, in Salesforce, and it has heavy formatting or embedded images.

- **Check the field length first.** `EmailMessage.HtmlBody` and `TextBody` are capped. A length at the cap means truncation, not a rendering bug.
- **Prefer a field that survived** (here `TextBody`) over one that didn't.
- **"Not blank" is not "usable".** Test whether content can actually be shown.
- **Tell the user when you show a fallback**, so they know to look elsewhere for what's missing.
