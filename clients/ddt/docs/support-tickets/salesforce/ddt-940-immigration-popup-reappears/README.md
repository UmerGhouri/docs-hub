# Immigration consultation popup reappears after No

| | |
|---|---|
| **Ticket** | DDT-940 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-07 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-09-04 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Screen Flow, formula field, Lightning record page) |
| **Area** | Hired Court Case record page: the Immigration Consultation Recommended popup |
| **Related** | None |
| **Code** | PR #313 in the ddt-salesforce repo |

---

## 1. Summary

The "Immigration Consultation Recommended" popup on a hired Court Case kept coming back after a user answered **No** and submitted. Two separate defects caused it, and either one alone was enough: the flow saved nothing when the answer was No, and the formula that decides whether to show the popup treated No the same as "not answered".

Both were fixed, and a third problem the fix exposed (a page section that hid the client's immigration status when the answer was No) was fixed in the same change.

## 2. What was reported

The popup asks "Does the client need an immigration attorney consultation?" with a Yes/No toggle. After choosing No and submitting, reloading the page showed the popup again.

Production data showed the scale: across 41,233 Court Cases, the answer field `Immigration_Consultation_Recommended_Pic__c` was `Yes` on 18, blank on 41,215, and **`No` on zero**. No user's No had ever been recorded. At the time of the fix, 21 Court Cases were in the "show the popup" state.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `CC_Immigration_Consultation_Recommended` | Screen Flow | New `Set_Not_Needed` assignment; the No path now goes through it to the record update |
| `Opportunity.Immigration_Consultation_Check__c` | Formula field (the popup's gate) | Last clause narrowed to "blank" only |
| `Opportunity_Record_BB_Default_Hired` | Lightning record page | The Immigration Concerns section shows for Yes or No; the two follow-up fields show for Yes only |

**Out of scope**

- **Backfilling answers.** No existing record was edited. Records already gated prompt once more, then stop.
- **Changing the toggle control.** See section 10.

## 4. How it works (background)

```
Hired Court Case page loads
  -> managed PopUp component checks Immigration_Consultation_Check__c
  -> if true, launches the CC_Immigration_Consultation_Recommended screen flow
  -> user answers Yes or No and submits
  -> the flow should write the answer, which turns the gate formula false
```

The popup component has no memory of its own. Its "Pester" mode only means "show whenever the gate is true". The popup stops only because the flow writes something that makes the gate false. Another popup on the same page ("Change to CM Stage") uses the same pattern and works.

## 5. Root cause

**Defect 1: the flow never wrote anything on No.** The decision sent **Yes** to `Set_Verified` and then `Update_the_Court_Case`. Its default connector, labelled No, went straight to `Go_to_Court_Case`: no assignment and no record update. The user saw Submit succeed, but nothing was saved, so the gate was still true on reload.

**Defect 2: the gate formula treated No as unanswered.** `Immigration_Consultation_Check__c` ended with:

```
OR(ISPICKVAL(Immigration_Consultation_Recommended_Pic__c,'No'),
   ISPICKVAL(Immigration_Consultation_Recommended_Pic__c,""))
```

So even with defect 1 fixed, a saved No would still show the popup.

**Evidence.** Group the answer field:

```sql
SELECT Immigration_Consultation_Recommended_Pic__c, COUNT(Id)
FROM Opportunity
GROUP BY Immigration_Consultation_Recommended_Pic__c
```

Yes 18, blank 41,215, No 0. A restricted Yes/No picklist with zero rows on one of its two values is near-proof that nothing ever writes it, and stronger evidence than reading the flow.

**Defect 3, found in review.** The "Immigration Concerns" page section was shown only when the answer was Yes. Once No could be saved, answering No would hide the whole section, including the client's immigration status field, which is also what drives the gate formula.

## 6. Solution

1. **Flow.** Added `Set_Not_Needed`, a mirror of `Set_Verified` that assigns `No`, and pointed the default connector at it. Both branches now meet at the existing `Update_the_Court_Case`, so No is saved by the same path already proven for Yes. No new write path was invented.
2. **Gate formula.** The last clause now tests blank only. Blank is the real "not yet answered" state: the picklist is restricted to Yes and No and has no default.
3. **Record page.** The section is now shown when the answer is Yes **or** No. The two follow-up fields are gated individually to Yes. Result:

```
blank -> section hidden (unchanged)
No    -> immigration status + the answer
Yes   -> all four fields (unchanged)
```

This was the conservative option. Moving the answer to an always-visible section was rejected because it would have shown immigration status on every hired Court Case, blank and irrelevant on nearly all of them.

The record page was retrieved from production first and edited on top of prod's copy, because the repo's copy was behind prod in two unrelated places. The committed file is prod's file plus 20 added lines.

## 7. Testing

**Formula, without touching data.** `Formula.recalculateFormulas` recomputes formula fields on an in-memory record, so the gate was tested with no DML, no triggers and no flows:

```apex
Opportunity o = new Opportunity(Immigration_Status__c = 'U-Visa', Case_Type__c = 'DUI',
    Discovery_Review_Meeting_with_Client__c = 'Discovery Not Discussed w/ Client',
    Immigration_Consultation_Recommended_Pic__c = answer);
Formula.recalculateFormulas(new List<Opportunity>{ o });
System.debug(o.Immigration_Consultation_Check__c);
```

| Answer | Production before the fix | Sandbox after the fix |
|---|---|---|
| blank | shows | shows |
| No | **shows** (the bug) | hidden |
| Yes | hidden | hidden |

**Flow.** Deployed to a sandbox and confirmed Active with matching Yes and No paths.

**Validation.** Check-only against production with two Apex test classes that insert and update Opportunities, so the changed formula is recalculated on save: 60 tests, 0 failures. Production confirmed fixed on 2026-09-07.

**Not covered:** an automated test of the popup itself. The post-deploy checks were manual.

## 8. Step by step

1. **Find the popup's gate.** On the record page, open the popup component and note the gate field in its visibility rule and the flow it launches.
2. **Group the answer field** with the query in section 5. A value with zero rows means nothing writes it.
3. **Open the flow** and follow every branch of the decision to see which ones reach a record update.
4. **Read the gate formula** and check which answers keep it true.
5. **Fix the flow** so every answer reaches the record update.
6. **Fix the formula** so only "blank" counts as unanswered. Test it with `Formula.recalculateFormulas` as in section 7.
7. **Check the record page** for anything gated on the old values, and adjust it. Retrieve the page from production first and edit prod's copy.
8. **Deploy** the flow, field and page together. The flow is saved as Active, so the deploy creates and activates a new version; confirm it is the active one.
9. **Verify:** submit No on a gated record and reload (popup gone, status and answer still visible); submit Yes on another (popup gone, all four fields visible); a blank record still hides the section.

## 9. Rollback

Redeploy the previous flow version, formula and page. That brings the bug back. Any No answers saved since the fix stay on the records; reverting the formula would make the popup show on them again.

## 10. Known limits

- **A click-through now saves No.** The toggle has no starting value, so it renders as "off", and a Submit without touching it falls to the No path. Before the fix that wrote nothing; now it is a permanent answer. Two options were offered to firm leadership (a radio group with no preselected value, or a confirmation step on the No path). Not changed in this ticket.
- **Records answered before the fix were not backfilled.** They prompt once more, then stop.

## 11. Reuse for similar issues

**How to recognise one:** "the popup keeps coming back" or "my answer doesn't stick".

- **Group by the answer field.** A value that never appears means nothing writes it.
- **Follow every branch of the flow**, especially the default connector. A path that navigates away without an update looks like success to the user.
- **Check the gate formula separately.** Two defects can each cause the same symptom; fixing one is not proof.
- **Use `Formula.recalculateFormulas`** to test a formula in production safely, with no DML.
- **When a fix makes a new value reachable, check what the page does with it.** A visibility rule that was never triggered in practice goes live the moment the writer is fixed.
