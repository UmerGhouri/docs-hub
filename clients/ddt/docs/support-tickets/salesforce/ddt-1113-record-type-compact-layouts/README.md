# Record-type deploys reset compact layouts

| | |
|---|---|
| **Ticket** | DDT-1113 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-25 (CI guard and deploy policy; nothing deployed to Salesforce) |
| **Priority** | High |
| **Reported by** | Support team, 2026-09-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (record types, compact layouts), CI (`scripts/compact-layout-guard.py`) |
| **Area** | Deploys that include record type metadata |
| **Related** | PR #450 and PR #467 (the deploys that caused the resets), PR #478 (the restore), DDT-1069 |
| **Code** | PR #491 in the ddt-salesforce repo; PR #498 (proof run, closed unmerged) |

---

## 1. Summary

Twice in one week, a green deploy that included record type files reset five record types' compact layouts to **Inherit from Primary**, and users saw the wrong highlights panel. A record type file that leaves out `compactLayoutAssignment` means "no assignment", not "leave it alone". We added a blocking CI check that fails any PR whose record type file would reset its compact layout, plus a post-deploy check in the deploy policy.

## 2. What was reported

- **2026-09-22:** PR #450 deployed record type files retrieved from production. Five record types lost their compact layout: Lead Defense, Lead PI, Lead SOS, Matter, and Person Account.
- **2026-09-24:** PR #467 reset the Matter record type again.

Both deploys were green. The support team found the changes in Setup Audit Trail and restored all five in PR #478 the same day. Until then users saw the wrong highlights panel on those records.

It was older than these two deploys: Setup Audit Trail held **19** compact layout mapping changes back to its retention limit (2026-03-28), and the Matter record type accounted for five of them.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `scripts/compact-layout-guard.py` | CI script | New. Four repo-only checks and an optional org audit mode |
| `scripts/data/compact-layout-inherit-baseline.txt` | Data file | New. The record types that correctly inherit from primary |
| `scripts/test_compact_layout_guard.py` | Tests | New. 29 checks |
| `.github/workflows/ci.yml` | CI | Guard and tests run in the blocking `python-tests` leg |
| `docs/DEPLOY-POLICY.md` | Policy | New "Record types and compact layouts" section with a post-deploy query |

**Out of scope**

