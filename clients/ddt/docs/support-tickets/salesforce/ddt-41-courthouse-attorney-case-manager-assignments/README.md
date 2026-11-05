# Changing primary attorney and case manager assignments by courthouse

| | |
|---|---|
| **Ticket** | DDT-41, DDT-112, DDT-164, DDT-175, DDT-242, DDT-256, DDT-290, DDT-310, DDT-529, DDT-623, DDT-700, DDT-865, DDT-885, DDT-915, DDT-937, DDT-939, DDT-1190, DDT-1279 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-09 (each ticket went live on its own completed date, listed in section 2) |
| **Priority** | Medium; High for DDT-112, DDT-164, DDT-175 and DDT-256 |
| **Reported by** | Attorneys and firm staff, 2026-05-26 to 2026-10-08 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record-triggered Flow, routing records, data updates), DayBack calendar |
| **Area** | Court Case routing: who is Primary Attorney and Court Case Manager on a hired case |
| **Related** | None recorded in the tickets beyond the 18 listed |
| **Code** | PR #632 in the `ddt-salesforce` repo (DDT-1190 only). Everything else was configuration and data. |

---

## 1. Summary

Eighteen tickets over five months asked for the same kind of change: when the firm reorganises who covers a courthouse, a case type or a language group, the **Primary Attorney** and **Court Case Manager** on Court Cases must change, both on the cases that are already open and on every case hired from then on.

Future cases are routed by one record-triggered flow that reads a small table of routing records, `Court_Case_Attorney_Assignment__c`. Existing cases are **not** re-routed when that table changes, so every request has two halves: edit the routing records, then update the open cases in bulk. Most of the problems in this group came from doing one half without the other, or from scoping the bulk update wrongly.

## 2. What was reported

Each ticket is a reassignment request. Dates are when it was raised; the status is its own completed date.

- **DDT-41** (2026-05-26, done 2026-05-26): reassign every current and upcoming Maywood case **in DayBack** to a new primary attorney. A later comment (2026-06-02) confirms all current and upcoming Maywood cases in DayBack were reassigned.
- **DDT-112** (2026-06-04, done 2026-06-05): an urgent bulk change of primary attorney across eight courthouse groups, some also filtered by case manager, language or case type. The ticket itself flagged that it gave Maywood to a different attorney than DDT-41 had. How that conflict was resolved is not recorded in the ticket.
- **DDT-164** (2026-06-12, done 2026-06-15): new case manager by courthouse, effective Monday 2026-06-15, on all open, active and future cases, with a Spanish / English split for Lake County. The requester clarified that client language decides the split and that "all Chicago courthouses are branch courthouses". Closing note: all Court Cases were assigned and the data updated.
- **DDT-175** (2026-06-16, done same day): a new case manager mapping by courthouse and language (Markham split by English / Spanish, Rolling Meadows, and the four Kane County courthouses). Closing note: current records updated, and the future routing set.
- **DDT-242** (2026-06-29, done 2026-06-30): a new primary attorney for all current and future Rolling Meadows cases.
- **DDT-256** (2026-07-01, done same day): a Civil Asset Forfeiture case was given the courthouse's default attorney instead of the attorney who handles all Civil Asset Forfeiture cases. The attorney confirmed that Civil Asset Forfeiture **and Order of Protection** cases go to that attorney regardless of courthouse, and asked for any open misrouted ones to be moved. The case was corrected and the routing fixed (see 5.2).
- **DDT-290** (2026-07-08, done 2026-07-09): one primary attorney for seven Chicago branch courthouses, plus one specific courtroom at Skokie.
- **DDT-310** (2026-07-10, done 2026-07-13): one case manager for all four Kane County courthouses, all languages.
- **DDT-529** (2026-07-27, done 2026-07-28): fill in a primary attorney on open Skokie cases that had **none**, without overwriting any that already had one.
- **DDT-623** (2026-08-03, done 2026-08-04): one case manager for all Markham cases, open and future, all languages.
- **DDT-700** (2026-08-10, done 2026-08-11): one case manager for every SOS (Secretary of State) Court Case. The first report said 208 records were changed and quoted 812 SOS cases. The requester queried the 812; the correct figure was **89 active, open SOS cases**, and 812 was every SOS case ever hired. The requester confirmed only open cases were wanted.
- **DDT-865** (2026-08-28, done 2026-08-31): primary attorney by county and courthouse: all of Kane County to one attorney, Rolling Meadows and Maywood to a second, Bridgeview and Markham to a third. Backfill plus future cases.
- **DDT-885** (2026-08-31, done 2026-09-01): Rolling Meadows moved to a different primary attorney again, backfill plus automation.
- **DDT-915** (2026-09-02, done 2026-09-04): an outgoing attorney's SOS cases and upcoming docket appearances moved to an incoming attorney. Scope confirmed as open cases plus all future cases.
- **DDT-937** (2026-09-04, done same day): a Spanish-speaking case manager on open Spanish-language cases at five courthouses (four Lake County courts and Skokie), plus auto-assignment for future ones.
- **DDT-939** (2026-09-04, done 2026-09-07): open Rolling Meadows cases **still** showed the previous primary attorney after DDT-885. 72 open cases were reassigned (70 from one attorney, 2 from another).
- **DDT-1190** (2026-10-02, done 2026-10-07): a new case manager matrix for seven case managers across Rolling Meadows, Maywood, Bridgeview, Will County, Daley Center and the branch courts, and Kane County, split by language for Daley / branch and Kane. On 2026-10-05 the requester reported that the result was wrong: Kane cases had moved to the wrong case manager, and Spanish speakers at Skokie showed two different case managers. It was corrected on 2026-10-06 (see 5.4).
- **DDT-1279** (2026-10-08, done 2026-10-09): Rolling Meadows, 26th & Cal and the branch courts moved to a new primary attorney for existing and newly hired clients, and every Daily Docket appearance assigned to the outgoing attorney was to be unassigned.

