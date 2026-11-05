# Medical records request modes 2 and 3

| | |
|---|---|
| **Ticket** | DDT-1002, DDT-1024 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-24 (DDT-1002 live since 2026-09-15, DDT-1024 since 2026-09-24) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-13 (DDT-1002) and 2026-09-16 (DDT-1024) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, validation rule, LWC, report, anonymous Apex) |
| **Area** | PI cases: medical records and bills requests on the Damages table (Mode 1 Treating, Mode 2 Requesting, Mode 3 Obtained) |
| **Related** | DDT-363 (the existing Records Retrieval flow), DDT-271 (Mode 3 dates read-only on purpose), DDT-724 (date shown a day early in this component), DDT-1100 (follow-up log, split out) |
| **Code** | PR #366 (DDT-1002), PR #456 and PR #464 (DDT-1024) in the `ddt-salesforce` repo; PR #465 added a related CI check |

---

## 1. Summary

Each medical damage on a PI matter moves through three modes: Mode 1 (treating, no records status), Mode 2 (requesting records and bills), Mode 3 (records or bills received). Two tickets improved that workflow:

- **DDT-1002:** when a matter enters the Records Retrieval stage, its Mode 1 medical damages now move to Mode 2 automatically instead of being started one by one.
- **DDT-1024:** Mode 3 now enforces the received date, amounts and treatment dates; treatment dates can be corrected in Mode 3 without going back to Mode 1; and a report lists overdue records requests. Along the way we found a validation rule that had been "Active" for months but **could never fire**, because it tested a picklist value that no longer existed.

## 2. What was reported

### DDT-1002 (2026-09-13)

When a matter's stage changed to Records Retrieval, its medical damages stayed in Mode 1, so someone had to open the Damages table and start each row by hand. On 2026-09-12, 48 Mode 1 medical damages across 23 matters were stranded this way on matters already at Records Retrieval. The request had felt like it had been built already; it had not. The neighbouring piece (DDT-363: completing the "Confirm MMI" task and setting MMI and treatment-complete fields on the same stage change) had shipped, but the damages step never had.

### DDT-1024 (2026-09-16)

A process request with four asks:

1. A follow-up log in Mode 2 to track call and email attempts.
2. First and Last Treatment Date editable in Mode 3, without reverting to Mode 1.
3. Service dates auto-populated from the treatment dates in Mode 3.
4. Required fields before leaving Mode 3: Record Status, Record Received Date, First and Last Treatment Date, Billed Amount and Outstanding Balance.

Measured in production on 2026-09-22 across 349 Mode 3 medical damages:

| Field | Filled | Blank |
|---|---|---|
| `Records_Status__c` | 349 | 0 |
| `Billed_Amount__c` | 313 | 36 |
| `Outstanding_Balance__c` | 309 | 40 |
| `First_Treatment_Date__c` | 287 | 62 |
| `Last_Treatment_Date__c` | 250 | 99 |
| `Records_Received_Date__c` | **187** | **162** |

## 3. Scope

**In scope**

| Component | Type | Ticket | Change |
|---|---|---|---|
| `Matter_Records_Retrieval_MMI` | Record-triggered flow on Matter | DDT-1002 | New element `Advance_Medical_Damages_To_Mode_2` |
| `Matter_Records_Retrieval_MMI` | FlowDefinition | DDT-1002 | Activates the new version |
| `backfill-medical-damages-to-mode-2.apex` | Anonymous Apex script | DDT-1002 | One-off catch-up for stranded rows |
| `Damages__c.Records_Received_Date_Required` | Validation rule | DDT-1024 | Repaired |
| `damagesTable` | LWC | DDT-1024 | "Edit treatment dates" toggle; amounts required on the Mode 3 form; treatment-date check on save |
| `DamagesTableControllerTest` | Apex test | DDT-1024 | Fixtures updated; two tests for the repaired rule |
| `Records_Requests_Needing_Follow_Up` | Report | DDT-1024 | New |

**Out of scope**

