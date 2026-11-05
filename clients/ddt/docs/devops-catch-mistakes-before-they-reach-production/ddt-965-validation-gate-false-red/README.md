# Validation gate posted a failure for runs that did not fail


| | |
|---|---|
| **Ticket** | DDT-965 |
| **Project** | DevOps: catch mistakes before they reach production |
| **Status** | Done 2026-10-28 |
| **Priority** | High |
| **Reported by** | Salesforce lead, 2026-09-09 |
| **Resolved by** | Support team |
| **Systems** | Python script, GitHub status checks, Salesforce (Metadata API deploy status) |
| **Area** | Deploy process: required `salesforce/validation` check |
| **Related** | DDT-747, DDT-817, DDT-825, DDT-967 |
| **Code** | PR #844 in the ddt-salesforce repo |


---


## 1. Summary

`post-validation.py` posts the required `salesforce/validation` status on a pull request. It posted "failure" for validation runs that never judged the code: one that never started, one that was cancelled with no errors, and one that succeeded 46 of 46 but was superseded by another deploy. A red required check tells the author their code is broken. The script now decides from the deploy's real final state: only a real test or component error posts red; a superseded, cancelled or unfinished run posts nothing and says to run it again.

## 2. What was reported

- 2026-09-09: on one pull request on 2026-09-08, the gate posted failure four times. The clearest case named a job that had succeeded with 46 of 46 tests and 0 errors, printed "superseded, re-validate", and then posted red anyway. This contradicts the script's own documented contract: exit 0 posts success, exit 1 posts failure, exit 2 means it could not do its job and posts nothing.
- 2026-09-09 (comment): a second mode. The CLI's wait ran out and it reported a job as "Canceled" while the org still had it queued as Pending. Reading "Canceled" invites an immediate retry, which queues a second job and makes the queue worse.

## 3. Scope

**In scope**

- `scripts/post-validation.py`: one function, `classify()`, now gives the verdict.
- `scripts/test_post_validation.py`: 11 new cases.
- `docs/ci/RULE-17-VALIDATION-GATE.md`: the exit-code line lists the "post nothing" cases.

**Out of scope**

- How supersession is detected: unchanged.
- Showing who holds the org, and an automatic single retry on supersession: suggested in the ticket, not built.
- No Salesforce metadata changed, so nothing was deployed.

## 4. How it works (background)

A validation is a check-only deploy against production. Production runs one deploy at a time; others wait in a queue. A real deploy cancels queued and running validations, and any new real deploy makes an older validation result unusable. On a working day several people and tools validate against production, so supersession is normal here, not rare. A full-suite run is about 20 minutes of exposure; even a 2-minute run with named tests was superseded.

## 5. Root cause

The script decided from "did I get a green result" instead of from the deploy's real final state. Anything that was not a clean pass, including Pending, Canceled with zero errors, and a superseded success, was posted as failure. The same shape caused the second mode: the tool drew a conclusion from its own timeout instead of reading `DeployRequest.Status`.

Evidence from the ticket:

| Real state of the job | What the script posted |
| -- | -- |
| Pending, never started | failure, "0 tests, 0 failed, 0/0 components" |
| Canceled partway, 0 errors | failure |
| Succeeded, 46 of 46, 0 errors (superseded) | failure |

## 6. Solution

The verdict now comes from one function:

- Succeeded and not superseded → post success.
- Failed with test errors or component errors → post failure. This is the only red.
- Superseded, Canceled, still Pending or InProgress when the wait runs out, Failed with no errors reported, or no status → exit 2, post nothing, and tell the user to run it again.

Why: a missing status still blocks the merge, honestly, and says "run it again". A red status sends the author looking for a bug that does not exist.

## 7. Testing

- 11 new test cases, including the three real jobs from the reported pull request. The test file passes locally, and CI was green on PR #844.
- Not covered: the closing comment says the ticket would be confirmed by the next real gate run; that run is not recorded in the ticket.

## 8. Step by step

1. When a required check goes red, look up the deploy's real state in the org, not only the CLI output.
2. List every final state a deploy can reach, plus "still waiting" and "client error".
3. Map each one to success, failure or "post nothing"; only real test or component errors are failures.
4. Put the mapping in one function and test each state, using real cases.
5. Update the documented exit codes to match.

## 9. Rollback

Revert PR #844. The gate then posts false failures again for superseded or unfinished runs. Nothing in Salesforce is involved.

## 10. Known limits

- Supersession still happens; the fix stops it being reported as a code failure, but the run must still be repeated.
- The gate does not show which other deploy holds the org.
- Using named tests (a few minutes) instead of the full suite shortens the window for supersession.

## 11. Reuse for similar issues

- Signs of the same problem: a required check is red but the deploy job in Setup shows Succeeded or Canceled with zero errors.
- A check should go red only when it has proven a real problem. When it could not do its job, it should post nothing and say why.
- Read the system's real status instead of inferring it from your own timeout.
