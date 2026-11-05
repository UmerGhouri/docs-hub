# Letter of Representation had no letterhead, mixed fonts and an old email

| | |
|---|---|
| **Ticket** | DDT-1308 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-12 |
| **Priority** | Medium |
| **Reported by** | Firm operations (PI team), 2026-10-12 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation (template editor) |
| **Area** | PI: Letters of Representation to insurance carriers and third parties |
| **Related** | None recorded |
| **Code** | None. A Dochly template change, so nothing is in git |

---

## 1. Summary

The Letter of Representation (LOR) generated from Dochly printed on a blank page with no firm letterhead, used inconsistent font styles and sizes, and gave carriers an outdated contact email address. The letterhead was added, the fonts were standardised, and the email was changed to the PI team's current shared inbox.

## 2. What was reported

A generated LOR, addressed to a health insurer, showed three problems:

1. **No letterhead.** The letter had no firm branding or header.
2. **Fonts did not match** the firm's other outgoing letters.
3. **Outdated email.** The sentence asking the recipient to reply "via email to ... or fax to ..." named an old firm address instead of the PI team's current inbox.

Why it matters: LORs go to insurance carriers and third parties on the client's behalf. They need to look professional, and replies need to reach the inbox the team actually reads.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| The Dochly LOR template | Dochly template | Letterhead added at the top |
| Same | Dochly template | Font style and size standardised across the document |
| Same | Dochly template | Contact email in the body replaced with the PI team's current shared inbox |

The ticket refers to "the Dochly LOR template". Production holds several LOR variants (for example AF, MedPay, UM and UIM letters). Which of them were changed is not recorded in the ticket.

**Out of scope:** the fax number in the same sentence was not reported as wrong and is not recorded as changed.

## 4. How it works (background)

```
Staff generate the LOR from the PI case
  -> Dochly renders the template body, including any letterhead and fonts set in the template
  -> merge fields fill in the client, carrier and file details
  -> the letter is sent to the carrier
```

The letterhead, the fonts and any contact details typed into the body all live in the template, so every letter generated from it inherits them.

## 5. Root cause

Not recorded in the ticket. All three problems are properties of the template itself (missing letterhead, inconsistent formatting, an old address typed into the body), not of the case data.

## 6. Solution

In Dochly's template editor:

- the firm letterhead was added;
- the fonts were fixed to match the firm's standard;
- the email address in the reply sentence was updated to the PI team's current shared inbox.

The exact font, size and letterhead source used are not recorded in the ticket.

## 7. Testing

The acceptance criteria asked for a regenerated LOR on a test case showing all three fixes. Whether that was done is not recorded in the ticket; the closing comment lists the three changes as made.

**Not covered:** the other LOR variants, if they were not part of the change.

## 8. Step by step

1. **Generate an LOR** and note the missing letterhead, the fonts and the email address.
2. **Open the LOR template** in Dochly's template editor.
3. **Add the firm letterhead** at the top of the page.
4. **Select the whole body** and set one font style and size to match the firm's other letters.
5. **Replace the old email address** in the reply sentence with the current inbox.
6. **Save**, generate again and check all three.
7. **Check the other LOR variants** for the same letterhead, font and email.

## 9. Rollback

Not applicable: reverting would restore the old email and the unbranded page.

## 10. Known limits

- **Contact details typed into a template go stale.** The next time the inbox or phone number changes, every template that types it in has to be edited by hand. DDT-1380 is an example of that cost for the office address.
- **Variants drift apart.** A fix applied to one LOR variant does not reach the others.

## 11. Reuse for similar issues

**How to recognise one:** a generated letter looks unlike the firm's other letters, or names a contact detail the firm no longer uses.

- **Check every variant of the template**, not just the one in the screenshot.
- **Search all templates for an old email address or phone number** when one is retired.
- **Paste text into a template as plain text** where possible, so it picks up the template's font instead of bringing its own.
