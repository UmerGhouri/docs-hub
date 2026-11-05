# Discovery Puller posts tag the case manager and attorney

| | |
|---|---|
| **Ticket** | DDT-1144 |
| **Project** | Support tickets |
| **Status** | Live since 2026-10-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-09-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, Chatter), Slack |
| **Area** | Discovery Puller: the audit post written when a discovery pull finishes |
| **Related** | None |
| **Code** | PR #553, PR #561 and PR #572 in the `ddt-salesforce` repo |

---

## 1. Summary

The firm wanted more people notified when the Discovery Puller finishes pulling discovery for a court case. The ticket asked for tags on the **Slack** post. After two rounds of clarification, the final behaviour is: the **Chatter** post on the Court Case @mentions the primary attorney, plus the attorney assigned to the next court date when that is a different person. The Slack post is back to exactly what it was before the ticket, and the case manager is not tagged.

## 2. What was reported

- **2026-09-28, DDT-1144 opened:** every Discovery Puller post should tag the primary case manager and the attorney on the next court appearance (if different from the primary attorney). A missing case manager, no upcoming appearance or no Slack ID must not break the post.
- **2026-09-29, approval:** the scope was corrected. The ticket said the Slack post already tagged the primary attorney. It did not: the Slack text only printed plain names. The only real @mention was on the Chatter post.
- **2026-09-30:** after the Slack version went live, the requester clarified that the tags belong on the **Chatter** post, not Slack.
- **2026-09-30, later:** a staff member said the post should not tag the case manager, and the approver agreed.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `DiscoveryPullTriggerHandler` | Apex class | Builds the mention line on the Chatter audit post |
| `DiscoveryPullTriggerHandlerTest` | Apex test | Tests for the mention rules and the failure paths |
| `docs/deploys/discovery-puller-chatter-mentions.md` | Deploy doc | Records the history and the final behaviour |

**Out of scope**

- **The Slack post.** It was returned to its pre-ticket text, which only says who ran the pull.
- **Failed or timed-out pulls.** They never got a Chatter post before this ticket, so they get no mentions now either.
- **The case manager.** Tagged in the first two versions, removed in the third at the requester's request.

## 4. How it works (background)

```
Staff start a discovery pull on a Court Case (Opportunity)
  → the pull finishes with status Done and at least one file
  → DiscoveryPullTriggerHandler writes an audit post to the Court Case's Chatter feed
  → it also sends a short Slack message saying who ran the pull
```

Before this ticket the Chatter post already @mentioned the primary attorney. The Slack message carried names as plain text, never as Slack tags.

## 5. Root cause

Not applicable: this was a feature request, not a fault.

The useful lesson is in the ticket's wording. It assumed the Slack post already tagged the primary attorney, so it asked for "more tags on the Slack post". Checking the code showed the real @mention was on Chatter. Building to the ticket's description first produced a Slack version (PR #553) that had two practical gaps, both recorded at the time:

- Most attorneys had no Slack member ID stored in Salesforce, so their names printed as plain text instead of a tag.
- Most attorneys were not in the discovery Slack channel, so a tag would not reach them anyway.

Chatter mentions need neither a Slack ID nor channel membership, which is why moving the tags to Chatter was the right final answer.

## 6. Solution

### 6.1 Who is mentioned

| Who | Field |
|---|---|
| Attorney | `Opportunity.Primary_Attorney__c` |
| Next court date | `Court_Appearance__c.Assigned_Attorney__c` on the **earliest** appearance where `Future_Court_Date__c = true` |

The Chatter post now starts with one line, then the usual audit body:

```
Attorney: @A · Next court date: @B
```

### 6.2 The rules, and why

- **Nobody is listed twice.** The next-court-date attorney is skipped when it is the primary attorney.
- **Anyone unassigned is left off.** If nobody is assigned, the post is the plain audit body. A missing court date never blocks the post.
- **The right fields.** `Assigned_Attorney__c`, not the older `Assigned_Lawyer__c` Contact lookup. The case manager field chosen in the first versions was `Opportunity.Court_Case_Manager__c`, not the older formula field with hardcoded Ids; it is no longer read.
- **Inactive users are written by name, not mentioned.** Salesforce's ConnectApi rejects the whole post if any mention is an inactive user. If nobody on the list is active, the handler skips ConnectApi and posts a plain feed item.
- **The audit post is never lost.** If the mention post fails for any reason, the existing fallback posts a plain feed item with the names as text, and the failure is written to `API_Error_Log__c` because nobody was notified. A failed lookup query is logged the same way.

### 6.3 The three PRs

| PR | Date | What it did |
|---|---|---|
| #553 | 2026-09-30 | Tags on the Slack post (attorney, case manager, next-court-date attorney) |
| #561 | 2026-09-30 | Tags moved to the Chatter post; Slack post restored to its pre-ticket text |
| #572 | 2026-10-01 | Case manager tag removed from the Chatter post |

## 7. Testing

`DiscoveryPullTriggerHandlerTest` covers:

- The full trigger path: a Done pull posts a feed item that leads with both attorney names in order, and a case manager set on the record is not named.
- No attorney and no court date: the post body is unchanged.
- The primary attorney is also on the next court date: listed once.
- Several appearances: the earliest future one wins over a past one and a later one.
- Active users become mention segments; inactive users become plain text.
- Only inactive users assigned: ConnectApi is skipped.
- Lookup failure and rejected mention post: each writes one `API_Error_Log__c` row, and nothing is written when nothing fails.

The final version was validated against production with 22 of 22 tests passing and the handler at 87% coverage.

**Not covered:** ConnectApi cannot post in an Apex test, so no test created a real mention. Test court dates are pinned to 16:00 GMT because a validation rule on `Court_Date__c` rejects times outside 12:00 to 01:00 GMT. The live check on a real pull was left with the requester; its result is not recorded in the ticket.

## 8. Step by step

1. Confirm which post carries the existing @mention (here, Chatter) before adding tags anywhere.
2. In `DiscoveryPullTriggerHandler`, load the primary attorney and the earliest future `Court_Appearance__c` with its `Assigned_Attorney__c`.
3. Build the mention list: drop blanks, drop the duplicate, write inactive users as text.
4. Post through ConnectApi only when at least one person is active; keep the plain feed item fallback and log any failure to `API_Error_Log__c`.
5. Add tests for each rule in section 7.
6. Retrieve the handler from production, confirm it matches `main`, then validate against production with `DiscoveryPullTriggerHandlerTest`.
7. After approval, deploy the two classes.
8. On the next Done pull, open the Court Case's Chatter feed and confirm the mention line and the notifications.

## 9. Rollback

- **To bring back the case manager tag:** redeploy the two classes as they were after PR #561.
- **To undo the Chatter mentions:** redeploy from PR #553 (Slack tags) or from the version before DDT-1144.

There is no data change to undo.

## 10. Known limits

- Only the **earliest** future appearance counts. If it has no assigned attorney, there is no next-court-date mention, even if a later appearance has one.
- Failed and timed-out pulls get no Chatter post, so no mentions.
- An inactive user is named but not notified.

## 11. Reuse for similar issues

- **Check what the system does today before building what the ticket says it should add to.** The ticket's "already tags" claim was wrong, and the first build followed it.
- **Pick the channel that reaches the people.** A Slack tag needs a stored Slack member ID and channel membership; a Chatter mention needs only an active Salesforce user.
- **Notification code must never block the record it describes.** Fall back to a plain post and log the failure.
- **Guard against inactive users** in any ConnectApi mention, because one inactive mention rejects the whole post.
