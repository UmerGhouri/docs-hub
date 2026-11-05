# Illinois lien reduction classification and totals

| | |
|---|---|
| **Ticket** | DDT-576 |
| **Project** | Support tickets (Salesforce) |
| **Status** | Live since 2026-08-31 (classification and totals only; the cap, pro-rata and letter fields were held) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom fields, roll-up summary fields, Apex, LWC, permission sets) |
| **Area** | PI settlements: the Liens table on a Matter's Financials page, and reducing medical liens under Illinois law |
| **Related** | None recorded |
| **Code** | PR #264 and PR #267 in the `ddt-salesforce` repo. PR #263 carries the same title as #264; no merge of it exists on `main`. |

---

## 1. Summary

Staff asked for the Financials calculator to do Illinois statutory lien reductions end to end: classify each lienholder, total the bills by class, apply the statutory caps, work out each lienholder's pro-rata offer, and print it into reduction letters. The build was **split on purpose**. Classifying liens and totalling them by class carries no legal position, so it shipped. The cap, the pro-rata offer and the letter fields encode a reading of the statute, and a figure in a letter is the firm telling a lienholder in writing what it is owed, so they were held until the reading was settled and checked on real matters.

What shipped: a three-value **Statutory Class** on each lien, roll-up totals by class on the Matter, two guard counts that flag when those totals are incomplete, and a Liens table grouped by class with subtotals.

## 2. What was reported

Lien-reduction maths for Illinois PI matters was done by hand in a spreadsheet, then copied into letter templates. The request (2026-07-29) had five parts:

1. Classify each lienholder as a medical **provider** or a health care **professional**.
2. Show **Total Provider Bills** and **Total Professional Bills** on the Matter.
3. Apply the Illinois caps: 40% of the settlement split into two 20% buckets when both classes are present, or one third when only one class is present.
4. Work out each lienholder's pro-rata share of its class bucket.
5. Expose all of it as merge fields for reduction letters.

At go-live there were **155 liens** in production, none classified, and **57** of them had no billed amount recorded.

## 3. Scope

**In scope (requirements 1 and 2)**

| Component | Type | Change |
|---|---|---|
| `Lien__c.Statutory_Class__c` | Restricted picklist | New. Three values, no default (see 6.1) |
| `Matter__c.Total_Provider_Bills__c` | Roll-up summary (SUM) | New |
| `Matter__c.Total_Professional_Bills__c` | Roll-up summary (SUM) | New |
| `Matter__c.Total_Non_Covered_Lien_Bills__c` | Roll-up summary (SUM) | New |
| `Matter__c.Unclassified_Lien_Count__c` | Roll-up summary (COUNT) | New |
| `Matter__c.Liens_Missing_Billed_Amount__c` | Roll-up summary (COUNT) | New, in PR #267 |
| `PI_Matter_Access`, `Damages_Records_Mode` | Permission sets | Edit on the new picklist; read on the Matter roll-ups (`PI_Matter_Access`) |
| `Lien__c-Lien Layout` | Page layout | Statutory Class placed next to Negotiation Status |
| `LienManagerController` and its test | Apex | Returns liens in class sections with subtotals |
| `lienManager` | LWC | Grouped table, subtotals, warnings, class on the create and edit modals |

**Out of scope, deliberately held**

- **The 40% and one-third caps** (requirement 3).
- **Per-lienholder pro-rata offers** (requirement 4).
- **Any letter merge field that states a dollar offer** (requirement 5). This is the hard line: nothing prints a figure to a lienholder until the statutory reading is confirmed and a partner has read the output on real matters.

## 4. How it works (background)

```
Matter__c
  └─ Lien__c (one per lienholder)
       Lienholder, Original_Lien_Amount__c (billed face value), negotiated payoff, status

Liens table (lienManager LWC) on the Matter's Financials page
  -> LienManagerController reads the Matter's liens
  -> before: one flat list and one "Total Medical Bills" figure
```

**The statute.** The Illinois Health Care Services Lien Act, 770 ILCS 23, limits what covered lienholders can recover from a PI settlement. As researched for this ticket:

- The 20% / 20% split between professionals and providers applies **only if** total covered liens reach 40% of the recovery. Below that there is no statutory reduction.
- Between classes the split is flat (20% each), not proportional to billing. Within a class, lienholders share pro rata.
- If one class does not use its full 20%, the unused amount is **reallocated** to the other class, up to a one-third ceiling per class.
- The Act covers a **closed list** of licensed categories. Many lienholders on a PI matter are not on it.

## 5. Root cause

This was a feature request, not a fault. What matters here is **why the ticket's formula was not built as written.**

