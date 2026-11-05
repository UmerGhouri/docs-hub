# Shared inbox filter by team member with counts

| | |
|---|---|
| **Ticket** | DDT-803 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-10 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-08-21 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC) |
| **Area** | Shared Inbox (the `emailInbox` component on email Cases) |
| **Related** | None |
| **Code** | PR #334 in the `ddt-salesforce` repo |

---

## 1. Summary

Staff asked whether the Shared Inbox could be filtered to one team member's assigned emails, and whether it could show an email count per person. We added an **"Assigned to"** filter to the inbox's filter bar. The per-person counts were deliberately left out: the firm decided they were the bigger half of the build and nobody had formally asked for them.

## 2. What was reported

The Shared Inbox showed every assigned email across the whole team, with received times, but there was no way to narrow it to one person or to see how many emails each person had. The requester later clarified she had been asking whether this already existed rather than making a formal request. If it were built, her goal was to track overdue and open emails per team member, and to see each person's open-email total as a measure of workload.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `EmailInboxController` | Apex class | `getInboxCases` takes a new optional `assignedToUserId` |
| `EmailInboxControllerTest` | Apex test | Five new tests for the filter; pre-existing empty catch blocks now assert the expected error |
| `emailInbox` (`.js`, `.html`, `.css`) | LWC | New "Assigned to" picker in the filter bar |
| `emailInbox/__tests__/emailInbox.test.js` | Jest test | Filter tests and a mount test |
| `.prettierignore` | Config | Exempts the `emailInbox` bundle (section 6.4) |

**Out of scope**

- **Per-member email counts.** The firm's decision on the ticket: build the filter, skip the counts. If workload visibility becomes a real need, it gets its own ticket.
- **Overdue flags per person.** Part of the requester's wider wish, not built.

## 4. How it works (background)

The Shared Inbox is a Lightning Web Component backed by `EmailInboxController.getInboxCases`. Before this change it filtered on three axes, each a list of active segments:

- **Status:** Open, Resolved
- **Owner:** Unclaimed, Mine, Team
- **Linkage:** Linked, Unlinked

The inbox already had an "Assign to team member" picker, fed by `getAssignableTeamMembers`, which lists the members of the inbox's queue.

## 5. Root cause

Not applicable: this was a feature request. The inbox simply had no per-person filter.

## 6. Solution

### 6.1 Apex: a fifth, optional filter

```apex
@AuraEnabled
public static List<InboxRowDTO> getInboxCases(
    List<String> statusSegments,
    List<String> ownerSegments,
    List<String> linkageSegments,
    String inboxQueueName,
    String assignedToUserId
)
```

- When `assignedToUserId` is set, the query filters `OwnerId` to that user.
- **It supersedes the Owner segments.** Naming one person already answers the Unclaimed / Mine / Team question, and honouring both would let a combination such as "Unclaimed plus one named person" cancel out to an empty list.
- Blank means no filter. A value that is not a user Id is rejected (a test covers it).

### 6.2 Keeping old callers working

The old four-argument signature stays as a plain public overload that passes `null` for the new argument. Apex allows only one `@AuraEnabled` method of a given name, so the annotation moved to the five-argument version and existing callers and tests compile unchanged.

### 6.3 LWC: the picker

- The roster reuses `getAssignableTeamMembers`, the same queue-member list the existing "Assign to team member" picker uses, so no new data is exposed.
- **If the roster fails to load, the picker degrades to "Anyone"** rather than breaking the inbox.
- It is a native `select` element rather than `lightning-combobox`, so desktop and mobile share one render path. LWC cannot bind `value` on a native select, so the current choice is carried on each option's `selected` flag.

### 6.4 Formatting exemption

All three `emailInbox` files already failed the Prettier check on `main` before this PR. Reformatting them would have rewritten 733 lines of live production UI and turned each one into a difference between git and production. The bundle was added to `.prettierignore` instead, the same way an earlier component was handled.

Two follow-up commits on the same PR cleared static-analysis findings the touched files had inherited: lint disables with written reasons for two deliberate timers, and test assertions in place of empty catch blocks. These changed no production behaviour.

## 7. Testing

- **Apex:** deployed to a sandbox, 39 of 39 tests pass, `EmailInboxController` at 79.7% coverage (above the 75% per-class bar). New tests cover: filter to a user; filter supersedes owner segments; filter to another user; blank means no filter; a non-user Id is rejected.
- **Jest:** 8 of 8, including a mount test that asserts the component still renders.
- **Production:** the deployed class contains the new `assignedToUserId` parameter (checked 2026-11-05).

**Not covered at merge time:** the PR recorded that the component had not been loaded in a browser as a non-admin user, which the firm had asked for before it shipped. A validate-only deploy compiles Apex and never renders an LWC. Whether that browser check was done afterwards is not recorded in the ticket.

## 8. Step by step

1. Add the new parameter to the `@AuraEnabled` method and move the annotation; keep the old signature as a non-annotated overload that passes `null`.
2. When the parameter is set, add `OwnerId = :assignedToUserId` to the query and skip the owner-segment clause.
3. Validate the value is a user Id and throw a clear error otherwise.
4. In the LWC, load the roster from the existing team-member method, render a native select with an "Anyone" option, and pass the chosen Id to `getInboxCases`.
5. Make a roster failure fall back to "Anyone".
6. Write Apex tests for each filter case and a Jest mount test.
7. Deploy to a sandbox, run the tests, and **open the inbox in a browser as a non-admin user** before production.
8. Deploy the class, test class and LWC bundle to production.

## 9. Rollback

Redeploy the previous versions of `EmailInboxController`, its test and the `emailInbox` bundle. Safe: the change adds a filter and writes no data. The old four-argument method still exists, so rolling back only the LWC also works.

## 10. Known limits

- **No per-person counts.** Workload visibility was the requester's real goal and is not delivered by this change.
- **The roster is the inbox queue's members.** Someone who owns emails but is not in the queue cannot be picked.
- **Choosing a person overrides the Owner segments**, so Unclaimed / Mine / Team have no effect while a person is selected.

## 11. Reuse for similar issues

**How to recognise one:** a request to slice an existing live component by one more dimension.

- **Ask whether it is a request or a question.** This one began as "does this exist?"; scoping it down to the cheap, useful half was the right call.
- **Add an optional parameter and keep the old signature** as a non-`@AuraEnabled` overload, so nothing else has to change.
- **Decide how new and old filters interact** and write it down; silently combining them can produce empty results.
- **Reuse the existing data source** for pickers rather than adding a new query.
- **A green pipeline does not render an LWC.** Open the screen before it ships.
