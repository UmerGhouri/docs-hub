# Dochly-signed PI Matters saved with no record type

| | |
|---|---|
| **Ticket** | DDT-1377 |
| **Project** | Dochly |
| **Status** | Live since 2026-10-21 |
| **Priority** | High |
| **Reported by** | Firm operations, 2026-10-20 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex), Dochly e-signature |
| **Area** | PI intake: converting the Lead into a Matter once the contract is signed |
| **Related** | DDT-1070 (PI contract moved from DocuSign to Dochly), DDT-1287 (same root cause, different symptom) |
| **Code** | PR #774 in the `ddt-salesforce` repo |

---

## 1. Summary

After the PI contract moved from DocuSign to Dochly, every Matter created by a Dochly signing was saved with **no record type**. The code that builds the Matter never set one, so Salesforce used the running user's default. Since the move, that user is the Dochly signing site's guest user, whose profile has no default.

The fix sets the record type in code, so it no longer depends on who runs the conversion. A one-off script then repaired the 11 Matters that were already affected.

## 2. What was reported

From 2026-09-29 to 2026-10-19, **11 of the 17** PI Matters created had no record type. All 11 were created by the **Dochly Sign Site Guest User**. Every Matter a person created in that time had the `Matter` record type, and so did all 72 Matters created from June to 2026-09-28.

Why it matters: a record with no record type falls back to the object's master record type. Anything assigned per record type, such as page layouts, picklist values and record-type filters in reports and list views, treats these Matters differently from every other Matter.


## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `HiredMatterConvertController` | Apex class | `buildMatter` sets `RecordTypeId` explicitly |
| `HiredMatterConvertControllerTest` | Apex test | One new test: `buildMatter_SetsTheMatterRecordTypeExplicitly` |
| `ddt-1377-backfill-matter-record-type.apex` | Anonymous Apex script | One-off repair of Matters with no record type |

**Out of scope**

- **The guest user's profile (`Dochly Sign Profile`).** We did not give it a default Matter record type. Fixing the profile would fix this one user and leave the same trap for the next automated user that runs the conversion. Fixing the code removes the dependence entirely.
- **Other effects of the guest user running the conversion.** The guest user was also being given Matter tasks. That is DDT-1287, fixed separately.

## 4. How it works (background)

```
Client signs the PI contract on Dochly's public signing site
  -> Dochly marks the signature request completed
  -> the Lead -> Matter conversion runs                    (as the Dochly guest user, since DDT-1070)
  -> HiredMatterConvertController.buildMatter builds the Matter__c
  -> Matter__c is inserted
```

Before DDT-1070, the conversion ran as an internal user. Internal profiles default to the `Matter` record type, so leaving the field blank happened to work.

**The Salesforce rule behind this:** when Apex inserts a record without setting `RecordTypeId`, Salesforce uses the **running user's** default record type for that object. If the user has no record type available, the record gets none.

## 5. Root cause

1. `HiredMatterConvertController.buildMatter` never set `RecordTypeId`.
2. Since DDT-1070, the Dochly path runs the conversion as the **Dochly Sign Site Guest User**.
3. That user's profile, `Dochly Sign Profile`, has the `Matter__c.Matter` record type set to **visible: false, default: false** (read from production on 2026-10-19).
4. So every Matter it created got no record type.

**Evidence.** This query shows whose Matters are missing a record type. Before the fix, every row belonged to the guest user:

```sql
SELECT CreatedBy.Name, COUNT(Id)
FROM Matter__c
WHERE RecordTypeId = null
GROUP BY CreatedBy.Name
```

**The lesson.** The code didn't change. What changed was *who runs it*. Code that leaves a field blank and relies on the running user's defaults works only as long as every caller happens to have the right defaults.

## 6. Solution

### 6.1 Set the record type in code

```apex
private static final Id MATTER_RECORD_TYPE_ID = Schema.SObjectType.Matter__c
  .getRecordTypeInfosByDeveloperName()
  .get('Matter')
  .getRecordTypeId();

@TestVisible
private static Matter__c buildMatter(/* ... */) {
  Matter__c m = new Matter__c();
  // Set the record type explicitly. Left blank, Salesforce uses the running
  // user's profile default, and the Dochly Sign Site Guest User's profile has
  // none, so every Dochly-signed Matter was saved with no record type.
  m.RecordTypeId = MATTER_RECORD_TYPE_ID;
  // ...
}
```

Why it's done this way:

- **Looked up by developer name, never a hardcoded Id.** Record type Ids differ between production and every sandbox. The developer name `Matter` is the same everywhere.
- **Looked up once per transaction**, in a static constant, not for every Matter.
- **`@TestVisible`** so a test can call `buildMatter` directly (see 7).

### 6.2 Repair the Matters already affected

`Matter` is the only record type on `Matter__c`, so every blank one has exactly one correct value.