1. **The ticket's Rule A always divided 40% into two buckets.** The statute only applies the split once covered liens reach 40% of the recovery. A calculator that always splits would generate reduction demands on matters where none is owed.
2. **The ticket had no reallocation.** On a $25,000 recovery where professionals bill $3,000 and providers bill $40,000, the ticket's method pays $3,000 and $5,000 ($8,000 total); the statute pays $3,000 and $7,000 ($10,000). A first draft of the research said the client's net is the same either way. **That was corrected the same day:** when one class underuses its bucket, skipping reallocation pays out less than 40%, so the client keeps more than the statute allows and covered providers get less, which is exactly what a provider's lien counsel would challenge.
3. **Two classes were not enough.** Chiropractors, imaging companies, pain clinics, PT and orthopedic groups billing as entities, Medicare and Medicaid, health insurers and claims administrators are **not covered by the Act**. Production liens already included Medicaid, a commercial health insurer, a claims administrator and a litigation funder. Forcing them into "provider" or "professional" would (a) assert in writing that the statute reduces a claim it does not reduce, and (b) inflate the class total, which is the pro-rata denominator, so every genuinely covered lienholder would be offered less than it is owed.
4. **The ticket's example figures were transcribed wrongly.** The ticket gave $21,000 / $29,805.76; the spreadsheet says $21,914.10 / $29,085.76, and only the spreadsheet's figures add up to its own overall total. Anyone checking a build against the ticket would have chased a variance that did not exist.

An early alternative, splitting the 40% between classes in proportion to billing, was also considered and **rejected**: the statute's text is a flat percentage per class, and the reallocation clause only makes sense with fixed buckets.

## 6. Solution

### 6.1 The classification field (PR #264)

`Lien__c.Statutory_Class__c`, a restricted picklist with three values and **no default**:

| API value | Label |
|---|---|
| `Health Care Professional` | Health Care Professional (770 ILCS 23) |
| `Health Care Provider` | Health Care Provider (770 ILCS 23) |
| `Not Covered by the Act` | Not Covered by 770 ILCS 23 |

Why this way:

- **Three values, not two.** The third keeps non-covered lienholders out of the cap maths while still showing their total.
- **The statutory category lists are in the field description and help text**, so whoever classifies reads the law instead of guessing from the lienholder's name. Professional is individuals only (licensed physician, dentist, optometrist, naprapath, clinical psychologist, physical therapist). Provider is the Act's listed facility types (licensed hospital, home health agency, ambulatory surgical treatment centre or facility, long-term care facility, emergency medical services). Everything else is Not Covered.
- **Blank is a real state.** Nothing is assigned automatically, and blank is counted (6.2).

### 6.2 The roll-ups and the two guards (PRs #264 and #267)

All roll-ups sum `Lien__c.Original_Lien_Amount__c`, the **billed face value**, not the negotiated payoff. These are the figures the cap maths will eventually divide by.

| Field | Operation | Filter |
|---|---|---|
| `Total_Provider_Bills__c` | SUM | class = `Health Care Provider` |
| `Total_Professional_Bills__c` | SUM | class = `Health Care Professional` |
| `Total_Non_Covered_Lien_Bills__c` | SUM | class = `Not Covered by the Act` |
| `Unclassified_Lien_Count__c` | COUNT | class not equal to any of the three values |
| `Liens_Missing_Billed_Amount__c` | COUNT | `Original_Lien_Amount__c` is blank |

Why two COUNT guards:

- **A blank lien hides itself in a SUM.** A lien with no class, or a class but no billed amount, adds 0 to a total and raises no flag. Because the class total is the pro-rata denominator, one missing lien quietly overstates every other lienholder's offer. A COUNT cannot be fooled that way.
- **`Unclassified_Lien_Count__c` filters "not equal to all three values"** rather than "equals blank", so it counts the unclassified lien whether or not it has an amount.
- **`Liens_Missing_Billed_Amount__c` was added in PR #267** because the first guard cannot see a lien that *is* classified but has no amount, and that was the common case: 57 of 155 liens.

### 6.3 The grouped Liens table (PR #267)

