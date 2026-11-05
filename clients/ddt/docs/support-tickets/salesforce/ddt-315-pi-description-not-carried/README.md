# PI lead description not carried to the Matter on signing

| | |
|---|---|
| **Ticket** | DDT-315 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-18 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-07-13 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | PI intake: converting the Lead into a Matter when the retainer is signed |
| **Related** | DDT-1070 (PI contract later moved from DocuSign to Dochly; same conversion code) |
| **Code** | PR #57 (first attempt) and PR #191 (the fix) in the ddt-salesforce repo |

---

## 1. Summary

When a PI client signed the retainer and a Matter was created, the intake narrative in the Lead's **Description** did not reach the Matter, so intake staff copied it by hand. The first fix (PR #57) went into the wrong class: the retainer-signed path never runs it. The real cause was that the conversion screen never read the Lead's Description and filled the field with a generated sentence instead.

PR #191 fixed it on the server: the Matter's description is now that sentence, a blank line, then the Lead's narrative.

## 2. What was reported

- **2026-07-13:** intake staff reported that, on signing, the Lead's Description was not on the new PI Matter, and that they were copying it over by hand. PR #57 was deployed and the ticket closed the same day.
- **2026-07-23:** reopened. A newly signed PI client's Matter again had no intake description.
- **2026-07-27:** reported again for another signed Matter, with a request to prioritise it, because case managers and attorneys rely on those intake notes.

According to PR #191, it was still happening on Matters created on 2026-08-17, and roughly half of signed PI Matters had lost the narrative.

## 3. Scope

**In scope**

| Component | Type | PR | Change |
|---|---|---|---|
| `ClientConvertController` | Apex class | #57 | Fallback to the Lead's Description when the form's is blank (did not fix the report; see 5) |
| `HiredMatterConvertController` | Apex class | #191 | `buildMatter` sets `Description__c` through a new `composeMatterDescription`; `loadLeadForConversion` selects `Description` |
| `HiredMatterConvertControllerTest` | Apex test | #191 | Four new tests (see 7) |

**Out of scope**

- **`convertToHiredMatter` LWC.** Deliberately not changed (see 6).
- **Matters already created without the narrative.** No backfill is recorded.

## 4. How it works (background)

```
Intake fills the Lead, including Description (the intake narrative)
  -> convertToHiredMatter (LWC) builds a form; its description is a generated incident sentence
  -> that same form description is merged onto the retainer the client signs
  -> client signs
  -> HiredMatterConvertController.buildMatter creates the Matter__c
     (sets Lead Status = 'Converted' itself; does not call Database.convertLead)
```

There are two different conversion classes:

| Class | Used by |
|---|---|
| `ClientConvertController` | The standard convert modal (a real `Database.convertLead`) |
| `HiredMatterConvertController` | The retainer-signed path and the panel's in-person Convert button |

Because the retainer path never calls `Database.convertLead`, every Lead converted that way still shows `IsConverted = false`. That is how PR #191 confirmed which path the reported clients had taken.

## 5. Root cause

1. In `convertToHiredMatter.js`, the `getLeadData` wire reads about fifteen fields off the Lead and **never reads `description`**. It then fills the form's description with the generated incident sentence.
2. `buildMatter` copied the form's description straight into `Matter__c.Description__c`.
3. So the server fetched the Lead's Description and the screen threw it away.

**Why the first fix did nothing.** PR #57 added a blank-description fallback to `ClientConvertController`. The reported clients never went through that class; they went through `HiredMatterConvertController`. And on that path, the form's description is never blank: `validateRequiredMatterFields()` requires it. So the same blank-check copied to the right class would also never have fired.

**The lesson.** Check which code path the affected records actually took (here, `IsConverted = false` gave it away) before fixing. A fallback that can never trigger passes every test and fixes nothing.

## 6. Solution

### 6.1 Compose the description on the server (PR #191)

```apex
private static final Integer MAX_DESCRIPTION_LENGTH = Matter__c.Description__c.getDescribe()
  .getLength();

// in buildMatter:
m.Description__c = composeMatterDescription(form.description, ld.Description);

private static String composeMatterDescription(String formDesc, String leadDesc) {
  String sentence = String.isBlank(formDesc) ? '' : formDesc.trim();
  String intake = String.isBlank(leadDesc) ? '' : leadDesc.trim();
  if (String.isBlank(intake)) {
    return sentence;
  }
  // Staff already pasted the narrative into the form - do not duplicate it.
  if (sentence.contains(intake)) {
    return sentence;
  }
  if (String.isBlank(sentence)) {
    return intake;
  }
  String combined = sentence + '\n\n' + intake;
  return combined.length() > MAX_DESCRIPTION_LENGTH
    ? combined.substring(0, MAX_DESCRIPTION_LENGTH)
    : combined;
}
```

