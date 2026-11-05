# Representation Agreement envelope subject stopped filling in

| | |
|---|---|
| **Ticket** | DDT-969 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-09 |
| **Priority** | Medium |
| **Reported by** | Case Manager, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Dochly e-signature |
| **Area** | Intake: sending the Representation Agreement for signature |
| **Related** | DDT-1093 (later fixes to the same template and its signing email) |
| **Code** | None. The fix was made in Dochly's envelope settings for the template. |

---

## 1. Summary

When a user created a Dochly e-signature envelope from the "Representation Agreement + Affidavit (ESIGN)" template, the email subject was blank, so it had to be typed by hand on every send. A default subject with a merge field for the client's name was added to the template, and the reporter confirmed it the same day.

## 2. What was reported

The subject used to fill in automatically and no longer did. This template is used for every representation agreement, a high-volume intake task, so the manual step cost time on every send.

## 3. Scope

**In scope**

| Component | System | Change |
|---|---|---|
| "Representation Agreement + Affidavit (ESIGN)" | Dochly e-signature template | Default subject added |

**Out of scope**

- **Other templates.** The ticket asked whether other Dochly templates had lost their subject the same way. That check is not recorded.

## 4. How it works (background)

```
User creates an envelope from a Dochly e-signature template
  -> Dochly fills the email subject and body from the template's defaults
  -> merge fields in them are resolved against the source record at send time
  -> the signer receives the email
```

The Dochly settings screen describes it as: edit the subject and body that signers will receive; merge fields are resolved against the source record at send time.

## 5. Root cause

Not recorded in the ticket. The ticket asked whether it was a template configuration change, an integration change or a Dochly-side change; which one is not recorded. What is recorded is that the subject had to be added back, so the template had no default subject at the time.

## 6. Solution

The wording was agreed on the ticket before it was added. The reporter recalled the old subject as "[Signature Required] Legal Representation for" followed by the client's full name. The subject added to the template, as shown in the confirmation screenshot:

```
[Signature Required] Legal Representation for {{Account.Name}}
```

`{{Account.Name}}` is a Dochly merge field. It is resolved against the source record when the envelope is sent, so each envelope carries that client's name.

## 7. Testing

The support team added the subject and asked the reporter to check it. The reporter confirmed it worked on the same day, 2026-09-09. No other test is recorded.

## 8. Step by step

1. Open the "Representation Agreement + Affidavit (ESIGN)" template's envelope settings in Dochly.
2. In **Subject**, enter `[Signature Required] Legal Representation for {{Account.Name}}`.
3. Save.
4. Create a test envelope from a record and check the subject shows the client's name.

## 9. Rollback

Clear the subject in the template's settings. That brings back the manual step, so there is no reason to.

## 10. Known limits

- **The cause of the original loss is unknown**, so it could happen again. If it does, check whether other templates lost their subject at the same time; that points to a package or bulk change rather than one template edit.

## 11. Reuse for similar issues

**How to recognise one:** a Dochly envelope opens with a blank or wrong subject or body, and users say it "used to fill in".

- **Agree the wording with the requester before you set it.** It took one question on the ticket and avoided a second round.
- **Use merge fields for anything record-specific**, such as the client's name, never typed values.
- **Keep a written copy of each template's subject and body.** Here the old subject had to be recalled from memory.