- `LienManagerController` returns the liens in four fixed-order sections, **Provider, Professional, Not Covered, Unclassified**, each with its billed subtotal, and exposes the three class totals. Empty sections are dropped.
- Each section header shows how many of its liens have **no billed amount**, so a subtotal covering only some of its rows does not look complete.
- A warning above the table appears when anything is unclassified or unpriced, saying not to calculate reductions yet. An incomplete denominator still renders a plausible number, which is worse than no number.
- A missing billed amount shows as a **dash with a tooltip**, not `$0.00`, which would look like a genuine zero bill. A section whose liens are all unpriced shows a dash for its subtotal too.
- Statutory Class is on the **create and edit modals**. The layout entry alone was not enough: staff work through this component, not the record page. (PR #264 put the field on no layout or component, so on its own nobody could set it.)
- Row striping was removed, because each section is its own table body and the stripe pattern restarted per section.

## 7. Testing

**Sandbox checks**, with anonymous Apex inside a savepoint that was rolled back:

- Classifying the sample matter's liens the spreadsheet's way reproduced it exactly: **$21,914.10 provider / $29,085.76 professional**.
- Classifying the same ten bills by the statute instead gave **$20,369.10 provider / $675.34 professional / $29,955.42 not covered**. The gap between those two results is the classification problem in section 5, in numbers.
- An unclassified lien with an amount and one without both raised `Unclassified_Lien_Count__c` to 2, with the class totals unchanged.
- Three liens (priced and classified, classified with no amount, neither) gave `Liens_Missing_Billed_Amount__c` = 2, `Unclassified_Lien_Count__c` = 1 and a provider total of 1000, proving the classified-but-unpriced lien adds nothing to its total and is now visible.

**Automated tests (PR #267).** 13 Apex tests pass, including two new ones for section order, the eligibility flags and empty-section suppression. 227 Jest tests pass across 23 suites, including new ones for grouped rendering, the incomplete-data warning and the dash for a missing amount. PR #267 also rewrote five pre-existing expect-a-throw tests whose `catch` blocks were empty, so they now assert on the exception. The static-analysis check had flagged them once the PR touched that Apex.

**Production validation.** PR #264 was validated against production as a dry run (7 of 7 components).

**Live state (2026-11-05):**

```sql
SELECT Statutory_Class__c, COUNT(Id) n FROM Lien__c GROUP BY Statutory_Class__c
```

| Statutory Class | Liens |
|---|---|
| blank | 205 |
| Health Care Professional | 31 |
| Health Care Provider | 3 |
| Not Covered by the Act | 9 |

So 43 of 248 liens have been classified since go-live.

**Not covered:** the cap, reallocation and pro-rata maths, which are not built.

## 8. Step by step

1. **Read the statute before modelling anything.** Confirm the category lists, the 40% threshold, the flat 20/20 split and the reallocation rule against current statutory text.
2. **Create `Lien__c.Statutory_Class__c`** as a restricted picklist with the three values in 6.1, no default, and the category lists in the description and help text.
3. **Create the five roll-ups on `Matter__c`** from 6.2, summing or counting over `Lien__c.Matter__c`.
4. **Grant the fields** in the permission sets staff already use for liens (`PI_Matter_Access`, `Damages_Records_Mode`): edit on the picklist, read on the roll-ups.
5. **Put the picklist where staff work**: the Lien layout and the `lienManager` create and edit modals.
6. **Group the table** in `LienManagerController` and `lienManager`, with subtotals, missing-amount badges, the warning and the dash for a missing amount.
7. **Check against the spreadsheet in a sandbox**, inside a savepoint, with the sample matter's figures.
8. **Validate against production**, get the PR approved, deploy.
9. **Classify the existing liens** from the definitions, and chase the missing billed amounts. The totals read zero until this is done.

## 9. Rollback

No rollback plan is recorded in the ticket. Everything here is additive: new fields, new sections in an existing component. Reverting the table means redeploying the previous `lienManager` and `LienManagerController`. Deleting `Statutory_Class__c` would also erase every classification staff have entered since go-live, so leave the field in place even if the UI is reverted.

## 10. Known limits

- **Totals are only as good as the classification.** Most production liens are still unclassified (205 of 248 on 2026-11-05), so most Matters' class totals are incomplete. The guard counts and the table warning say so.
- **Missing billed amounts shrink the denominator.** Every lien without `Original_Lien_Amount__c` makes the other lienholders' eventual share look larger than it should.
- **No reduction is calculated yet.** Requirements 3, 4 and 5 are not built. When they are: gate on the 40% threshold, apply 20/20, reallocate unused capacity, apply the one-third ceiling, then go pro rata within each class, in that order. Once reallocation exists, the one-third ceiling can bind; it does when one class holds more than about 83.3% of covered billed charges.
- **Out-of-state lienholders.** The research flagged that some lienholders on the sample matter were outside Illinois, and asked whether the Illinois cap applies to them. Not recorded in the ticket whether that was settled.
- **Negotiating non-covered lienholders** down to a comparable figure is a business decision. The only position taken here is not to tell them the statute requires it.
- **The legal reading is internal analysis for a build decision, not a formal opinion.** Recheck the category lists against current statutory text before any field prints a figure to a lienholder.

## 11. Reuse for similar issues

**How to recognise one:** a request to automate a calculation that ends in a document sent to a third party, especially one based on a statute or contract.

- **Split what carries a legal position from what does not.** Ship the classification and totals first; hold anything that asserts an amount someone is owed.
- **Read the source, not the spreadsheet.** The spreadsheet encoded a simplified version of the rule that would have produced wrong demands.
- **Check the ticket's example numbers against the original.** Here they were mistyped.
- **Make "unknown" a visible state.** Leave the picklist blank by default, count the blanks, and warn on screen.
- **Use COUNT guards beside SUM roll-ups** when a missing value silently becomes a zero in the total.
- **Show a missing amount as missing**, never as `$0.00`.
- **Put a new field where staff actually edit**, not only on the page layout.