- **Record type metadata files.** The audit found all 80 record types on `main` already matched production (PR #478 had fixed them), so none changed.
- **Giving the other two guard jobs blocking status.** Noted as a separate question.

## 4. How it works (background)

```
Record type file in git (.recordType-meta.xml)
  -> deployed to production
  -> Salesforce sets the record type's compact layout to whatever the file says
     (element present: that layout; element missing: Inherit from Primary)
```

The compact layout drives the highlights panel at the top of a record page. Deploys, validation and tests do not check it, so a reset only shows up as the wrong panel.

## 5. Root cause

`compactLayoutAssignment` in a record type file is **absolute, not additive**. A file that omits it does not leave the org's assignment alone; deploying it sets the record type back to Inherit from Primary.

The five files never carried the element before PR #478. The assignments had been made in Setup and were never retrieved into git, so every deploy of those files undid them.

**Three of the ticket's premises were checked and did not hold:**

1. **"Every record type file must carry the element."** In production, **50 of the org's 80 record types** have no assignment and are correct that way. A blanket rule would block every PR touching one of them.
2. **"The CustomObject retrieve route drops the element."** Not reproducible on 2026-09-25. Retrieving the Lead object and retrieving the record type on its own both returned the assignment. So the guard checks the file, not how it was retrieved.
3. **"Audit every record type and fix the files."** Already done by PR #478. All 80 record types on `main` matched a fresh production retrieve.

## 6. Solution

### 6.1 The CI guard

`scripts/compact-layout-guard.py` enforces: **a record type file must carry `compactLayoutAssignment`, or be listed on purpose in `scripts/data/compact-layout-inherit-baseline.txt`.**

Its four repo-only checks:

| Check | Fails when |
|---|---|
| `REMOVED` | A PR drops the element from a file that had it |
| `MISSING` | A file has no element and is not on the baseline |
| `UNKNOWN` | The named compact layout does not exist under the record type's own object |
| `STALE` | A record type is on the baseline but carries an assignment, so the list cannot rot |

It runs over the whole tree in CI (`--all`), which needs no credential. `--base-ref` gives the more precise `REMOVED` wording when a merge base is available. `--audit-org` compares the tree against a live org; it is not a CI job because no CI job holds a production credential.

**Why inside `python-tests`:** `quality` is the required check, and it depends on `python-tests`. Running the guard there makes it block merges without a repository settings change.

### 6.2 The deploy policy

`docs/DEPLOY-POLICY.md` gained a section saying:

- Retrieve production's current record type and diff the element before deploying one.
- **Person Account record types only accept a PersonAccount-type compact layout** (`Client_Compact_Layout`). An Account-type layout is rejected even if its name suggests Person Account.
- After any record type deploy, query Setup Audit Trail for compact layout mapping changes.

Two corrections were needed for the post-deploy query to work. `SetupAuditTrail.Display` cannot be filtered in SOQL, so filter on `Action`. And only resets to Inherit from Primary are ever logged, so an empty result never proves a restore worked.

```sql
SELECT CreatedDate, CreatedBy.Name, Section, Display
FROM SetupAuditTrail
WHERE Action = 'updatedCompactLayoutMapping'
ORDER BY CreatedDate DESC
LIMIT 20
```

## 7. Testing

- **`scripts/test_compact_layout_guard.py`: 29 of 29 passing**, including one run over the repo's real 80 record types.
- **Run against the real commits:**

| Commit | Record types touched | Guard result |
|---|---|---|
| PR #450 | 8 | Fails, naming all five that were reset; passes the three that carried assignments |
| PR #467 | 1 | Fails, naming the Matter record type |
| PR #478 (restore) | 5 | Passes |
| DDT-1069 commit | 2 | Passes; both correctly inherit |
| `main`, whole tree | 80 | Passes |

- **Proof PR #498** stripped `compactLayoutAssignment` from the Lead Defense record type. `python-tests` and `quality` went red naming it. It was closed unmerged.

**Not covered:** a record type on the baseline can still be deployed with no assignment, by design. If its assignment is set in Setup later and not retrieved, the guard cannot know.

## 8. Step by step

1. **Before deploying a record type**, retrieve production's current copy: `sf project retrieve start -m RecordType:<Object>.<Name> -o <prod-alias>`.
2. **Diff `compactLayoutAssignment`** between that copy and your file. Keep production's value.
3. **Push the PR.** If the guard fails with `MISSING` or `REMOVED`, add the element back. Add the record type to the baseline file only if production really has no assignment.
4. **Within ten minutes of deploying**, run the Setup Audit Trail query in 6.2. A row whose `Display` says the layout changed to Inherit from Primary at your deploy's time is a reset your deploy caused.
5. **If a reset happened**, restore the assignment in production **and** fix the file in git, or the next deploy of that file resets it again. Confirm the restore by retrieving the record type, not from the audit trail.
6. **To audit the whole tree**, run `python scripts/compact-layout-guard.py --audit-org <prod-alias>`.

## 9. Rollback

Remove the two guard steps from `python-tests` in `.github/workflows/ci.yml`. The scripts and docs have no runtime effect and can stay. Removing the guard brings back the risk of silent resets.

## 10. Known limits

- **The baseline must be maintained.** A record type that gains a compact layout in Setup trips `STALE` until it is removed from the baseline and its file is updated.
- **The guard reads files, not the org.** If an assignment is changed in Setup and not retrieved into git, only `--audit-org` will find it.
- **Setup Audit Trail logs resets only**, and keeps about six months.

## 11. Reuse for similar issues

**How to recognise one:** after a green deploy, something configured in Setup quietly goes back to a default, and nothing in the deploy output mentions it.

- **Some metadata elements are absolute**: leaving one out of a file sets it to the default. Check whether the element you are deploying behaves that way.
- **Measure the ticket's rule against production before enforcing it.** Here half the record types correctly had no assignment.
- **Check the file, not the retrieve route.** Routes can change; the deployed file is what matters.
- **Prove a guard on the commit that caused the incident**, and on one that should pass.
- **Put a new guard where it actually blocks merges**, not in a job nothing depends on.
