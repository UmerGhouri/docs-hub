# Lien Reduction letter left prompt answers blank, with mixed fonts and stray spacing

| | |
|---|---|
| **Ticket** | DDT-996 |
| **Project** | Dochly |
| **Status** | Live since 2026-09-14 |
| **Priority** | Medium |
| **Reported by** | Reductions team, 2026-09-10 |
| **Resolved by** | Support team |
| **Systems** | Dochly document generation |
| **Area** | PI reductions: lien reduction and lien release letters |
| **Related** | None recorded |
| **Code** | None. The fixes were made in the Dochly template editor. |

---

## 1. Summary

A live test of the Dochly Lien Reduction and Release of Lien letter templates found formatting faults in both, and five answers typed into the Lien Reduction letter's prompt screen that never reached the letter. Both templates were fixed in Dochly's template editor; the fix was reported on 2026-09-11 and the ticket closed on 2026-09-14.

## 2. What was reported

The reductions team listed these after generating both letters for one test client:

**Formatting, all letters**

- Font style and size differed between merge fields and body text. Date of Service, for example, printed in a different font and size from the rest of the body.
- A large blank gap between the header image and the date line, where a single line was expected.
- Double or triple spaces before some merged figures.

**Lien Reduction letter: prompt answers that did not merge**

| Prompt answer | What the letter showed |
|---|---|
| "Sent Via" method and email address | The `Via:` line was blank |
| Settlement Offer Amount | Missing from the text |
| Total Billed Amount | Blank |
| Distributable Amount | Blank |
| Response Deadline Date | The deadline line was empty |

**Lien Reduction letter: styling request.** The proposed reduced payoff amount was calculating correctly; the team asked for it to print in **bold and underlined**.

**Release of Lien letter.** All data merged correctly; it only needed its font style and size matched to the rest of the template.

## 3. Scope

**In scope**

| Component | System | Change |
|---|---|---|
| Lien Reduction letter | Dochly template | Fonts, spacing, five prompt answers merged, payoff amount bold and underlined |
| Release of Lien letter | Dochly template | Font style and size |

**Out of scope**

- Any other Dochly letter template.

## 4. How it works (background)

```
User generates a lien letter from the record
  -> Dochly shows a prompt screen for values the record doesn't hold
     (sent-via method, offer amount, billed and distributable amounts, response deadline)
  -> the template merges record fields and prompt answers into the letter
  -> the PDF is generated
```

A value typed into the prompt screen reaches the letter only if the template body contains a merge field that references that prompt.

## 5. Root cause

Not recorded in the ticket. The symptom, values entered on the prompt screen that never appear, fits a template body that did not reference those prompts, but which merge fields were missing or misnamed is not recorded. The formatting faults are also not explained.

## 6. Solution

The completion note on the ticket reads: "it is fixed". The specific merge fields, fonts and spacing used are not recorded.

## 7. Testing

The support team asked the reporter to check the result on 2026-09-11. A confirmation from the reductions team is not recorded on the ticket.

## 8. Step by step

1. Generate both letters on a test record, filling in every prompt.
2. In the Lien Reduction template, make sure each prompt has a merge field in the body: sent-via method and address, settlement offer amount, total billed amount, distributable amount, response deadline.
3. Format the reduced payoff amount merge field bold and underlined.
4. In both templates, set one font style and size for the body and every merge field.
5. Remove the extra blank lines between the header image and the date, and the extra spaces before merged figures.
6. Generate both letters again and compare them line by line with what was entered.

## 9. Rollback

Restore the previous template version in Dochly, if one is kept. That brings back the blank fields, so there is no reason to.

## 10. Known limits

- **Formatting is per merge field.** A merge field added later can bring its own font and spacing with it, so the same fault can come back one field at a time.

## 11. Reuse for similar issues

**How to recognise one:** a user fills in a Dochly prompt and the value is missing from the document, or merged values look different from the text around them.

- **Every prompt needs a matching merge field in the body.** A prompt the body never references is asked for and then dropped, with no error.
- **Test with every prompt filled in**, then read the output against the inputs.
- **Pasted text brings its own font.** When a merge field prints in a different font, clear the formatting on that span and apply the body style.
- **Ask for the formatting rules up front**, such as which amounts must be bold or underlined, so they're built in the first time.
