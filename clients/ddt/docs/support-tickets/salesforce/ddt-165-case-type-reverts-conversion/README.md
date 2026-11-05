# Case Type reverting after lead conversion

| | |
|---|---|
| **Ticket** | DDT-165 |
| **Project** | Support tickets |
| **Status** | Done 2026-07-22 |
| **Priority** | High |
| **Reported by** | Intake team, 2026-06-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Court Case / Opportunity fields, field history) |
| **Area** | Case type and primary citation on Court Cases after Lead conversion |
| **Related** | None recorded |
| **Code** | None. Answered with an explanation |

---

## 1. Summary

Intake reported that a Lead set as a traffic felony became "Aggravated Speeding" after conversion. Field history showed the conversion had not changed the case type. Case Type was still correct; "Aggravated Speeding" was the **Primary Citation**, a separate field. A follow-up report showed the other half of the story: Case Type is set **automatically from the citations added to the case**, so entering the citations changes it.

## 2. What was reported

- **2026-06-13:** a Lead set as Traffic Felony showed Aggravated Speeding on the converted Court Case.
- **2026-06-17:** on another case, staff marked it a traffic misdemeanor, entered a leaving-the-scene citation and then a petty citation, and the case type switched to petty.

How many cases were affected is not recorded in the ticket.

## 3. Scope

**In scope:** investigation and explanation. The first example case was corrected by the support team on 2026-06-15.

**Out of scope**

- **Changing how Case Type is derived from citations.** Not changed under this ticket.
- **Lead conversion mapping.** The investigation showed conversion was not involved.

## 4. How it works (background)

```
Lead converted
  -> Court Case (Opportunity) created with its Case Type
  -> staff add citations to the case
  -> Case Type is updated automatically from the citations
  -> Primary Citation (a separate field) holds the specific charge
```

Two fields are easy to confuse here:

| Field | What it holds |
|---|---|
| `Case_Type__c` | The kind of case, such as Felony |
| `Primary_Citation__c` | The specific charge, such as an aggravated speeding class |

Which automation updates Case Type from the citations, and how it chooses between several citations, is not recorded in the ticket.

## 5. Root cause

**First report.** Field history on the converted Court Case showed the conversion did not change the case type. Case Type was set to Felony, which was correct. "Aggravated Speeding" was the Primary Citation field, which still held a misdemeanor-class aggravated speeding charge.

**Second report.** Case Type is not a value staff set once. It updates automatically based on the citations added to the case, so adding a lower-level citation after a higher one can move it.

**The ticket's first guess was wrong.** It suspected a flow or trigger on Lead conversion overwriting the case type. Conversion was not involved: one report was the wrong field being read, and the other was the citation-driven update working as built.

## 6. Solution

- **2026-06-15:** explained that Case Type was correct and the value seen was the Primary Citation; if the charge is really felony-level, update Primary Citation directly. The support team also corrected the example case.
- **2026-06-18:** explained that Case Type updates automatically from the citations added to the case, which is why it changed after citations were entered.

No configuration or code was changed.

## 7. Testing

Field history on the first example Court Case was traced to confirm conversion did not change Case Type. No other check was recorded.

**Not covered:** the ticket's suggested regression test of converting a Lead with each case type was not recorded as run.

## 8. Step by step

1. **Open the Court Case's field history** for `Case_Type__c` and `Primary_Citation__c`. See what changed, when, and by whom (or by which automation user).
2. **Confirm which field the user is looking at.** If the "wrong" value is in Primary Citation, Case Type may be fine.
3. **If Case Type changed after conversion**, check whether it changed when citations were added. If so, it is the citation-driven update, not conversion.
4. **Explain to the user** which field holds what, and that adding citations updates Case Type.
5. **Correct the record** if the citation itself is wrong: fix the citation or Primary Citation, not Case Type directly, since the automation will set it again.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **A hand-set Case Type does not stick** once citations are added, per the 2026-06-18 explanation. Staff who classify a case by hand will keep seeing it change.
- **The rule that picks Case Type from several citations is not documented in the ticket**, so it is hard for staff to predict.

## 11. Reuse for similar issues

**How to recognise one:** "the value changed after I did X" on a field that automation also writes.

- **Read field history before reading automation.** It tells you when the change happened and who made it.
- **Check the user is looking at the field they think they are.** Two fields with related values are easy to mix up on a busy page.
- **If a field is derived, tell staff where it comes from**, and fix the source data rather than the derived value.
