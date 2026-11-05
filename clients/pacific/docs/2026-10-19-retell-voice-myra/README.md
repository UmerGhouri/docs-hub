# Retell voice changed to Myra

| | |
|---|---|
| **Status** | Live since 2026-10-19 |
| **Requested by** | Client admin |
| **Done by** | Client admin ([Welcome]) and support team (all other agents) |
| **Systems** | Retell |
| **Change log rows** | 41 |

---

## 1. Summary

The client admin picked a new voice for the AI calls. Every calling agent was switched from **"Vanessa New"** to **"Myra"**.

## 2. Scope

| Agent | Used for |
|---|---|
| Pacific Mortgage [Welcome] | Inbound calls to the AI calling number; the old first call |
| Pacific Mortgage [Retry] (`[agent id]`) | Day 3, 7, 14 calls (Send Welcome Retry Calls) |
| Pacific Mortgage [No Contact] (`[agent id]`) | Day 21 + final calls (Send No Contact Retry Calls) |
| [Follow Up], [Welcome Legacy Booking], [Test Agent] | Not used by the active call sequence |

Chatbot agents ([Chatbot Cal.com], [Chatbot SF Booking]) have no voice and were not changed.

## 3. How to change a voice

1. Retell dashboard → **Agents** → open the agent.
2. Agent settings (top bar) → **Voice** → choose the voice.
3. **Publish** the agent. Unpublished changes don't affect live calls, which use the published version.
4. Repeat for every calling agent.

_Screenshot in the private copy: 01-agents-on-myra.png (All calling agents on Myra)._

## 4. Rollback
Set the voice back to "Vanessa New" on each agent and publish.

## 5. Reuse for similar issues
- **Agent-level settings** (voice, voicemail, language) must be changed on **every** agent that calls. Make picks the agent per call, not the phone number.
- To find which agent a Make scenario uses, open the scenario's Retell HTTP module and read `override_agent_id`. The agent ID is also in the dashboard URL.