## 3. Scope

**In scope**

| Component | Type | What these tickets changed |
|---|---|---|
| `Court_Case_Attorney_Assignment__c` | Custom object (routing table) | Rows added or edited so new cases route to the right people. Tab: App Launcher → **Court Case Attorney Assignments** |
| `[Court Case] Primary Attorney & CM Assignment v3` (`Court_Case_Primary_Attorney_CM_Assignment_v3`) | Record-triggered flow on Opportunity | Not edited by these tickets as far as the sources show; it reads the routing table |
| `Opportunity.Primary_Attorney__c`, `Opportunity.Court_Case_Manager__c` | User lookups on Court Case | Bulk-updated on open cases |
| `Court_Appearance__c.Assigned_Attorney__c` | User lookup on the Daily Docket appearance | Bulk-updated (DDT-915) or cleared (DDT-1279) on upcoming appearances |
| `Courthouse_List` | Global value set | Three courthouse values added (PR #632, DDT-1190) |
| `Court_Case_Managment` | Permission set | Not changed; DDT-1190 confirmed the requester already had it, which lets the holder create and edit routing rows |
| DayBack | Calendar app | Primary attorney changed on Maywood events (DDT-41) |

**Out of scope**

- **Closed cases.** Every ticket that set a scope limited the bulk update to open cases (DDT-700, DDT-915, DDT-939, DDT-1190). Three upcoming appearances on already-closed SOS cases were left for the requester to review by hand (DDT-915).
- **Lead routing.** DDT-915 noticed that SOS *leads* route through a separate lead queue. The requester said SOS leads belong with intake, so no routing-table change applies to them.
- **The account-owner defaults inside the flow** (see 10). None of these tickets changed them.

## 4. How it works (background)

Court Case is the relabelled `Opportunity`. Routing happens in one after-save flow. This description is read from the copy of the flow in the repo (last changed in git on 2026-07-03). **Production's active version was last modified on 2026-10-20**, so re-read the live flow before relying on any detail below.

```
Court Case saved
  -> runs only if Hire_Status__c = Hired and a Courthouse__c is set, and one of:
       a) the case is new (and Courthouse__c is not UNKNOWN)
       b) the case is Open and has no Court Case Manager
       c) the case is Open and its Courtroom__c changed
  -> look for a routing row, in this order, and use the FIRST that matches:
       1. same Case Type + same Courthouse            (no courtroom)
       2. same Case Type, no Courthouse               (e.g. Civil Asset Forfeiture, SOS types)
       3. same Courthouse + same Courtroom, Language = English, client is English
       4. same Courthouse + same Courtroom, Language = Spanish, client is Spanish
       5. same Courthouse, Language = English, client is English   (no courtroom, no case type)
       6. same Courthouse, Language = Spanish, client is Spanish   (no courtroom, no case type)
       7. any row for the same Courthouse                          (fallback, unordered)
  -> copy the row's Primary_Attorney__c and Court_Case_Manager__c onto the Court Case
  -> then set the client Account's owner (see 10)
```

Facts that matter for every reassignment:

- **The client language** on the case is `Case_Client_Language__c`, a formula: `TEXT(Account.Language__c)`. It is a text formula, so it cannot be used in `GROUP BY`.
- **A courthouse with one row serves every language.** On 2026-11-05 Rolling Meadows had a single English row; Spanish-speaking clients there reach it through step 7.
- **Case-type rows win over courthouse rows.** On 2026-11-05 production had 80 routing rows (47 English, 33 Spanish), seven of them case-type rows: Civil Asset Forfeiture, Order of Protection and five SOS types.
- **Courthouse values must match exactly.** The routing row's courthouse comes from the `Courthouse_List` global value set; the Court Case's courthouse is its own picklist. A Court Case courthouse with no matching value in `Courthouse_List` can never be routed.
- **Changing a routing row does not touch any existing case.** Only cases that hit trigger a), b) or c) afterwards pick it up. Open cases that already have a case manager keep the old people until someone updates them.

