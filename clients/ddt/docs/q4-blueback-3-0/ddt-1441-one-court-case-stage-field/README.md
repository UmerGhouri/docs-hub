# One stage field for the court case, on desktop and phone


| | |
|---|---|
| **Ticket** | DDT-1441 |
| **Project** | Q4: BlueBack 3.0 |
| **Status** | Live since 2026-11-03 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-27 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC, Lightning pages, Flow) |
| **Area** | Case management: court case stage tracking |
| **Related** | DDT-1452, DDT-1456, DDT-1454, DDT-1027 |
| **Code** | PR #839 and PR #845 (deploy record) in the ddt-salesforce repo |


---


## 1. Summary

The hired court case page had path components writing two different stage fields, and on desktop most defense case types (DUI included) showed no stage path at all, so their stage never moved past the first one. `Opportunity.Current_Stage__c` was chosen as the one stage field, because everything BlueBack runs on already reads it. The desktop page now shows the `courtCasePath` component for every hired case, and the phone page uses the same component instead of the one that wrote `Phase__c`. Deployed to production on 2026-10-28; the phone check was done 2026-11-03.

## 2. What was reported

- 2026-10-27: pick one stage field and say why; list everything that reads or writes either field; keep the other field in step during the change if needed; clean DUI values to the planner's stage names; retire the old path components once the new BlueBack 3.0 path is live. A read on 2026-10-20 showed 294 of 301 open hired DUI and Felony DUI cases at "Filings and documentation" on one field and 203 on the other, and neither was true.

## 3. Scope

**In scope**

- `courtCasePath` LWC: declared for phone and desktop; court-case stage clicks no longer need `Active_Plan_Builder__c`; phone layout; 3 new Jest tests.
- `Opportunity_Record_BB_Default_Hired` (desktop page): `courtCasePath` shown for Hire Status = Hired, with the case-type filter removed.
- `Opportunity_Hired_BB_Mobile_Only` (phone page): `customPath` replaced by `courtCasePath`.
- Deploy doc with the decision and the full read/write inventory.

**Out of scope**

- Mirror-writing `Phase__c`: not needed, since nothing depends on it except the old phone path and one report column.
- DUI value clean-up after the Resolution merge (DDT-1454).
- Retiring the old path components (DDT-1456) and the engine that moves the stage on its own (DDT-1452).
- Matters (PI): unchanged; their clicks still need the flag.

## 4. How it works (background)

A hired court case has a planner: a list of stages per case type. When the stage changes, the `Court_Case_Stage_Tracking` flow writes a `Stage_Tracking__c` entry row, and if Plan Builder is active on the case, the stage's tasks are created. `Phase__c` is a separate restricted picklist that depends on case type, with different wording and more steps (14 DUI phases against 9 DUI stages).

## 5. Root cause

Read from production on 2026-10-27:

| Page | Path component | Writes | Shown for |
|---|---|---|---|
| Desktop hired page | `courtCasePath` | `Current_Stage__c` | Hired and an SOS type or Aggravated Speeding only |
| Phone hired page | `customPath` | `Phase__c` | every record |

On desktop, DUI, Petty, Misdemeanor and similar cases had no path, so `Current_Stage__c` kept its value from hire: 299 of 307 open DUI and Felony DUI cases read "Filings and documentation". Phone users moved `Phase__c` instead. A third component in the repo wrote a field that no longer exists in production.

`Current_Stage__c` was chosen because task creation, the stage-change check, the BlueBack controller, the SLA timer, the task service, closing-stage clean-up, the Case Plan Builder and the stage-tracking flow all read it, and as a text field it takes the planner's stage names word for word. Only one report groups or filters by it; none by `Phase__c`.

## 6. Solution

- Desktop: the path shows for every hired case.
- Phone: the same path, with the action button stacked under it and narrower steps so more of the path fits.
- Court-case clicks save without the Plan Builder flag. The flag was on for only 1 of 297 open DUI cases, so without this change the newly shown DUI path would have refused every click. The flag still decides whether tasks are created.
- `Phase__c` is left as it is; two flows still set it in harmless cases.

Why no mirror-write: nothing that matters reads `Phase__c`.

## 7. Testing

- Jest: 3 tests (court-case click saves with the flag off; no auto-save with the flag off; Matter click still blocked). The click test fails against the old component.
- Partial sandbox: desktop click on a DUI case saved `Current_Stage__c`, wrote the Stage Tracking entry row and created the stage's task. Test data restored.
- Full sandbox: check-only and deploy 3/3. Desktop click saved `Current_Stage__c`, left `Phase__c` unchanged, wrote one tracking row and created three tasks. Restored afterwards.
- Production 2026-10-28: check-only passed (3/3 tests, 8/8 controller tests); the quick deploy was refused because four other deploys ran in between, so the same files were deployed directly. A retrieve afterwards confirmed the changes.
- Phone (Full sandbox, 2026-11-03): the path showed the right stage on a DUI case and on a PI matter, the same as desktop.
- Not covered: the first live DUI click with the flag off in production.

## 8. Step by step

1. List every page and component that writes each candidate field, and who sees it.
2. Inventory readers and writers with the dependency API, a report retrieve and the pages.
3. Choose the field most of the logic already reads.
4. Show one path component on every page and device for that field.
5. Remove any gate that would make the newly shown path refuse clicks, but keep gates on side effects such as task creation.
6. Test a click: stage saved, tracking row written, other field unchanged.
7. Test on a phone as well as desktop.

## 9. Rollback

Redeploy the two Lightning pages and the `courtCasePath` bundle from the commit before PR #839. Stage values written while live stay on the records; they are planner stage names.

## 10. Known limits

- Court cases never get Stage Tracking exit rows (only 6 of 598 recent rows have one). Left for DDT-1452.
- Case types with no planner rows show "No Stage Found".
- On the phone, a case switched on for BlueBack 3.0 still showed the old path with its "Mark Stage as Completed" button; the mobile page was then given the same switch as desktop and checked on a DUI test case in the Full sandbox on 2026-11-03. Cases whose type is not switched on keep the old mobile page.
- 18 hired cases with the flag on hold a stage name that does not match the planner exactly and would be rewritten on open; none is open.

## 11. Reuse for similar issues

- Signs of the same problem: a stage looks stuck on most records of some types, or desktop and phone users disagree. Check which field each page's path writes and for whom it is shown.
- Pattern: one source field, one component on every form factor, and an inventory of readers and writers in the deploy doc.