`Description` was also added to the Lead query in `loadLeadForConversion`.

Why it's done this way:

- **Server-side, in `buildMatter`**, so it covers both the signed-retainer path and the in-person Convert button.
- **No change to the LWC, on purpose.** The form's description is printed on the client-facing retainer. Putting the intake narrative there would print internal notes (insurance details, intake staff names) on a legal document. The contract keeps the clean sentence; the Matter gets the sentence and the narrative.
- **Not a blank-check.** As in section 5, the form value is never blank on this path.
- **No duplicates.** If staff already pasted the narrative into the form, it is not added twice.
- **The length cap is read from the field**, not hardcoded. The Lead's Description alone can be very long, and overflowing the Matter field would throw and roll back the whole hire.
- **The query change is required.** Without `Description` in the SOQL, reading `ld.Description` throws inside the conversion and rolls back the entire hire, which is worse than the bug.

## 7. Testing

| Test | What it proves |
|---|---|
| `convertPiLead_CarriesLeadNarrativeOntoMatter` | Uses a non-blank form description different from the Lead's, and asserts the exact result: sentence, blank line, narrative. **Fails under the PR #57 pattern.** |
| `convertPiLead_NarrativeAlreadyInForm_IsNotDuplicated` | The narrative appears once when staff already pasted it |
| `convertPiLead_BlankLeadDescription_KeepsSentenceOnly` | A blank Lead description leaves the sentence alone |
| `convertPiLead_OverlongCombinedDescription_IsTruncatedNotThrown` | An over-long result is cut at the field length, keeps the sentence at the start, and does not throw |

**Sandbox:** 23 of 23 tests passing. End to end in the sandbox: contract sent, signed, conversion run; the Matter's description came out byte-for-byte as sentence, blank line, Lead description, with the narrative present exactly once.

**Live:** the support team confirmed in the ticket on 2026-08-18 that the Lead's description now carries over when a PI client signs.

**Not covered:** no backfill of Matters created before the fix.

## 8. Step by step

1. **Take one affected Matter and its Lead.** Check `Lead.IsConverted`. `false` with Status "Converted" means the retainer path (`HiredMatterConvertController`), not the standard convert.
2. **Trace the field from the Lead to the Matter** on that path: the LWC wire, the form, `buildMatter`. Find where the value is dropped.
3. **Check what is shown to the client.** If the form value is printed on the contract, fix it on the server, not in the form.
4. **Add the Lead field to the server-side query** before reading it.
5. **Compose, dedupe and cap the value** as in 6.1.
6. **Write a test that fails under the earlier non-fix**: a non-blank form value that differs from the Lead's.
7. **Run it end to end in a sandbox** (send, sign, convert) and compare the Matter's description exactly.
8. **Validate against production, get approval, deploy**, and confirm on the next real signing.

## 9. Rollback

Redeploy `HiredMatterConvertController` and its test from the commit before PR #191. That brings the bug back. The PR #57 fallback in `ClientConvertController` is harmless and can stay.

## 10. Known limits

- **The truncation keeps the start.** If the combined text is over the field length, the end of the narrative is cut.
- **The duplicate check is exact text.** A narrative pasted into the form with small edits will appear twice.
- **Only Description is carried.** The ticket also asked for a full field-mapping audit of the Lead to Matter conversion; no such audit is recorded.
- **Later changes to the conversion code must keep `composeMatterDescription`.** A later reconcile of this class against production explicitly preserved it; any future rewrite should too.

## 11. Reuse for similar issues

**How to recognise one:** a Lead field "doesn't carry over" to the record created from it, and a fix has already been shipped once.

- **Find the path the affected records really took.** In this org a "converted" PI Lead may never have gone through `Database.convertLead`.
- **Check whether a fallback can ever fire** before relying on it. A required field is never blank.
- **Don't put internal notes in values that are merged onto client documents.** Fix the target record on the server instead.
- **Make the regression test fail under the previous attempt.** That proves it tests the real path.
- **Read field lengths from describe**, not literals, when building long text.
