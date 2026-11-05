# New credit rules for new records only: schema, door flag and rule version


| | |
|---|---|
| **Ticket** | DDT-824 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-16 |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, custom metadata, custom fields) |
| **Area** | Attribution: the credit rule on Court Case and Lead |
| **Related** | DDT-574, DDT-813, DDT-816, DDT-822, DDT-825, DDT-826, DDT-1026, DDT-1027 |
| **Code** | PR #404 in the ddt-salesforce repo (the engine wiring, PR #406, was finished under DDT-1026) |


---


## 1. Summary

The attribution rebuild changes how credit is decided, but the firm did not want already-credited records to change on go-live day. This ticket set the rule "forward only": new records get the new rules, existing ones keep theirs, and a rule-version field on Opportunity and Lead makes the two eras distinguishable in any report. It also settled a contradiction in the old rules: a "door" (a main phone line or office walk-in campaign) should not win credit on its own. The ticket was split. Part 1 (schema, door flags, configuration and two selector classes, PR #404) is live with the new behaviour switched off. Part 2 (wiring the engine and switching the door rule on) moved to DDT-1026.

## 2. What was reported

- 2026-08-24 (DDT-824): the credit rule, forward only. Add `Attribution_Rule_Version__c` on Opportunity and Lead; make the engine read the touch log; stamp first / last / assist flags at conversion; add returning-client recency; add a cold-client window in `Attribution_Config__mdt`; fix an unbounded query and a `LIMIT 10000` bug; settle the mainline contradiction before writing code. Done when a new Court Case is credited under the new rules with a version, an old one is untouched, and switching only the configuration from First Touch to Last Touch changes the answer.
- 2026-09-11: input that an "Excluded" token (manual-entry campaigns) still set the channel, hiding 229 real answers.
- 2026-09-16: firm leadership's decision that a door stops winning on its own; later the same day, the split into part 1 and DDT-1026.

## 3. Scope

**In scope (part 1, PR #404)**

- `Channel_Mapping__mdt.Is_Door__c`, set true on the seven door rows (the four Mainline campaigns and the three office campaigns).
- `Attribution_Config__mdt`: `Door_Never_Wins_Alone__c` (shipped false), `Attribution_Model__c` (Last Touch), `Cold_Client_Window_Days__c` (1825; returning gap stays 30 days).
- `Attribution_Rule_Version__c` on Opportunity and Lead.
- `AttributionTokenSelector` and `AttributionTouchSelector` with tests; related changes in `AttributionCreditRunner`, `AttributionCreditService` and its test.

**Out of scope**

- Engine wiring, the conversion-time touch stamper, recency fields and the done-when proof: DDT-1026.
- Re-scoring the 40,950 already-credited records: DDT-826.
- Crediting Matters, which would need six new fields and a third runner path.

## 4. How it works (background)

There are two chained engines. `ReferralSourceService` writes the referrer name and type onto Lead, Court Case, Matter and Client. `AttributionCreditService` with `AttributionCreditRunner` and `AttributionCreditBatch` writes the six attribution fields on Court Case and Lead, picking a token as the first non-blank of Campaign name, automated source and how-heard answer, then resolving it through `Channel_Mapping__mdt`. The hierarchy is Returning, then Referral, then Channel. `Attributed_As_Of__c` already dates each answer, which is what makes forward-only possible.

## 5. Root cause

Two rule defects were measured in production before code was written:

- **Doors won credit alone.** `AttributionCreditService.patternBucket` mapped Mainline and office tokens to Direct / Phone (and a test asserted it), while the design's rule 1 said a door never wins alone. 3,339 of the 3,433 records credited Direct / Phone came from a door campaign. Of those, 1,067 carried a real answer behind the door (144 of them a referral or returning answer, so in the wrong tier), and 2,126 had nothing else and would become Unknown. Two figures in a first count (1,213 and 190) were overstated and were corrected the same day.
- **Excluded tokens still set the channel.** `resolve()` fixes the channel from the token before it checks whether the token is marked Excluded. 437 records carried a manual-entry automated source; 314 were Unknown; 229 of those had a how-heard answer that was never read.

Both come from token selection, not resolution: the runner discards the other candidates before `resolve()` runs.

## 6. Solution

- **Forward only.** Existing records keep their answers; the restricted picklist value Returning stays, so 1,934 records holding it are not broken. The new rules apply to new records.
- **Rule version field.** Stamped by the engine on every write, so a report comparing months can tell a rule change from a trend.
- **One mechanism for both defects:** token-selection fall-through. If the chosen token is a door or Excluded, move to the next candidate in precedence; if nothing is left, the record is Unknown. Firm leadership accepted the cost explicitly: a confident Direct / Phone becoming an honest Unknown, consistent with the position that a fabricated answer is worse than a blank one.
- **The door flag lives on `Channel_Mapping__mdt`,** where the door rows are already declared and where `Excluded` is the precedent for "not an answer". The registry's type field is the telephony vendor and the wrong home.
- **The switch ships off.** Part 1 adds schema, flags and selectors with no behaviour change; flipping `Door_Never_Wins_Alone__c` belongs to part 2.
- **Model choice is a flag on the touch row,** not code that orders by date, so First, Last and weighted credit read from one table (fields from DDT-813).

## 7. Testing

- Selector tests in PR #404 (654 lines across the two test classes).
- Read back from production on 2026-09-16: `Is_Door__c` true on all seven door rows; configuration values as above with the door switch false; `Attribution_Rule_Version__c` present on both objects; both selectors live; the touch stamper correctly not yet deployed.
- The done-when (new versus old record, First versus Last Touch from configuration alone) could not be proven until the engine was wired, so it moved with part 2 to DDT-1026.

## 8. Step by step

1. Measure the impact of each rule change in production before deciding (who moves, and to what).
2. Get the decision recorded on the ticket, including the cost accepted.
3. Add a rule-version field before any rule change ships.
4. Put "this token is not an answer" flags in deployable configuration, not code.
5. Ship schema and flags with the behaviour switch off; switch on in a separate, verified step.
6. Rewrite tests that assert the old rule; do not weaken them.

## 9. Rollback

Part 1 changed no behaviour: with `Door_Never_Wins_Alone__c` false, the engine gives the old answers. Rolling back the switch is a configuration change. Removing the new fields would need a destructive deploy. A rollback was not recorded in the ticket.

## 10. Known limits

- 94 Direct / Phone records that are not door-sourced were not examined.
- The model switch, recency and the rule's done-when depend on DDT-1026.
- Merging PR #404 closed the ticket automatically while part 2 was open, the third time in the project; this became DDT-1027.

## 11. Reuse for similar issues

- When rules change, version the output so old and new answers can be told apart.
- Prefer forward-only changes plus a separate, measured re-score over rewriting history on go-live day.
- When two sources of truth contradict (a rule and a test), measure the population on each side and have the decision made explicitly.
- If a value is "not an answer", let selection fall through to the next candidate instead of special-casing it later.
- Split multi-PR work into separate tickets, or keep ticket ids off non-final branches, so merges do not close tickets early.