- **Ask 1, the Mode 2 follow-up log.** Split out to DDT-1100 as its own ticket.
- **Ask 3, auto-populated service dates.** Closed with nothing to build (see 6.2).
- **Record Status as a required field.** You cannot be in Mode 3 without it, so a rule requiring it could never fire, which is exactly the bug being fixed.
- **36 more Mode 1 medical damages on matters already past Records Retrieval** (Litigation 24, Negotiation 5, Resolved 4, Demand Phase 3). Moving them would queue records requests on cases already settled or in suit. It was decided to leave them.

## 4. How it works (background)

```
Damages__c, Damage_Type__c = Medical
  Mode 1  Treating     Records_Status__c blank
  Mode 2  Requesting   Ready to Request -> Requested
  Mode 3  Obtained     Received Records | Received Bills | Received Both
```

- **Stage change (before DDT-1002).** `Matter_Records_Retrieval_MMI` fires when `Current_Stage__c` changes to `Records Retrieval`. It completed the "Confirm MMI" task and set MMI fields. It did nothing to damages.
- **Moving to Mode 3 in the Damages table.** The component asks for the received date, **changes the status first**, and only then opens the edit form where amounts and dates are typed. This order matters for everything in section 5.
- **Mode 3 dates are read-only by default.** DDT-271 made them read-only on purpose: a billed amount logged against the wrong service window ends up in a document sent to a carrier, with nothing downstream to catch it.
- **The lien letter.** The `PI Release of Lien` Dochly template merges `First_Treatment_Date__c` and `Last_Treatment_Date__c` directly.

## 5. Root cause

### DDT-1002

Not a defect: the step was never built. Checked rather than assumed: none of the 17 active Matter-triggered flows mentioned damages or records status; there were no flows, triggers, Process Builders or workflow rules on `Damages__c`; and only the Damages table's own controller referenced `Records_Status__c`.

Side finding: `Matter_Records_Retrieval_MMI` had been **active in production since July and existed nowhere in git**. The first commit of PR #366 recorded the live version unchanged, so the real change reads as a diff.

### DDT-1024

**A guard that never fired.** `Records_Received_Date_Required` read:

```
ISPICKVAL(Records_Status__c, 'Received') && ISBLANK(Records_Received_Date__c)
```

`Received` was no longer an active picklist value. The three-mode workflow had replaced it with `Received Bills`, `Received Records` and `Received Both`. The rule showed **Active** in Setup, passed every deploy, and tested for a state nobody could be in. That is why 162 of 349 Mode 3 records had no received date.

**The lesson.** When a picklist value is retired, *writing* it fails loudly. A *validation rule* that tests it fails silently, forever, while still looking healthy.

**The first fix attempt failed for a second reason.** Two new validation rules (amounts required, treatment dates required) were written and then withdrawn. They fired on the save that changes the status, but the component changes the status **before** it opens the form holding those fields. Production validation caught it: 6 Apex tests failed, three on `markRecordsObtained`. Against the 220 Mode 2 records about to make that move, the amounts rule would have blocked **209** and the dates rule **172**. A smaller gate didn't help: `ISNEW()` alone fires on nothing, because **0 of 349** Mode 3 records were created in Mode 3; all arrived by transition.

## 6. Solution

### 6.1 DDT-1002: a third element in the existing flow

Not a new flow. One new Update Records element in `Matter_Records_Retrieval_MMI`, chained after the existing task update:

```
Update Damages__c where
  Matter__c        = the triggering Matter
  Damage_Type__c   = Medical
  Records_Status__c is null
set Records_Status__c = 'Ready to Request'
```

- It writes only to damages on the triggering matter.
- Neither damages validation rule blocks it: one fires only on `Requested`, the other only on non-medical types.
- `Ready to Request` is an existing active value.

**Back-fill.** `scripts/apex/backfill-medical-damages-to-mode-2.apex` with `DRY_RUN` first; safe to re-run, since it only picks up rows still blank. The dry run found 48 rows across 23 matters. When the rows were read individually, **44 were on matters the firm had disengaged from or dropped**, some since the previous November. Running the script would have queued records requests for people the firm no longer represents, so only the 4 live ones were moved, by hand.

### 6.2 DDT-1024

**Repaired received-date rule.** Now tests all three Mode 3 values, medical only, and fires only on create or a status change:

