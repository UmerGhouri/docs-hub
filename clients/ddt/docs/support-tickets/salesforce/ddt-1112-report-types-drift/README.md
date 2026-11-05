# Report types in git out of step with production

| | |
|---|---|
| **Ticket** | DDT-1112, DDT-1162 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-02 (git and tooling only; nothing deployed to production) |
| **Priority** | Medium (DDT-1112), Low (DDT-1162) |
| **Reported by** | Support team, 2026-09-24 (found while building other reports) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (custom report types), repo tooling (`scripts/drift-check.py`) |
| **Area** | Reporting metadata and the drift check between production and git |
| **Related** | DDT-963 (worked around the stale file), DDT-1026 (PR #571 hit the same problem on one report type) |
| **Code** | PR #493 and PR #590 in the ddt-salesforce repo |

---

## 1. Summary

The custom report type files in git had fallen out of step with production. Most of the difference was columns for Opportunity fields that production had already deleted, so any deploy of one of those files would fail. PR #493 brought the `Opportunities` report type in line with production and taught `drift-check.py` to compare report types. PR #590 then brought 24 more report types in line, one file at a time. Nothing was deployed: production already held the correct version.

## 2. What was reported

- **DDT-1112, 2026-09-24.** While building new reports, the support team found that the `Opportunities` report type in git did not match production. The ticket assumed git was *behind* (490 columns in git, 568 in production) and that deploying git's copy would silently remove 78 columns from production. The work on DDT-963 had to create a separate report type to avoid touching the file.
- **DDT-1162, 2026-09-30.** The automated review on PR #571 (DDT-1026) found 52 dead columns on one report type. Measured against production the same day: **23 report types** in git still listed columns for **96 distinct Opportunity fields** that no longer exist in production, all removed by earlier Opportunity field clean-ups. Counts per file ranged from 5 to 96 dead columns.

Nothing was broken in production. The risk was in git: deploying any of these files would fail on the first missing field.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Opportunities.reportType-meta.xml` | Custom report type | Replaced with production's copy (PR #493, then 14 more dead columns removed in PR #590) |
| 23 more `*.reportType-meta.xml` files under `force-app/main/default/reportTypes/` | Custom report types | Each replaced with production's copy (PR #590) |
| `Court_Cases_ONLY.reportType-meta.xml` | Custom report type | One remaining dead column removed (PR #590; not in the ticket's list of 23) |
| `scripts/drift-check.py`, `scripts/test_drift_check.py` | Repo tooling | Drift check now compares custom report types as well as Apex |
| `CLAUDE.md`, `docs/ci/REPO-HYGIENE.md` | Repo docs | Drift-check scope statement corrected |

**Out of scope**

- **Deploying anything.** Production already matched the files being written into git.
- **A bulk sweep of every drifting report type.** The project rules forbid copying production over git in one sweep. Each file was retrieved, diffed and read on its own.
- **Report types that exist only in production.** The first drift run found 5; they were reported on the ticket, not added.

## 4. How it works (background)

A custom report type file lists every column a report built on it may use. Deploying one **replaces the whole type**, so every column it names must exist in the target org.

```
Field added or deleted in Setup on production
  -> production's report type changes
  -> the file in git is not retrieved again
  -> git and production drift apart, silently
```

Before this change, `drift-check.py` compared only Apex classes and triggers, so report type drift was invisible.

## 5. Root cause

**The ticket's first guess had the direction backwards.** DDT-1112 said production had more columns than git. Measured on 2026-09-25 it was the other way round: production had **490** columns and git had **568**. The 82 columns that existed only in git all named custom fields production had already deleted (0 of the 82 were live in `FieldDefinition`). Only 4 columns existed in production but not in git.

So a PR touching the file would not silently delete columns from production. It would fail outright, with an error of the form `In field: field - no CustomField named Opportunity.<field> found`. That is why the file was a blocker: two earlier pieces of work had hit it and routed around it.

DDT-1162 was the same cause across more files: Opportunity fields were deleted in production, and the report type files in git were never refreshed.

**How DDT-1162 was measured:** every Opportunity field in production was read from `FieldDefinition` (595 fields). Describe calls were not used because they hide fields the running user cannot see. Each report type on `main` was compared against that list, counting only columns whose table is the Opportunity.

## 6. Solution

### 6.1 Bring each file in line with production, one at a time

For each report type:

1. Retrieve it from production.
2. Diff it against `main` and read the diff.
3. Replace `main`'s copy with production's.

PR #590 made one commit per file, and each commit message records what was removed and anything that was added. Results:

- **97 dead Opportunity columns removed** across the files.
- **22 files also gained live columns** that production had and git did not (for example three live Opportunity fields on several Court Case report types). The ticket expected removals only and said to stop on any addition; these were written up per file in the commit messages instead of being dropped.
- `Opportunities` lost a further 14 dead columns; `Court_Cases_ONLY` lost one.

### 6.2 Make the drift check cover report types

`drift-check.py` now retrieves custom report types from production into a temporary directory (never into the working tree) and compares them with git. The Tooling API has no queryable body for a report type, which is why a Metadata API retrieve is used.

Two ways the check could report clean while comparing nothing were designed against and pinned by tests:

- Deriving the component name the way the Apex path does would put every report type into a bucket that does not count against a clean exit.
- A retrieve that succeeds but returns nothing would do the same.

Both now exit with code 2. The script prints an agreement count per metadata type so a reader can see report types were actually compared. `--skip-report-types` gives the faster Apex-only run.

**First run after the change:** 598 components agreed (518 classes, 44 report types, 36 triggers), 65 disagreed, of which 57 were report types, and 5 report types were live in production with no record in git. DDT-1162 then cleared 24 of those report types.

## 7. Testing

- **`scripts/test_drift_check.py`** covers the report type comparison, including the two silent-skip cases above, which must exit 2.
- **PR #590:** each file's diff was read and recorded in its commit message. Nothing was deployed.
- **Live check:** the first drift run (section 6.2) showed report types being compared and counted.

**Not covered:** `SSS_End_Date_for_flow_del__c` is still listed in 7 report type files. It is a field in the recycle bin and production's own copies list it too, so git cannot drop it without disagreeing with production.

## 8. Step by step

1. **Run the drift check:** `python scripts/drift-check.py` (on this machine `python` rather than `python3`). Note which report types disagree.
2. **For one report type**, retrieve it from production with the Metadata API into a disposable location.
3. **Diff it against `main`.** Expect removals of columns for deleted fields. For any column production has that git does not, confirm the field is live and say so in the commit message.
4. **Replace `main`'s copy** with production's. One file per commit.
5. **Repeat** for each report type. Never copy the whole folder in one go.
6. **Open the PR against `main`.** State per file what the diff contained, and that nothing is deployed.
7. **Re-run the drift check** after merge and confirm those report types now agree.

## 9. Rollback

Revert the commit for the file concerned. That puts back a version production does not have, so there is no reason to do it. Nothing in production changed, so there is nothing to undo there. The drift-check change can be bypassed with `--skip-report-types`.

## 10. Known limits

- **Report types drift again whenever someone edits one in Setup** or deletes a field it lists. The drift check finds it, but only when someone runs it; it is not a CI job because it needs a production credential.
- **The drift check covers Apex classes, triggers and custom report types only.** A clean report does not mean every metadata type agrees.
- **Recycle-bin fields** such as `SSS_End_Date_for_flow_del__c` stay in the files until they are purged in production.

## 11. Reuse for similar issues

**How to recognise one:** a deploy fails with `no CustomField named ... found` on a file you did not mean to change, or a reviewer finds columns for fields that no longer exist.

- **Measure the direction before acting.** Count both sides. The ticket's assumption that git was behind would have led to the wrong fix.
- **Use `FieldDefinition` to list fields**, not describe, which hides fields the running user cannot see.
- **Sync production into git one file at a time** and read each diff. Never sweep a whole folder.
- **Write down additions as well as removals.** A sync that is expected to be removals-only often is not.
- **When you extend a check, make it fail loudly** if it compares nothing.
