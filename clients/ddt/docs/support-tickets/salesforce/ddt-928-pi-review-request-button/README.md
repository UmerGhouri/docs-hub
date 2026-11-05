# Review Request button on the PI matter page

| | |
|---|---|
| **Ticket** | DDT-928 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-22 (ticket marked done 2026-09-21) |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-03 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow, custom fields, formula fields, quick action, permission set, email templates), Heymarket SMS |
| **Area** | Asking a client for an online review once their case is done |
| **Related** | DDT-1067 (later fix to the same flows' "already reviewed" path) |
| **Code** | PR #424 and PR #429 in the `ddt-salesforce` repo |

---

## 1. Summary

The PI team asked for the **Review Request** button that criminal defense Court Cases already have. It was not a button copy: the button runs a five-screen flow built around Court Cases, and Review Request records had no way to point at a PI matter. Instead of copying the flow, a PI path was added **alongside** the existing one, so one flow serves both and review reporting stays in one place. Review requests gained a `Matter__c` lookup, six client formulas gained a Matter fallback that existing records never take, and two PI email templates were built by hand.

## 2. What was reported

- **2026-09-03:** the PI team wanted to send a review request straight from a PI matter, the same way the criminal defense team does from a Court Case. Their screenshots showed the Court Case action bar with **Review Request**, and a PI matter's action bar without it.
- **2026-09-07:** the support team found it was a feature, not a layout change, and asked the firm's approver to choose between copying the flow for PI or making one flow serve both.
- **2026-09-08:** approved as **one flow serving both**, with four conditions: do not change what the five screens do for a Court Case; migrate none of the existing Review Request records (13,390 at the time); state what a PI review request attaches to before deploying; and run a criminal defense case through the existing button in a sandbox to prove it is unchanged.
- **2026-09-20:** the attachment shape was approved.
- **2026-09-22:** live in production; the PI team was asked to test.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `ReviewRequest__c.Matter__c` | Lookup to Matter | New. Exactly one of `Matter__c` or `Opportunity__c` is ever set |
| `Client_Id__c`, `Language__c`, `RevReq_Client_Email__c`, `RevReq_Client_First_Name__c`, `RevReq_Client_Last_Name__c`, `RevReq_Client_Phone_Number__c` on `ReviewRequest__c` | Formula fields | Fall back to `Matter__r` only when `Opportunity__c` is blank |
| `Reviewed_by_Client__c`, `Review_Requested_Date__c`, `Review_Requested_By__c`, `Review_Request__c` on `Matter__c` | Fields | New tracking fields, mirroring the Court Case ones |
| `Google_Review_URL__c`, `FindLaw_Review_URL__c`, `Facebook_Review_URL__c` on `Matter__c` | Url fields | New. The PI email reads them (PR #429) |
| `SF_Review_Request_Button_for_Court_Case` | Screen flow | PI path added beside the Court Case path |
| `Sub_Review_Request_Process` | Subflow | PI path added: creates the record, sends SMS and email |
| `Matter__c.Review_Request` | Quick action | New, added to the Matter record page in App Builder |
| `Matter_Wizard_Access` | Permission set | Read and edit on the new fields |
| `DDT_Review_Request_PI`, `DDT_Review_Request_PI_ES` | Email templates | New, built by hand in each org (see 6.4) |

**Out of scope**

- **Existing Review Request records.** None migrated or backfilled.
- **What the five screens do for a Court Case.** Unchanged, by condition.
- **`RevReq_ClientTag__c`.** It reads Court-Case-only fields (case type, citation), so it stays blank for PI.
- **The Matter record page file.** The repo's copy was 427 lines different from production, with the repo side ahead. Deploying either copy was wrong, so the button was added in App Builder instead.
- **Profiles.** Field access goes through the additive permission set.

## 4. How it works (background)

Before the change, on a Court Case only. (In this org the standard Opportunity object is labelled "Court Case"; a PI case is a `Matter__c`.)

```
Review Request button (quick action on the Court Case)
  -> SF_Review_Request_Button_for_Court_Case  (five screens)
       1. Was this client asked in the last 30 days?  -> if so, a red screen says who and when, and stops
       2. Pick a template by the client's language
  -> Sub_Review_Request_Process
       3. Rotate a review location: least recently used of the Google records across the three offices
       4. Send: two Heymarket SMS callouts and an email
       5. Create a ReviewRequest__c and stamp the Court Case
```

The 30-day check keys on the **client**, not the case, which is what stops a client being asked twice. The subflow is also called from four other places (the intake flow, two disposition flows and an Apex controller), all of which pass a Court Case.

## 5. Root cause

Not applicable as a defect: this was a feature request. The reasons it could not be a simple button copy:

1. **The button runs a flow built around Court Cases.** It reads Court Case fields throughout and cannot just be pointed at a Matter.
2. **Review Request records had no link to a PI case.** All existing ones attached to Court Cases through `Opportunity__c`, so a PI review request had nowhere to save its case.
3. **The client formulas all read through `Opportunity__r`.** On a PI review request they would show no name, email, phone or language.
4. **The existing email templates are bound to Opportunity.** A sandbox test of a PI email failed with `We don't recognize the field prefix Opportunity`.

## 6. Solution

### 6.1 Where a PI review request attaches

To two things: the **client Account**, which already worked because both review-request lookups key on the client and `Matter__c.Account__c` points at the same Account; and the **case**, through the new `ReviewRequest__c.Matter__c` lookup. Exactly one of `Matter__c` or `Opportunity__c` is set on any record.

### 6.2 Formulas that existing records never notice

Each of the six client formulas became:

```
IF( ISBLANK(Opportunity__c), Matter__r..., Opportunity__r... )
```

The Opportunity branch is byte-identical to production's. Every existing record has `Opportunity__c` set, so `ISBLANK` is false and they evaluate exactly what they did before. Checked after the deploy: **0 of 12,858** existing review requests read a blank client name, the same as before.

### 6.3 The flows: reads need no branch, writes do

- **Reads run unconditionally.** `Get_Matter` finds nothing when the record is a Court Case, and the Court Case lookup finds nothing when it is a Matter, so there is no fork at the top and the Court Case element order is untouched. Formulas pick whichever record was found.
- **Writes get their own copies.** Each write targets one record, so the PI path has its own update, assignment and create elements beside the Court Case ones rather than sharing them.
- **The SMS needs no new field.** The Heymarket action takes a phone string; the E.164 phone field also lives on Account and was populated on 293 of 298 Matter clients.
- **Production's copy of the subflow was used as the base, not the repo's.** Production runs the two Heymarket SMS callouts in a new transaction; the repo said current transaction. Building on the repo's copy would have silently changed that.

### 6.4 The PI email templates are built by hand

The PI templates cannot be deployed: Metadata API and a direct insert both reject the image references the Email Template Builder writes (`UNSAFE_HTML_CONTENT`). They are built in the Builder once per org, so their Ids differ per org, and the flow finds them by **developer name**, not Id. Developer names are unique org-wide, so exactly one can match. If neither is found, a decision ends the PI run **before any record is written**, so a missing template cannot lock a client out under the 30-day check.

### 6.5 Defects caught in review, and the lesson from each

- **A formula referenced itself** (`ClientFirstName` in its own else branch). It sat on the Court Case path too, so criminal defense SMS would have gone out with a blank first name.
- **`Send_Email` was not PI-aware.** Its related record and recipient read the Court Case, which is null on the PI path.
- **The new lookup had no field access.** Under the org's rule, a new field is invisible until a permission set grants it.
- **The "already reviewed" toggle read the Court Case**, so a PI client who had already reviewed would have been offered a new request.
- **Eleven `TEXT()` calls wrapped Text operands** (Ids and lookups), which is invalid. **Production's validation accepted them twice; the sandbox rejected them.** A green production validation is not proof a flow formula is valid. Validate flow changes in a sandbox too.
- **The three review-URL fields were first judged "write-only"** after checking only the flow for reads. The email template reads them. "Nothing reads it" needs every reader checked.

### 6.6 Rollout

The fields, lookup, formulas, permission set and quick action went out first, unreachable by users. The flows went out once the `TEXT()` fixes and PI templates were in place, active as v33 (button) and v16 (subflow). The button was then added to the Matter record page in App Builder, which is the moment the feature went live.

## 7. Testing

**Sandbox, 2026-09-21 and 22:**

| What | Result |
|---|---|
| Court Case through the button, after the PI work | unchanged: same screens, same email |
| PI matter through the button | review request created with `Matter__c` set and `Opportunity__c` blank |
| The PI email | arrived with the correct client name and all three review links |
| Matter stamping | the three tracking fields and the three URLs written |
| Rotation | a different office's Google record on each of two runs, least recently used first |
| PI 30-day check | red screen with the correct requester and date |
| Existing records in production | 0 of 12,858 with a blank client name |

**Also settled in the sandbox:** putting a Court Case Id into a `Matter__c` Id filter returns zero rows rather than an error, and the subflow's four other callers all pass a Court Case, so they only ever take the Court Case branch the button test already covered.

**Production validation** compiled the 18 components and ran one Apex test. It renders no screen, sends nothing and runs no flow, so it is not evidence about the flows; the sandbox run is.

**Not covered:** an automated test of either flow. The live check was handed to the PI team.

## 8. Step by step

1. **Confirm it is a feature.** Open the action behind the button. A Flow quick action is real work; get the approver's decision on copy versus shared flow before building.
2. **Retrieve both flows from production** and diff them against the repo. Build on production's copy.
3. **Add the lookup** from the shared child record to the new parent object, and the parent's tracking fields.
4. **Change the formulas** to `IF( ISBLANK(<existing lookup>), <new path>, <existing path> )`, keeping the existing branch byte-identical.
5. **Add the new path to the flows**: unconditional reads, separate writes, new elements placed at the end of their own type group (flow metadata needs same-type elements contiguous).
6. **Grant field access** in an additive permission set, including the new lookup.
7. **Run the sandbox tests in order**: baseline on the existing path first, then the existing path after the change, then the new path, then a repeat within 30 days.
8. **Build the PI email templates in the Builder** in production, set **Related Entity Type** to Matter (a clone keeps Opportunity), and swap the merge fields. Check with:

   ```sql
   SELECT DeveloperName, RelatedEntityType, IsActive FROM EmailTemplate
   WHERE DeveloperName IN ('DDT_Review_Request_PI','DDT_Review_Request_PI_ES')
   ```

   Expect two rows, both Matter, both active.
9. **Deploy the fields first, then the flows**, after approval, re-validating at deploy time.
10. **Add the quick action to the record page in App Builder.**
11. **Confirm as a PI team member, not an admin.**

## 9. Rollback

- **Flows:** redeploy production's previous active versions (v32 for the button, v15 for the subflow), retrieved and kept before the deploy.
- **Formulas:** revert to production's previous copies. They hold no data.
- **New fields, lookup and quick action:** remove from the permission set and delete. Clear any PI review request's `Matter__c` before deleting the lookup.

## 10. Known limits

- **The Court Case screens are no longer byte-identical.** One screen component's initial value now reads a formula whose Court Case branch is the original expression.
- **The PI templates live only in each org's Builder.** A new sandbox or a rebuilt org needs them built again, or the PI path ends without sending.
- **The "already reviewed" path was later found to flag the client's most recent review request regardless of which case the button was pressed on**, and was fixed under DDT-1067. At that point (2026-09-29) `Matter__c` was set on 0 of 13,532 review requests, so the PI path had not yet been used.
- **The Matter record page drift** (repo ahead of production) was left for its own ticket.

## 11. Reuse for similar issues

**How to recognise one:** "add the button that object A has to object B", where the button is a Flow quick action.

- **Open the action before estimating.** A button that runs a flow is a feature.
- **One shared flow beats a copy** when reporting must stay together; add the new path beside the old and keep the old one's elements untouched.
- **Fallback formulas that existing records never take** let a shared child object serve a second parent with zero data change.
- **Find every reader before calling a field unused**: flows, email templates, reports, page layouts.
- **Validate flow changes in a sandbox.** Production validation compiles metadata; it has accepted invalid flow formulas here.
- **Look up hand-built templates by developer name**, and stop before any write if the lookup finds nothing.