```
(ISNEW() || ISCHANGED(Records_Status__c))
&& ISPICKVAL(Damage_Type__c, 'Medical')
&& (
    ISPICKVAL(Records_Status__c, 'Received Bills')
    || ISPICKVAL(Records_Status__c, 'Received Records')
    || ISPICKVAL(Records_Status__c, 'Received Both')
)
&& ISBLANK(Records_Received_Date__c)
```

- **It's safe as a rule** because the component prompts for the received date and sends it *with* the status change, so the field is filled at the moment the status flips.
- **Gated on the status change** so the 349 existing Mode 3 records stay editable. A plain rule blocks every save on a violating record, so someone correcting an amount on an old record would be stopped and asked for a date nobody remembers. The decision was to catch everyone going forward and clean the back catalogue deliberately.
- It mainly bites on status changes made outside the Damages table (Data Loader, the record page, an integration), which is where a guard is wanted.

**Amounts required on the Mode 3 form (PR #456).** `Billed_Amount__c` and `Outstanding_Balance__c` are `required` on the Mode 3 edit form, so it won't submit without them. The requirement applies only to `Received Bills` and `Received Both`. Requiring an amount on `Received Records` would make that status unsaveable, and it exists precisely to record "records in, bill still out". A $0 Outstanding Balance saves. 32 existing records missing an amount now need one before any other edit saves.

**Editable treatment dates, behind a toggle (PR #456).** The Mode 3 view looks exactly as before. A new **"Edit treatment dates"** link swaps First and Last Treatment Date to inputs in place. The toggle resets every time the form opens, so nobody lands on an editable view. This keeps DDT-271's guardrail and removes the walk back to Mode 1.

**Treatment dates required, checked on save (PR #464).** 101 Mode 3 records were missing at least one treatment date. A `required` attribute doesn't work here, because with the toggle shut the dates are read-only output fields. So the check sits in the form's save handler:

- It applies to `Received Records` and `Received Both`, not `Received Bills` (the dates come off the records).
- **When it blocks, it opens the toggle** and names the missing date, rather than refusing a save against a field the user can't see.
- A date the user **cleared** counts as missing. It is not silently replaced by the stored value.
- It judges the status **submitted** in the form, and only on the Mode 3 form. The review caught an earlier version that could block a Mode 2 row on a form with no date fields on it.

**Ask 3 closed with no build.** The requester clarified she wanted Mode 1 dates carried into Mode 3, editable there, and feeding the lien letter. `First_Treatment_Date__c` and `Last_Treatment_Date__c` are **the same two fields in both modes**, so there is nothing to carry across. The editable part shipped in PR #456, and the lien letter already merges those fields. `Service_Date__c` was left unused: it is a single date, can't hold a treatment window, and was filled on 1 record of 877.

**Overdue follow-up report.** `Records_Requests_Needing_Follow_Up`: open medical requests past their own `Next_Follow_Up_Date__c`, or with no follow-up date, oldest first. Baseline on 2026-09-22: 157 rows, 116 past their date (33 by over a month) and 41 with no date. The oldest open request was 246 days old.

**CI check (PR #465).** A new CI leg catches validation rules that test a picklist value that isn't active. On its first run it found a second dead rule, on another object, which became its own ticket.

## 7. Testing

### DDT-1002

- **Sandbox, fired for real.** A matter moved into Records Retrieval: 2 Mode 1 medical damages flipped; 2 already at Ready to Request did not change; 4 non-medical rows did not change; a Mode 1 medical damage on a different matter did not change. The existing MMI behaviour still worked.
- **Production validation:** 2 of 2 components, 27 of 27 tests. A validation compiles the flow; it does not run it.
- **Live check (2026-09-14).** A matter moved into Records Retrieval in the normal course of work, not as a test. Its four Mode 1 medical damages flipped to Ready to Request in the same second. A row already received and rows already at Ready to Request were left alone. No damage on any other matter moved. The org-wide count of non-medical rows was unchanged.

### DDT-1024

- **Validation rule:** 17 scenarios in the sandbox, 0 failures, via two committed scripts. Each rule was re-tested **in isolation**, because with several fields blank the first rule to fire hid whether the others worked.
- **`DamagesTableControllerTest`** caught the withdrawn rules (6 failures). Lesson: **run the tests of the code that writes the object**, not only the tests of what you changed. A validation rule breaks Apex without being Apex.
- **Jest:** 36 tests after PR #456 (8 new) and 52 after PR #464 (10 new), each new assertion mutation-tested.
- **Browser check (PR #456, production):** dates open on the toggle, save, and are read-only again on reopen; 7/16 entered, 7/16 stored (no date shift); `Received Both` with a blank amount is blocked; `Received Records` with blank amounts saves; a $0 balance saves; the report returned 164 rows.
- **After PR #464:** missing-date records fell from 101 to 99 the next day as staff filled them in. The requester confirmed on 2026-09-24 that it was working.

## 8. Step by step

### DDT-1002

1. Confirm nothing already does it: read every active flow on Matter and check for flows, triggers and rules on `Damages__c`.
2. Retrieve the flow from production. If it isn't in git, commit the live version unchanged first.
3. Add the Update Records element (6.1) after the existing elements.
4. Test in the sandbox by actually moving a matter's stage, then read the rows back.
5. Validate against production, get approval, deploy.
6. **Production deploys flows as Draft.** Confirm the version this deploy created is the Active one and contains `Advance Medical Damages To Mode 2`. Don't just check for a version number; someone saving a draft in Flow Builder can shift it.
7. Run the back-fill with `DRY_RUN = true`. **Read the rows before running it for real.** Check whether each matter is still active.
8. Wait for a real matter to reach Records Retrieval and check its damages.

### DDT-1024

1. Measure the Mode 3 fill rates (section 2).
2. Read each validation rule on `Damages__c` against the picklist's **active** values.
3. Repair `Records_Received_Date_Required` (6.2), gated on `ISNEW() || ISCHANGED(Records_Status__c)`.
4. Add the toggle, the form-required amounts and the save-time date check to `damagesTable`.
5. Validate against production with `RunSpecifiedTests DamagesTableControllerTest`.
6. Open a Mode 3 medical damage in the browser and try each case in section 7 before and after deploy.
7. Deploy the report and confirm it returns rows.

## 9. Rollback

- **DDT-1002:** set the FlowDefinition's active version back to the previous one. Leave damages already moved to Ready to Request: it's a legitimate value a person could have set, and reverting it could undo real work.
- **DDT-1024:** the `required` attributes and the save check are small, isolated changes in `damagesTable`; redeploy the previous bundle. Deactivating the repaired rule returns to the old state, where nothing was enforced.

## 10. Known limits

- **`Current_Stage__c` is free text**, not a picklist. The flow and back-fill match the literal `Records Retrieval`. Reword that stage anywhere and both stop matching, silently.
- **The flow fires on the stage change only.** A medical damage added after the matter is already at Records Retrieval stays in Mode 1.
- **The received-date rule catches status changes only.** The 162 existing records without a received date are left to be cleaned up from the report, not forced.
- **The amounts and dates checks do touch old records**: 32 missing an amount and about 100 missing a date must be completed at their next Mode 3 save.
- **The Mode 2 to Mode 3 move is not gated** for treatment dates; the row is caught at its next Mode 3 save.
- **The follow-up log** (ask 1) is DDT-1100.

## 11. Reuse for similar issues

**How to recognise one:** a required field is blank on many records even though a rule "requires" it.

- **Check every validation rule against the picklist's active values** whenever a picklist changes. An Active rule on a retired value never fires.
- **Know the write order before choosing where to enforce.** If the UI changes the status before showing the fields, a rule on the status change blocks people with nothing to fill in. Enforce on the form's save instead.
- **When you block, show the field.** Open the hidden input rather than refusing against something off screen.
- **Gate new rules on `ISCHANGED`** so the back catalogue isn't frozen for unrelated edits.
- **Extend the automation that already fires** on a transition rather than adding a new flow.
- **Read back-fill rows before running them.** A dry run's count says nothing about whether each row should be touched.
- **Test the code that writes the object** whenever you add a validation rule.
