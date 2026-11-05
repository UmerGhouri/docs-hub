# Decision: score SOS calls on the same rubric


| | |
|---|---|
| **Ticket** | DDT-880 |
| **Project** | AI Call Scorer |
| **Status** | Done 2026-09-21 (decision; nothing deployed) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-31 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (consult score data) |
| **Area** | Intake: sales consult call coaching, rubric design |
| **Related** | DDT-879, DDT-1056, DDT-1061 |
| **Code** | No code change |


---


## 1. Summary

The question was whether calls about Secretary of State (SOS) licence hearings should be scored on the same rubric as criminal defense calls, since five of the eleven rubric sections behave differently for SOS. It was parked until there was evidence. The planned 30-day backfill was declined, but production soon held 275 scored calls, 37 of them SOS. SOS and defense calls failed at almost the same rate, so firm leadership decided on 2026-09-21 to keep one rubric and set two triggers for revisiting.

## 2. What was reported

- 2026-08-31: record the open question so it is not quietly decided by whoever next edits the rubric. Settle it from scored data, not from a hunch.
- 2026-09-21: the backfill this ticket was waiting on (DDT-879) was declined on 2026-09-20; check whether live scores were enough to decide.

## 3. Scope

**In scope**

- Comparing the stored band of all current scored runs, split by case-type family.

**Out of scope**

- Building a second rubric or changing any points.

## 4. How it works (background)

Five sections differ for SOS calls: the qualifier asks about the licence revocation rather than eligibility; a specific success-rate claim is approved only for SOS hearings; an SOS call once scored full marks on the pitch using none of the defense pillars; the hire rate is 20% for SOS against 45% for defense; and the quote conversation is shaped differently. Splitting the rubric would mean a second rubric to calibrate, maintain and dispute.

## 5. Root cause

Not applicable: a design decision. The evidence, from 275 current scored runs (2026-09-08 to 2026-09-21):

| Family | Runs | Strong | Adequate | Failing | Adequate or better |
|---|---|---|---|---|---|
| Defense and other | 238 | 2 | 44 | 192 | 19.3% |
| SOS | 37 | 0 | 6 | 31 | 16.2% |
| All | 275 | 2 | 50 | 223 | 18.9% |

SOS case types counted: formal hearing, informal hearing, abstract research, RRR and out-of-state hearing.

## 6. Solution

Decision (firm leadership, 2026-09-21): keep one rubric; do not build a second.

Why: if the defense rubric were punishing SOS calls at any meaningful size, SOS would sit far below defense. It sat about 3 points below, and defense was failing at about four in five too. The harshness is across the whole rubric, not specific to SOS. With only 37 SOS runs and no significance test, this supports "do not split now", not "SOS and defense behave identically".

Revisit triggers:

1. SOS volume passes about 150 scored calls and the gap widens.
2. SOS reps' per-section disputes cluster on the five sections listed in section 4.

## 7. Testing

Not applicable: no change was made. The figures came from a read of production data on 2026-09-21; no significance test was run.

## 8. Step by step

1. Split the stored outcomes by the group in question and compare the share at Adequate or better.
2. Compare the gap with the overall failure rate before blaming the group.
3. If the sample is small, decide "not now" and write down specific revisit triggers.

## 9. Rollback

Not applicable: no change was made.

## 10. Known limits

- 37 SOS calls is a small sample; a 3-point gap is within noise.
- The comparison uses the overall band, not section scores, which were not yet reportable (DDT-1061).

## 11. Reuse for similar issues

- When a single scoring model seems unfair to one group, compare that group with the rest before building a separate model; a high failure rate everywhere points at the model, not the group.
- Use people's disputes as evidence: clustered disputes on the same sections are a stronger signal than a small score gap.