```apex
// Run AFTER the code fix is deployed, or new blanks keep arriving.
// Safe to re-run: it only touches rows that are still blank.
//   sf apex run -o <prod-alias> -f scripts/ddt-1377-backfill-matter-record-type.apex
Boolean DRY_RUN = false;   // set to true first to see the count without writing

Id rtId = Schema.SObjectType.Matter__c.getRecordTypeInfosByDeveloperName()
    .get('Matter').getRecordTypeId();

List<Matter__c> rows = [SELECT Id FROM Matter__c WHERE RecordTypeId = null];
System.debug(LoggingLevel.ERROR, 'Matters with no record type: ' + rows.size());

if (!DRY_RUN && !rows.isEmpty()) {
    for (Matter__c m : rows) m.RecordTypeId = rtId;
    Integer ok = 0;
    List<Database.SaveResult> srs = Database.update(rows, false);
    for (Integer i = 0; i < srs.size(); i++) {
        if (srs[i].isSuccess()) { ok++; continue; }
        System.debug(LoggingLevel.ERROR, 'FAILED ' + rows[i].Id + ': '
            + srs[i].getErrors()[0].getMessage());
    }
    System.debug(LoggingLevel.ERROR, 'updated ' + ok + ' of ' + rows.size());
}
System.debug(LoggingLevel.ERROR, 'still blank after run: '
    + [SELECT COUNT() FROM Matter__c WHERE RecordTypeId = null]);
```

Why it's written this way:

- **Dry run first**, so you see the count before anything is written.
- **`Database.update(rows, false)`** saves every row it can and reports each failure, so one bad record doesn't hide the others.
- **It counts again at the end**, so the script itself shows whether it worked.

## 7. Testing

**The Apex test.** `buildMatter_SetsTheMatterRecordTypeExplicitly` builds a Matter and checks the record type *before* it's inserted. At that point `RecordTypeId` is blank for every user unless `buildMatter` sets it, so the test proves the code sets it. **It fails with the fix removed**, which was checked against production.

**Why not test the real guest-user signing?** It can't run end to end in a test. A flow in the conversion tries to make the guest user an Account owner, and production avoids that through a site setting that tests don't get. So the first real Dochly signing after the deploy was the live check.

**Live check (2026-11-05):**

| Query | Result |
|---|---|
| `SELECT COUNT(Id) FROM Matter__c WHERE RecordTypeId = null` | **0** |
| Matters created by the Dochly guest user since 2026-10-21 with the `Matter` record type | **2**, and no Matters since then without one |

**Not covered:** an automated test of the full guest-user signing.

## 8. Step by step

The order matters: **deploy the code before running the repair**, or new blank Matters keep arriving behind it.

1. **Confirm the problem and who causes it.** Run the query in section 5. Every row should belong to the Dochly guest user.
2. **Confirm the guest profile has no default.** Setup → Profiles → `Dochly Sign Profile` → Object Settings → Matters → Record Types. `Matter` is not assigned.
3. **Change `buildMatter`** as in 6.1 and add the test from section 7.
4. **Validate against production** with the changed class's test, and confirm it passes.
5. **Get the pull request approved**, then deploy.
6. **Run the repair with `DRY_RUN = true`.** The count should match step 1 (it was 11).
7. **Run it with `DRY_RUN = false`.** The log should read `updated 11 of 11` and `still blank after run: 0`.
8. **Wait for the next real Dochly signing** and check that its Matter has the `Matter` record type.

## 9. Rollback

- **Code:** redeploy the previous version of `HiredMatterConvertController`. That brings the bug back, so there is no reason to do it unless the new line itself fails (see 10).
- **Repair script:** it doesn't record which rows it changed, so it can't be undone exactly. That's acceptable here because `Matter` is the only record type and there was no other correct value. **For a repair with more than one possible value, log the Ids first.**

## 10. Known limits

- **It always uses the `Matter` record type.** If a second Matter record type is ever added (one per practice area, say), every PI Matter will still get `Matter`. Revisit `buildMatter` then.
- **If the `Matter` record type is renamed or deleted, conversion stops entirely.** `.get('Matter')` returns null and the constant throws, so every conversion fails, not just Dochly ones. Rename it only together with a code change.
- **The guest user still runs the conversion.** Anything else that depends on the running user (record owner, default values, sharing) needs the same care. DDT-1287 was one such case.

## 11. Reuse for similar issues

**How to recognise one:** records created by an automated user (a site guest, an integration user, Automated Process, a scheduled job) look different from records a person created through the same process.

- **Group by `CreatedBy`** to find it fast: `SELECT CreatedBy.Name, COUNT(Id) FROM <object> WHERE <field> = null GROUP BY CreatedBy.Name`.
- **List what the code leaves blank**, and what Salesforce fills in from the running user instead: record type, owner, field defaults, sharing.
- **Set record types explicitly**, by developer name, never by Id.
- **When a process moves to a different running user**, check everything that depends on the running user before it goes live. DDT-1070 changed who ran the conversion and caused both this and DDT-1287.
- **When the real user can't be simulated in a test**, test the record *before* insert, where the running user's defaults haven't been applied yet.
- **Repair scripts:** dry run, touch only rows that are still wrong, report each failure, safe to re-run, and count again at the end.
