# Two flows with hardcoded how-heard to Campaign tables


| | |
|---|---|
| **Ticket** | DDT-1080 |
| **Project** | Marketing Attribution |
| **Status** | Done 2026-09-22 |
| **Priority** | Medium |
| **Reported by** | Firm leadership (split from DDT-560), 2026-09-22 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Flow / custom metadata) |
| **Area** | Marketing attribution: the channel dictionary |
| **Related** | DDT-560, DDT-561, DDT-1083 |
| **Code** | PR #447 in the ddt-salesforce repo (documentation only) |


---


## 1. Summary

When DDT-560 retired one flow that set a Court Case's Campaign from a hardcoded list of how-heard answers, two more live flows were found with the same kind of table. Both tables were extracted and compared with the channel dictionary. They were almost entirely dead: only one rule per flow could still fire, and it agreed with the dictionary. Both flows were left as they were, and a standing rule was written so no new table of this kind is ever added.

## 2. What was reported

- 2026-09-22: `Client_After_Save_Campaign_Influence` and `Lead_After_Save` carry hardcoded Campaign Ids keyed on how-heard answers, a second, unversioned channel dictionary. Diff each table against `Channel_Mapping__mdt`, decide per rule, and make sure no flow sets a Campaign from a how-heard answer the dictionary does not know. Do not simply deactivate either flow.

## 3. Scope

**In scope**

- The campaign legs of `Client_After_Save_Campaign_Influence` (Account) and `Lead_After_Save` (Lead).
- `docs/attribution/CHANNEL-BUCKETS.md` and `docs/attribution/CAMPAIGN-LAUNCH-CHECKLIST.md`.

**Out of scope**

- Removing the dead rules from the flows: not worth opening a large general-purpose flow for rules that cannot fire.
- `Court_Case_CampaignID_Updater`: retired under DDT-560.

## 4. How it works (background)

The Campaign on a record is the attribution engine's highest-precedence token. `Channel_Mapping__mdt` is the dictionary that maps tokens to channels. A flow that sets the Campaign from a how-heard answer is effectively a second dictionary that nobody owns. Switching such a flow off would silently change attribution on every new record it would have touched.

## 5. Root cause

Diff, 2026-09-22 (tables extracted from git, all Campaign Ids resolved to names in production, every token and campaign name checked against the dictionary):

| Flow | Rules | Trigger answer still active | Fired in last 90 days |
|---|---|---|---|
| `Client_After_Save_Campaign_Influence` | 29 | 1 | 8 Accounts, all on that one answer |
| `Lead_After_Save` (campaign leg) | 29 | 1 | 5 Leads, all on that one answer |

- The other 28 answers on each flow are inactive picklist values, so those rules cannot fire.
- All 29 tokens and all 29 campaign names have dictionary rows; 28 of 29 rules write a Campaign in the same channel as the token.
- One disagreement, in `Lead_After_Save` only: a direct-mail mailer answer points at a lead-vendor campaign instead of the mailer's own campaign, a copy-paste error. It is unreachable because the answer is inactive.

## 6. Solution

1. Left both flows unchanged. The one live rule (a medical-provider referral answer to its own campaign) agrees with the dictionary and also feeds Campaign Influence reporting, which the resolver does not.
2. Recorded the mailer mis-map in `CHANNEL-BUCKETS.md` to fix the next time `Lead_After_Save` is edited for another reason.
3. Wrote the standing rule in `CHANNEL-BUCKETS.md`: no new how-heard to Campaign rule goes into a flow; a new token gets a `Channel_Mapping__mdt` row, and if a Campaign is wanted it is created with the same name. Added a checkbox for it on the campaign launch checklist. Why: the real fix for a second dictionary is to stop growing it.

## 7. Testing

Not applicable: no code or flow change. The diff itself is the evidence (section 5).

## 8. Step by step

1. Extract every hardcoded Id and condition from the flow XML.
2. Resolve each Id to a record name in production.
3. Check each trigger value is still an active picklist value, and count how often each rule fired.
4. Compare each token and target against the dictionary.
5. Decide per rule: fold in, keep, or retire. Post the diff before deciding.
6. Write a standing rule so the pattern does not grow again.

## 9. Rollback

Not applicable: only documentation changed.

## 10. Known limits

- The 28 dead rules stay in both flows; if any of those picklist answers is reactivated, its rule fires again, including the mis-mapped mailer rule.

## 11. Reuse for similar issues

- Signs of the same problem: a flow with a long decision on a picklist answer and hardcoded record Ids in its assignments.
- Check reachability (active picklist values, recent fires) before planning a rewrite; dead rules can be documented rather than removed.
- Do not deactivate a flow that writes a high-precedence token without knowing what replaces it.
