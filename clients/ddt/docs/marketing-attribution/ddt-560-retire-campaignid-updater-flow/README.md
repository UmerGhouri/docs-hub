# Retire a flow that kept its own hidden campaign table


| | |
|---|---|
| **Ticket** | DDT-560 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-22 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow) |
| **Area** | Campaign assignment on Court Cases; attribution dictionary |
| **Related** | DDT-556, DDT-558, DDT-561, DDT-574, DDT-826, DDT-1080 |
| **Code** | PR #438 in the ddt-salesforce repo |


---


## 1. Summary

`Court_Case_CampaignID_Updater` filled a Court Case's Campaign from its own hardcoded table of source names. That made it a second, unversioned channel dictionary that could contradict the official one, and it rewrote the Campaign, the attribution engine's top-precedence input. The analysis found its table agreed with the official dictionary on every row, but none of its 9 rules could still match anything: 8 of its source values had been retired and the 9th never existed. The flow was switched off on 2026-09-22.

## 2. What was reported

- 2026-07-28: extract the flow's full table, diff it against `Channel_Mapping__mdt`, decide per row, then either retire the flow or document it as a raw Campaign filler under version control with a named owner. For the July backfill it was deliberately left running so the Campaign could settle first.

## 3. Scope

**In scope**

- `Court_Case_CampaignID_Updater` FlowDefinition: active version 13 set to 0.

**Out of scope**

- `Client_After_Save_Campaign_Influence` and `Lead_After_Save`, which carry the same kind of hardcoded table and still write campaigns: filed as DDT-1080.
- The flow file itself: left in git unchanged so it can be switched back on.

## 4. How it works (background)

The flow ran after save on Court Case. When `CampaignId` was blank, it compared the client's "How did you hear about us?" with a list of 9 source names and set a matching Campaign. Other automation also fills `CampaignId`: the main Court Case after-save flow (copying the client's campaign), `RC_Campaign_Processor` with its nightly run, and a Talkdesk sweeper.

## 5. Root cause

The table drifted out of date. Evidence from the analysis on 2026-09-18:

- All 9 trigger values and all 9 campaigns it wrote were already in `Channel_Mapping__mdt` with identical meaning: 0 disagreements of 9.
- The flow matched on the client's how-heard picklist. 8 of its 9 values were retired picklist values; the 9th was spelled differently from the live value. 0 of 9 rules were reachable.
- In production, 0 client accounts held any of the 9 values, and 0 of 7,793 Opportunities on its campaigns could have been stamped by it.
- The DDT-556 settle pass agreed: of 100 cases touched, the flow filled 0.

## 6. Solution

The FlowDefinition was deployed with `activeVersionNumber` 0. Why retire rather than document: it was unreachable, redundant with the dictionary, and a hazard if any retired picklist value were ever turned back on. Why now: the history re-score in DDT-826 needed `CampaignId` stable; a second table that could change it during the run would make the re-score impossible to reproduce.

## 7. Testing

- Production check-only before deploy.
- After deploy, read back from production through the Tooling API, not from the deploy result: versions 11, 12 and 13 all Obsolete, and the flow definition shows no active version.
- Not covered: no automated test; flow deactivation only.

## 8. Step by step

1. Extract the flow's full table from its metadata.
2. Diff every row against the official dictionary.
3. Check each rule can still match: are its source values active picklist values, and do any records hold them?
4. Count records the flow could have stamped.
5. Set the FlowDefinition's active version to 0; keep the flow file.
6. Confirm in production that no version is active.

## 9. Rollback

Set `activeVersionNumber` back to 13 on the same FlowDefinition and deploy. Version 13 runs exactly as before.

## 10. Known limits

- Two sibling flows with the same pattern were still active at closure (DDT-1080).
- About 5,476 in-window cases with no Campaign will not get one from this path; other automation fills `CampaignId` where it can.

## 11. Reuse for similar issues

- A hardcoded lookup table inside a flow is a hidden dictionary. Diff it against the governed one and check whether its keys still exist.
- "Do not just deactivate it" depends on whether it still writes anything. Count what it can reach before deciding.
- After a flow deploy, confirm the active version by reading production; a green deploy has left flows inert before in this org.