## 5. Root cause

Most of these tickets are planned reorganisations, not defects: nothing was broken, the firm changed who covers what. Where something did go wrong, these were the causes.

### 5.1 Routing changed, open cases not updated

Because routing only runs on new cases (or open cases with no case manager), changing a routing row leaves every open case where it was. DDT-939 is the example: four days after Rolling Meadows was moved to a new attorney (DDT-885), 72 open Rolling Meadows cases still showed the previous attorneys. Whether DDT-885 included a backfill is not recorded in the ticket; DDT-939 is the bulk update that brought those cases in line.

### 5.2 A courthouse row answered a case-type question

DDT-256: a Civil Asset Forfeiture case at Daley Center got the Daley Center attorney. The Slack exchange on the ticket establishes the cause: the courthouse's attorney was assigned because the case went through courthouse routing. The fix was described as "fixed the existing automation so this shouldn't happen again"; the exact change is not recorded in the ticket. Production today has case-type rows for both Civil Asset Forfeiture and Order of Protection, which is the shape the flow needs for a case type to override the courthouse (step 2 in section 4).

### 5.3 Counting the wrong population

DDT-700 first reported 812 SOS cases. That was every SOS case ever hired; the requester knew there were only about 80 active ones, and the correct figure was 89 open. The update itself had been limited correctly. The report was not. Count with the same filter you update with.

### 5.4 A re-run that changed more than it was asked to, and courthouses the table could not hold

DDT-1190's first pass (2026-10-02) moved Kane County cases to the wrong case manager and left Skokie Spanish speakers split between two case managers. The ticket records what was corrected, not exactly why the first pass went wrong. The correction on 2026-10-06:

- open cases at each listed courthouse re-set to the matrix; Kane cases returned to the right English and Spanish case managers; closed cases not touched;
- three courts (Frankfort, and two Chicago branch-court values) **could not be routed at all**, because they existed on Court Case but not in `Courthouse_List`, so no routing row could name them. PR #632 added the three values; the routing rows for them were added as data after it deployed.

The requester then answered three open questions (which case managers cover the Lake County branch courts, Skokie Spanish speakers, and the new Will County court), and an "All Done!" comment followed. The rows changed in that last pass are not listed in the ticket.

### 5.5 Two systems

DDT-41 changed the primary attorney in **DayBack**, not Salesforce. DDT-112, a week later, gave Maywood to a different attorney in Salesforce. The ticket raised the conflict before running; the outcome is not recorded.

## 6. Solution

Every request was handled the same way. The pieces below are the method; which pieces a ticket needed depended on whether it asked for future routing, existing cases, or both.

### 6.1 Change the routing table (future cases)

App Launcher → **Court Case Attorney Assignments**. Each row holds:

