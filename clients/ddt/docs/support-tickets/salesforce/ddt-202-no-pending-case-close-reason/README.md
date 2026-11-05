# No Pending Case option when closing a potential case

| | |
|---|---|
| **Ticket** | DDT-202 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-19 |
| **Priority** | Medium |
| **Reported by** | Intake team, 2026-06-18 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, picklist metadata) |
| **Area** | Closing a potential Court Case (Opportunity) as lost |
| **Related** | Not recorded in the ticket |
| **Code** | Commit "DDT-202: Add "No Pending Case" Defense reason to Close Potential Case modal" in the `ddt-salesforce` repo |

---

## 1. Summary

Intake asked for a **No Pending Case** reason in the Close Potential Case modal, for prospects who turn out to have no pending case to defend. Without it, reps had to pick an unrelated reason, which skewed close-reason reporting.

The value was added to the `Lost_Reason__c` picklist on Opportunity as a Defense-only reason in the **Not Eligible** category, deployed to production and checked live on 2026-06-19.

## 2. What was reported

On 2026-06-18 an intake rep asked for a new reason in the Close Potential Case modal. The modal showed categories such as Price and Competition but nothing for "the prospect has no pending case". When asked which category it belonged under, the requester answered "Not Eligible or Resolved before retention". It was placed under **Not Eligible**.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `scripts/data/lost-reasons.csv` | Source data | One new row: `Not Eligible,No Pending Case,FALSE,TRUE,FALSE` |
| `Opportunity.Lost_Reason__c` | Picklist field | New value `No Pending Case` |
| `Opportunity.Lost_Reason_Category__c` | Formula field | Maps `No Pending Case` to `Not Eligible` |
| `Opportunity` record types `Hired` and `Potential` | Record types | `No Pending Case` made available on both |
| `lostReasonModal/lostReasonData.js` | LWC data file | New entry: category `Not Eligible`, `sos: false`, `defense: true` |

**Out of scope**

- **Reports and page layouts.** The ticket said none were needed unless the value had to appear in existing close-reason reports. Not recorded whether any report was changed.
- **SOS and PI.** The value is Defense-only. PI reasons do not live on Opportunity.

## 4. How it works (background)

```
Rep opens a potential Court Case (Opportunity, Potential record type)
  -> clicks the close action, which opens the lostReasonModal LWC
  -> the modal lists reasons grouped by category, filtered to Defense or SOS
     from the case type
  -> the chosen value is saved to Opportunity.Lost_Reason__c
  -> Lost_Reason_Category__c (a formula) derives the category for reports
```

The reason list is **generated, not hand-edited**. `scripts/data/lost-reasons.csv` holds one row per reason (category, value, and whether it applies to SOS, Defense and PI). `scripts/build-lost-reason-metadata.js` reads that file and rewrites four things together: the picklist value set, the category formula, the two record types' allowed values, and the LWC's `lostReasonData.js`. The formula's description says "Do not edit by hand".

## 5. Root cause

Not applicable: this was a request for a new value, not a defect.

## 6. Solution

One line added to the CSV:

```text
category,value,sos,defense,pi
Not Eligible,No Pending Case,FALSE,TRUE,FALSE
```

Then the generator was run:

```bash
node scripts/build-lost-reason-metadata.js
```

That produced the matching changes in all four places at once. The new LWC entry reads:

```js
"No Pending Case": {
    "category": "Not Eligible",
    "sos": false,
    "defense": true
},
```

Why it was done this way:

- **Edit the CSV, not the XML.** The picklist, the formula, the record types and the LWC all have to agree. Editing one by hand leaves the others behind. A value missing from a record type is not selectable on that record type, and a value missing from the formula falls into the formula's default category.
- **Add it to both record types.** Potential is where it is chosen. Hired is included because the picklist is restricted, and a closed record that later moves record type must keep a valid value.

The generator also rewrote two existing record-type values from an encoded form (`%2F`) to a plain `/`. That came from regenerating the files, not from this request.

## 7. Testing

- **Live check.** The ticket's closing comment says the change was deployed to production and verified live on 2026-06-19.
- **Automated tests.** Not recorded in the ticket.

**Not covered:** whether any existing close-reason report needed the new value.

## 8. Step by step

1. **Confirm the category with the requester** before building. Here it was Not Eligible.
2. **Add a row to `scripts/data/lost-reasons.csv`** with the category, the exact value, and `TRUE`/`FALSE` for SOS, Defense and PI.
3. **Run `node scripts/build-lost-reason-metadata.js`.**
4. **Read the diff.** It should touch `Lost_Reason__c`, `Lost_Reason_Category__c`, both record types and `lostReasonData.js`, and nothing else you did not expect.
5. **Retrieve the production versions** of the record types and field before deploying, because existing files can be stale on a branch. Apply the change on top of production's version.
6. **Deploy** the five metadata files and the LWC bundle after approval.
7. **Open a potential Defense case** and confirm the value appears under Not Eligible and saves.

## 9. Rollback

Set the row's Defense flag to `FALSE` in the CSV and regenerate, or deactivate the value in Setup. Do not delete the picklist value once records use it: the picklist is restricted, so existing records would hold a value the field no longer accepts.

## 10. Known limits

- **The generator rewrites whole files.** Running it on a branch whose record-type files are older than production's can undo unrelated production changes. Compare against production first.
- **The category is a formula over the value text.** Renaming a value without regenerating sends it to the formula's default category.

## 11. Reuse for similar issues

**How to recognise one:** a request to add, rename or hide a close reason in the Close Potential Case modal.

- Change `scripts/data/lost-reasons.csv` and regenerate. Never hand-edit the generated files.
- Ask for the category and the practice area (SOS, Defense, PI) up front. The ticket's requester answered with two candidate categories; pick one and record it.
- To retire a value, mark it deprecated so existing records stay valid, rather than removing it.
