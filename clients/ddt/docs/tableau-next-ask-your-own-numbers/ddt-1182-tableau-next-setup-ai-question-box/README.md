# Turn on Tableau Next and its AI question box in production


| | |
|---|---|
| **Ticket** | DDT-1182 |
| **Project** | Tableau Next: Ask-Your-Own-Numbers |
| **Status** | Live since 2026-10-21 (reached production) |
| **Priority** | High |
| **Reported by** | Firm leadership, 2026-10-02 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Tableau Next, Data Cloud, Agentforce, permission sets) |
| **Area** | Self-serve reporting: ask the firm's numbers in plain English |
| **Related** | DDT-1183, DDT-1185, DDT-1187, DDT-1188 |
| **Code** | PR #782 in the ddt-salesforce repo (record of the permission set) |


---


## 1. Summary

The firm had Tableau Next licences (5 Creator, 41 view-only) but nobody could ask it a question: its data store, Data Cloud, had been disconnected since early 2025, and the AI feature was switched off. The old Data Cloud setup was cleared out, Data Cloud was set up fresh inside the production org, and on 2026-10-20 the Tableau Agent and related features were switched on. A new agent, "DDT Tableau Analytics", answers "How many cases did we sign in August 2026?" with 267, the approved number. Firm leadership ran the same question and got the same answer.

## 2. What was reported

- 2026-10-02: confirm Tableau Next is fully enabled (not just licensed), confirm the natural-language question feature is on and can answer a question, record the release and features available, and decide who gets the last 2 Creator seats. Done when a working test question and answer is posted.

## 3. Scope

**In scope**

- Clearing the old, disconnected Data Cloud setup (one connection and 8 system data streams last refreshed 2025-01-09).
- Setup switches: Tableau Agent, Data Q&A Indexing and Semantic Enrichments, Data Analysis (Tableau Next features), and Agentforce (org-wide).
- A new agent, `DDT Tableau Analytics`, built from Salesforce's Analytics and Visualization template, with the `DDT Pilot` semantic model as its default.
- A new permission set, `DDT_Tableau_Agent_Access`, that grants access to that agent only.
- Deactivating the 2024 "Einstein Copilot" agent.

**Out of scope**

- Streaming the firm's objects into Data Cloud (DDT-1183) and building the model (DDT-1185).
- The last 2 Creator seats: firm leadership decided to leave them empty for now.
- Retrieving the agent itself into git (a follow-up noted in PR #782).

## 4. How it works (background)

Tableau Next does not read Salesforce objects directly. It reads Data Cloud, where Salesforce objects are copied by data streams. A semantic model on top of Data Cloud defines each business term (metric, dimension, synonyms). The question box is an Agentforce agent that reads the semantic model and answers in plain English. The question panel only lists agents the user has been given access to.

## 5. Root cause

Not applicable: this was a setup task, not a fault. The blockers found along the way:

- Data Cloud had been disconnected since early 2025, with no instance behind it. It was cleared with Data Cloud Setup's "Clean Up and Reset" on 2026-10-06, after Salesforce's dependency check passed and no packages were listed as affected.
- Tableau Agent was Disabled in Setup. Its terms say it consumes AI credits, so firm leadership asked for credit tracking (DDT-1183) to be in place first.
- Turning on Agentforce also made the agent panel visible to 6 staff who held an older default-agent permission. They would have seen the 2024 Einstein Copilot agent, which answered the August question with 5,851 because it does not know the approved terms.

## 6. Solution

On 2026-10-20 (Central time, from the Setup Audit Trail unless approximate):

1. Tableau Agent on (3:39 AM).
2. Data Q&A Indexing and Semantic Enrichments on, so the agent can read the model (about 3:49 AM).
3. Data Analysis on (3:51 AM).
4. Agentforce on, org-wide (3:55 AM). Salesforce had been switching this on for every org since September.
5. New agent `DDT Tableau Analytics` activated (about 4:14 AM), with `DDT Pilot` as its default model.
6. `DDT_Tableau_Agent_Access` deployed and assigned to the four Creator users. It grants access to that one agent and nothing else (no object, field, Apex or app permissions).
7. Einstein Copilot deactivated, so staff with the older permission could not get a wrong answer from it.

Why a separate permission set: access to the agent is then given or removed in one step, and only the people meant to test it see it.

## 7. Testing

- 2026-10-20: the Salesforce lead asked the agent "How many cases did we sign in August 2026?" on the DDT Pilot model page. Answer: 267 (Hire Status = Hired, Did Not Hire = false, counted by Hire Date), matching Salesforce.
- Firm leadership asked the same question the same day and got 267.
- After the Data Cloud clean-up: all licence counts and all 21 Data Cloud, Tableau and Agentforce permission-set assignments were unchanged, and the Tableau Next test workspace was still there. No wider test of other features was run.

## 8. Step by step

1. Check licences and seats in Setup (Tableau Next Creator and view-only, Data Cloud).
2. If Data Cloud is disconnected, use Data Cloud Setup → Clean Up and Reset. Write down every connection, data stream and data space it removes.
3. Set up Data Cloud fresh in the production org and stream the needed objects (DDT-1183).
4. Turn on credit tracking before any AI feature, so the cost of the test can be seen.
5. Turn on Tableau Agent, Data Q&A Indexing, Data Analysis and Agentforce. Note each time.
6. Create the agent from the Analytics and Visualization template and set its default model.
7. Give access through a permission set that grants only that agent.
8. Check which older agents became visible when Agentforce was switched on, and deactivate any that give unapproved answers.
9. Ask one question with a known answer and compare it to Salesforce.

## 9. Rollback

Remove the four assignments and delete `DDT_Tableau_Agent_Access`. The Setup switches can be turned off in Setup. Einstein Copilot can be reactivated if wanted. The Data Cloud clean-up cannot be undone, but it only removed an instance that had been disconnected since early 2025.

## 10. Known limits

- The data model object names removed by the clean-up were not saved before it ran.
- The agent was built in Agentforce Builder and is not yet in git.
- The 2 remaining Creator seats were left empty.

## 11. Reuse for similar issues

- Signs of the same problem: Tableau Next is licensed but questions fail, or the agent panel shows only an older Copilot agent. Check Data Cloud's connection and the Tableau Agent switch first.
- Always test with one question whose answer is already known from Salesforce.
- After turning on Agentforce, check who can now see the agent panel and which agents it lists.
