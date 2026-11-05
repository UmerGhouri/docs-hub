# Trust reconciliation stopped being run after 1 September


| | |
|---|---|
| **Ticket** | DDT-1150 |
| **Project** | Trust Ledger |
| **Status** | Done 2026-10-12 (investigation; fixed by DDT-898) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-09-29 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Trust Tie-Out screen, read only) |
| **Area** | Client trust account: the reconciliation habit |
| **Related** | DDT-897, DDT-898, DDT-1312, DDT-1314 |
| **Code** | No code change |


---


## 1. Summary

The Trust Tie-Out reconciliation screen was live and finance staff had been trained on it, but no reconciliation was signed after 1 September. This ticket asked why, explicitly before building any alert. The answer: step 3 of the screen asked the user to type the date each deposit landed, which the bank file already held, and without those dates the walk could not be finished. DDT-898 removed that blocker. Firm leadership and finance staff ran a full reconciliation on the live screen on 2026-10-08 and it tied. No alert was built, and September was skipped rather than back-filled.

## 2. What was reported

- 2026-09-29: opened at firm leadership's request while reviewing DDT-897. Only two reconciliation records existed, the later dated 1 September; nothing for four weeks. Neither was dated a month end, so the monthly statement (DDT-897) could produce nothing. The gap was now longer than the one that prompted the screen in the first place.

## 3. Scope

**In scope**

- Finding out why reconciliation stopped: a blocker, an unworkable cadence, or nothing telling anyone it was overdue.

**Out of scope**

- Building an alert, deliberately: an alarm for a process nobody runs is a second thing to ignore, and the cause had not been asked yet.

## 4. How it works (background)

The Trust Tie-Out screen walks the reconciler through steps: enter bank balances, upload the bank CSV, confirm deposits (step 3), review payments, and sign off. Each sign-off saves a `Trust_Reconciliation__c` record. Step 3 blocks money going out on matters whose deposits are not yet confirmed. The screen shows a streak to whoever opens it, which only helps someone who opens it.

## 5. Root cause

Recorded by firm leadership on 2026-10-12: after 9/1, step 3 asked finance staff to type the landed date for each deposit even though the uploaded bank file already had it (the defect fixed in DDT-898). Without those dates the reconciliation could not be completed, so it stalled. Nobody skipped it on purpose. The evidence was the record count (two reconciliations, the last on 1 September) and the step 3 behaviour confirmed in code on DDT-898.

## 6. Solution

- No code in this ticket. DDT-898 made the bank file fill the landed dates (live 2026-09-28, with follow-ups live 2026-10-07).
- Firm leadership and finance staff ran the bank-file test and a full reconciliation together on the live screen on 2026-10-08; it tied and was signed. The screen fixes from that session became DDT-1312.
- No reminder or alert, because the cause was a blocker, not a missing reminder.
- September skipped: there is no reconciliation dated 9/30 and none was back-filled. DDT-1314 then changed the statement to work from any sign-off, so nobody has to reconcile on the last day of a month.

## 7. Testing

Not applicable: no code changed. The evidence of resolution is the tied reconciliation signed on 2026-10-08 and later sign-offs (13 October was used for a statement in DDT-1314).

## 8. Step by step

1. Count the reconciliation records and find the date of the last one.
2. Before building any reminder, sit with the person who runs the process and run it together.
3. Identify the step where they stop and why.
4. Fix the blocker (here, DDT-898).
5. Run a real reconciliation end to end and confirm it ties.
6. Only then decide whether an alert is still needed.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- Nothing tells anyone automatically if reconciliation stops again; the quarterly warning printed on statements (DDT-1314) only appears when a statement is issued.
- September 2026 has no month-end reconciliation and never will.

## 11. Reuse for similar issues

- Signs of the same problem: a working screen with no records for weeks. Ask what stops the user before adding reminders.
- A step that asks a person for data the system already holds is a likely place for a process to stall.
- A downstream feature that depends on a habit (the statement) is only as good as that habit; track the habit separately from the feature.