| Field | Meaning |
|---|---|
| `Courthouse__c` | From `Courthouse_List`. Blank only on a case-type-wide row |
| `Courtroom__c` | Text. Set only for a courtroom-specific rule (DDT-290's one Skokie courtroom) |
| `Case_Type__c` | Set only for a case-type rule (Civil Asset Forfeiture, Order of Protection, SOS types) |
| `Language__c` | English or Spanish |
| `Primary_Attorney__c`, `Court_Case_Manager__c` | The two people the case receives |

- **Edit the existing row; don't add a second one for the same courthouse and language.** Step 7 takes "any row" with no sort order, so duplicates make the result unpredictable.
- **Fill in both people on every row.** The flow copies both fields, so a blank attorney on a row blanks the attorney on the case.
- **To make a case type override every courthouse**, create a row with that `Case_Type__c` and no courthouse (DDT-256, DDT-915). DDT-915 found the SOS case-type rows already pointed to the incoming attorney, so only the existing cases needed moving.
- **A courthouse missing from the picklist** has to be added to `Courthouse_List` first, through a deploy (PR #632).

Giving a requester `Court_Case_Managment` lets them maintain this table themselves; that was the decision on DDT-1190.

### 6.2 Update the open cases (existing cases)

The scripts actually run for these tickets are not recorded. This is a template in the same shape, with the safeguards the tickets asked for: count first, snapshot before, open cases only, and a count after.

```apex
// Template: bulk-reassign open Court Cases at one courthouse.
// Run with DRY_RUN = true first and compare the count with the requester's expectation.
Boolean DRY_RUN = true;
String COURTHOUSE = '3rd - Rolling Meadows Courthouse';   // exact picklist value
Id NEW_ATTORNEY = null;      // look up the User by name first; never paste an Id from another org
Id NEW_CASE_MANAGER = null;  // leave null to keep the existing value

List<Opportunity> rows = [
    SELECT Id, Primary_Attorney__c, Court_Case_Manager__c
    FROM Opportunity
    WHERE Hire_Status__c = 'Hired' AND Case_Status__c = 'Open'
      AND Courthouse__c = :COURTHOUSE
    LIMIT 2000
];
System.debug(LoggingLevel.ERROR, 'open cases in scope: ' + rows.size());
// Snapshot: copy this line from the log before running for real; it is the only rollback.
for (Opportunity o : rows) {
    System.debug(LoggingLevel.ERROR, 'BEFORE,' + o.Id + ',' + o.Primary_Attorney__c + ',' + o.Court_Case_Manager__c);
}
if (!DRY_RUN) {
    for (Opportunity o : rows) {
        if (NEW_ATTORNEY != null) o.Primary_Attorney__c = NEW_ATTORNEY;
        if (NEW_CASE_MANAGER != null) o.Court_Case_Manager__c = NEW_CASE_MANAGER;
    }
    List<Database.SaveResult> srs = Database.update(rows, false);
    Integer ok = 0;
    for (Integer i = 0; i < srs.size(); i++) {
        if (srs[i].isSuccess()) { ok++; continue; }
        System.debug(LoggingLevel.ERROR, 'FAILED ' + rows[i].Id + ': ' + srs[i].getErrors()[0].getMessage());
    }
    System.debug(LoggingLevel.ERROR, 'updated ' + ok + ' of ' + rows.size());
}
```

Variants the tickets needed:

- **Only fill blanks** (DDT-529): add `AND Primary_Attorney__c = null`, so existing assignments are never overwritten.
- **Only one outgoing person** (DDT-915, DDT-939): add `AND Primary_Attorney__c = :OLD_ATTORNEY`.
- **By case type** (DDT-256, DDT-700, DDT-915): filter on `Case_Type__c IN (...)` instead of the courthouse.
- **By language** (DDT-937, DDT-1190): `Case_Client_Language__c` is a formula and can be filtered but not grouped. Check for spelling variants first, as DDT-937 asked.

### 6.3 Docket appearances

Upcoming Daily Docket appearances carry their own attorney, `Court_Appearance__c.Assigned_Attorney__c`, which a Court Case reassignment does not change.

- DDT-915 moved 10 upcoming appearances for the reassigned SOS cases to the incoming attorney.
- DDT-1279 cleared the outgoing attorney from all of that attorney's upcoming appearances (set to blank), so they could be handed out again.

Appearances on cases that are already closed were not moved; they were listed for the requester to review, since they may be re-hearings (DDT-915).

### 6.4 DayBack

DDT-41 was done in DayBack itself: every current and upcoming Maywood event had its primary attorney changed. How DayBack's primary attorney relates to the Salesforce fields is not recorded in the ticket.

## 7. Testing

No automated tests apply: the changes were routing rows and data, plus one value-set change (PR #632) with no Apex.

What the tickets record as checks:

| Ticket | Check |
|---|---|
| DDT-700 | Requester compared the reported count with her own; corrected to 89 open SOS cases |
| DDT-915 | 76 open SOS cases across 6 SOS case types, plus 10 upcoming appearances, reported back |
| DDT-939 | 72 open Rolling Meadows cases reported back (70 + 2 by previous attorney) |
| DDT-1190 | Requester spot-checked and found the first pass wrong; the second pass was reported back and the requester answered three follow-up questions |
| Others | "Done" with no recorded count |

**Spot check run on 2026-11-05**, to show the shape of a verification query:

```sql
SELECT Primary_Attorney__c, Court_Case_Manager__c, COUNT(Id) n
FROM Opportunity
WHERE Hire_Status__c = 'Hired' AND Case_Status__c = 'Open'
  AND Courthouse__c = '3rd - Rolling Meadows Courthouse'
GROUP BY Primary_Attorney__c, Court_Case_Manager__c
```

It returned two groups: **75** open cases with the same attorney and case manager as the Rolling Meadows routing row, and **1** with a different pair. One outlier is normal (a manual reassignment, for example) but is exactly what to look at after a bulk update.

**Not covered:** no ticket records creating a test case at each courthouse and language after the routing change, which DDT-1190's acceptance criteria asked for.

## 8. Step by step

1. **Get the rule in writing, unambiguously.** Which courthouses (exact picklist values), which languages, which case types, which role (Primary Attorney or Court Case Manager), and **AND or OR** when several criteria are listed. DDT-112 stopped to ask exactly this.
2. **Confirm scope**: open cases only, or more? Future cases too? Overwrite existing assignments, or fill blanks only?
3. **Check for conflicts with earlier requests**, including in other systems such as DayBack (DDT-41 against DDT-112).
4. **Look up the people** by name in the target org and confirm each user is active.
5. **Check every courthouse exists in `Courthouse_List`.** If one is missing, add it by a deploy first (PR #632).
6. **Edit the routing rows** (6.1). One row per courthouse and language; both people filled in.
7. **Count the open cases in scope** with the same filter you will update with, and send the count to the requester before changing anything (DDT-700).
8. **Run the bulk update** with a dry run first and a snapshot of the old values (6.2).
9. **Move or clear upcoming docket appearances** if asked (6.3). List any on closed cases instead of changing them.
10. **Re-run the count** grouped by attorney and case manager, and report the number changed in the ticket.
11. **Ask the requester to spot-check** a new case and an existing one.

## 9. Rollback

- **Routing rows:** edit them back. Only cases routed after the change are affected.
- **Bulk updates:** only reversible from a snapshot of the previous values. DDT-700 was described as reversible; none of the tickets records where a snapshot was kept. Take one every time (6.2).
- **Docket appearances cleared to blank** (DDT-1279) cannot be restored without a snapshot.
- **`Courthouse_List` values** (PR #632): leaving them is harmless; removing one that a routing row or a case uses is not.

## 10. Known limits

- **Existing cases never re-route on their own.** Every routing change needs its own bulk update. This is the single most common reason a requester comes back.
- **"Other" language clients.** Steps 3 to 6 only match clients whose language is English or Spanish. Anyone else falls to step 7, which takes any row for the courthouse with no ordering. Where a courthouse has separate English and Spanish rows, such a client may get either. DDT-1190 asked for "English and Other" to go to the English case manager; that is only guaranteed by a bulk check afterwards.
- **A courtroom change re-runs routing** on an open case and overwrites both people with the routing row's values, including any manual reassignment.
- **The account owner is set separately.** After routing, the flow sets the client Account's owner from a formula that, in the repo copy, hard-codes two case managers for Spanish-speaking clients at four Lake County courts and for Spanish SOS cases. Routing-row changes do not affect that formula; changing those people needs a flow change.
- **Two courthouse lists.** Court Case's own `Courthouse__c` picklist and `Courthouse_List` can drift apart. A value added to one and not the other can't be routed.
- **The repo copy of the flow is behind production.** Re-read the live flow before changing anything that depends on its exact order.

## 11. Reuse for similar issues

**How to recognise one:** "make X the attorney / case manager for courthouse Y", "cases at Y still show the old person", or "this case type should always go to Z".

- **Split every request into two halves:** future routing (rows in Court Case Attorney Assignments) and existing open cases (a bulk update). Ask which halves are wanted.
- **Case type before courthouse.** A rule that applies "regardless of jurisdiction" is a case-type row with no courthouse.
- **Count with the update's own filter** and report open cases, not all-time cases.
- **Snapshot before you write**, and update with `Database.update(rows, false)` so one failure doesn't hide others.
- **Check the picklist first.** A courthouse a routing row can't name can't be routed.
- **Don't forget the docket.** Appearances carry their own attorney.
- **When a requester says the result is wrong**, compare open cases grouped by attorney and case manager against the routing rows, courthouse by courthouse; the outliers are the cases to explain.
