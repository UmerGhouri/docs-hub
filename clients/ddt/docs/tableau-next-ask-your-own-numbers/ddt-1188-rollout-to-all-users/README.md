# Roll out the AI question tool to every user, with a one-page how-to


| | |
|---|---|
| **Ticket** | DDT-1188 |
| **Project** | Tableau Next: Ask-Your-Own-Numbers |
| **Status** | Live since 2026-10-30 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-10-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Tableau Next, Data Cloud licences, permission sets), Google Docs, Slack |
| **Area** | Self-serve reporting: rollout |
| **Related** | DDT-1186, DDT-1187, DDT-1182 |
| **Code** | No code change |


---


## 1. Summary

After a one-week pilot with three staff (2026-10-22 to 10-29), every active Salesforce user got access to the "DDT Tableau Analytics" agent on 2026-10-30, through licences and permission sets rather than one-off grants. A one-page how-to in a Google Doc was shared and pinned in Slack. No wrong answers were reported during the pilot; two first-try misses were found and handled.

## 2. What was reported

- 2026-10-02: pilot with 3 to 5 users for a week and log every wrong or "I don't know" answer; then give view-only access to everyone through the access-rules permission set; write a one-page how-to (where to click, 5 example questions, what to do when an answer looks wrong) and a short list of the terms people can use.

## 3. Scope

**In scope**

- 37 active users on a Salesforce licence: Tableau Next view-only and Data Cloud licences, three permission sets (Tableau unmetered view-only, `DDT_Tableau_Agent_Access`, `DDT_Tableau_Data_Reader`) and Viewer on the DDT Pilot model.
- The how-to Google Doc and the announcement.

**Out of scope**

- Integration and system accounts: left out.
- The standard Data Cloud User permission set: not given (see DDT-1187).

## 4. How it works (background)

On any Salesforce page, a user opens the agent panel (top right), picks "DDT Tableau Analytics" and types a question. Each answer has a "Sources" line naming the definition used. Access needs the view-only licence, the agent permission set, the reduced Data Cloud permission set and Viewer sharing on the model.

## 5. Root cause

Not applicable: this was a rollout. Pilot findings:

- A case list showed 265 instead of 267 because two records with identical names were merged into one row.
- "Unanswered dials" first gave a generic error, then the right number (1,822 for September 2026) on the second try.

## 6. Solution

1. Pilot with three staff, chosen by firm leadership, set up with exactly the access every user would later get.
2. Wrong answers reported in the firm's tech-support Slack channel, each becoming a ticket in this project.
3. Fixed the merged-rows problem: lists now include the Court Case ID.
4. Rolled out to all active users on 2026-10-30.
5. Updated the how-to: per-rep numbers moved out of "Not available yet", examples added for rep, attorney, case manager and court appearances, and a tip that "my" questions ask for your name first. Posted and pinned in the firm's announcements channel.

How-to contents: how to open the agent, 5 example questions, tips (say the month and year, start a new conversation per question, check "Sources"), the words it understands, what is not available yet, and how to report a wrong answer. Trust balances, salaries and client personal details are not available at all.

Why a pilot first: each wrong answer found by a few users becomes a fix before everyone relies on the tool.

## 7. Testing

- Pilot access tested as two pilot users: August signed = 267.
- Pilot usage (2026-10-22 to 10-29): one user asked 23 questions in 9 chats, one asked 2 in 1 chat, and the third account ran 99 questions in 33 chats, most of them the full test bank.
- Wrong answers reported: 0.
- After rollout, a query confirmed no active user was missing any licence, permission set or model share.

## 8. Step by step

1. Pick 3 to 5 pilot users and give them exactly the production access set.
2. Name one channel for reporting wrong answers, and turn each into a ticket.
3. Track first-try misses, not only final answers.
4. After the pilot, fix or ticket every miss.
5. Assign licences and permission sets to every active user; leave integration accounts out.
6. Query afterwards that nobody is missing a piece.
7. Share the how-to where staff already look and pin it.

## 9. Rollback

Remove the three permission sets and the model share from users, and release the view-only licences. Nothing else changes.

## 10. Known limits

- Answers that need terms still marked "not available yet" (call rates, one-call close, speed to lead, call score) are not answered.
- First-try misses can happen right after a term changes, because the agent re-indexes with a delay.

## 11. Reuse for similar issues

- Pattern: pilot with the exact production access set, count questions per pilot user so "no wrong answers" has a number behind it, then roll out by permission set.
- After any bulk access change, query that every intended user has every piece.
