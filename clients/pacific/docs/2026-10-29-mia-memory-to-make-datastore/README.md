# Mia's memory moved to a Make Data Store

| | |
|---|---|
| **Status** | Live since 2026-10-29 (verified 2026-11-02) |
| **Requested by** | Client admin, 2026-10-29 |
| **Resolved by** | Support team |
| **Systems** | Make (Data Store, scenarios), Google Sheets (removed) |
| **Area** | Mia text agent: conversation memory |
| **Change log rows** | 75–81 |

---

## 1. Summary

Mia remembers each conversation (the AI thread ID per phone number, and whether the lead said "Stop") in a Google Sheet. That sheet, "Mia Text Agent DB", was in the **previous developer's personal Google account**, connected to Make through a connection named after him ("Sheet [...]").

We moved this memory into a **Make Data Store in Pacific's own Make account**, imported all 88 existing conversations, and switched both Mia scenarios to it.

## 2. Client request

> "Yes please change that, nothing should be saved in an affiliate's personal email."

## 3. Scope

**In scope**

| Scenario | Use of the memory |
|---|---|
| `Mia Text Agent [production]` | Find the conversation by phone; save the thread after each reply |
| `Mia Stop Mia Text` | Mark the lead's conversation `mia_text = Stop` |
| `Mia Text Agent [sandbox]` | Same as production, separate store |

**Out of scope:** Mia's instructions and replies, RingCentral connections, Salesforce. The previous developer's other files (not listed or opened).

## 4. Data model

Data structure "Mia Text Agent DB", **key = lead phone number (as text)**.

| Field | Meaning | Old sheet column |
|---|---|---|
| `from_number` | Lead's phone (search field) | A From Number |
| `user_number` | Lead's phone (used by Mia Stop) | B User Number |
| `thread_id` | OpenAI / Make AI-agent thread, so Mia continues the same conversation | C Thread ID |
| `contact_id`, `user_id` | Carried over from the sheet | – |
| `ringcentral_conversation_id` | RingCentral conversation ID | E |
| `ringcentral_message_id` | Last inbound message ID | F |
| `last_message` | Mia's last reply | H |
| `mia_text` | `Stop` when the lead opted out | I |

| Store | ID | Size |
|---|---|---|
| Production "Mia Text Agent DB" | [id] | 2 MB (88 records at import) |
| Sandbox "Mia Text Agent DB [Sandbox]" | [id] | 1 MB |

## 5. Solution

### 5.1 Backup first
A temporary read-only Make scenario read the sheet (`google-sheets:makeAPICall`, range `'Message Logs'!A1:I`) and returned it through a webhook response. It was **deleted right after**. Saved as `Mia-Text-Agent-DB-Backup-2026-10-29.xlsx`: Conversations (matched to leads), an exact copy of Message Logs, and an About tab. 88 rows, 37 "Stop".

### 5.2 Mia Text Agent
| Module | Before | After |
|---|---|---|
| #8 | Google Sheets search | `datastore:SearchRecord`: `from_number = {{toString(9.phone)}}`, **continue if no result** |
| #13 | Google Sheets add/update row | `datastore:AddRecord`: key `{{toString(9.phone)}}`, **overwrite = true** (add or replace) |
| Route filters | Sheet columns | `thread_id` / `mia_text` from the store |
| #50 / #37 / #39 | Duplicate-row clean-up for the sheet | Removed (one record per key can't have duplicates) |

### 5.3 Mia Stop
`#3 datastore:SearchRecord` (`user_number = {{toString(2.phone)}}`) → `#5 datastore:UpdateRecord` (key `{{3.key}}`, `mia_text = Stop`, **upsert = false**) → confirmation text. Same behaviour as before: if the lead never texted Mia, there's no record and nothing changes.

### 5.4 Import
All 88 rows were re-read right before the switch, written through a temporary import webhook (then deleted), and the two scenarios were switched within minutes.

## 6. The bug found in testing

**Every message started a new conversation.** The phone was saved as a **number**, but the search compared it with text, so it never matched.
**Fix:** save **and** search with `{{toString(...phone)}}`. The bad test records were deleted.

Sign it works: a continuing conversation runs **8 operations**; a new one runs 9.

## 7. Testing
- **Sandbox (2026-10-29):** one new and seven continued messages from a test number. Mia recalled earlier details and continued the thread.
- **Production check (2026-11-02):** RingCentral history of the main line since the switch showed 7 numbers sent 8 texts, and **Mia answered all 8 within 3 minutes**, including a continued conversation.

## 8. Rollout
1. Backup the sheet (5.1).
2. Create the production store and structure; import the fresh read.
3. Update both Mia scenarios (blueprint backups taken first).
4. Confirm the old Google Sheets connection has **0 usages**.

## 9. Rollback
Restore `prod_mia-agent_before_datastore.json` / `prod_mia-stop_before_datastore.json` (Make → scenario → ⋮ → Import blueprint). They point back at the sheet, so this only works while the old Google Sheets connection still exists.

## 10. Known limits / follow-ups
- **Still to do:** a live "Stop" test in production; delete the old Google Sheets connection in Make; ask the previous developer to delete the sheet.
- **Store size:** each record is under 1 KB; 2 MB holds thousands. Make Pro allows 20 MB of data stores in total.
- **The Make UI's execution detail doesn't show module outputs through the API.** To read store contents, use Data Store → Browse, or a temporary webhook scenario.

## 11. Reuse for similar issues
- **Moving off someone's personal account:** back up → build in the sandbox → import a fresh copy → switch → confirm the old connection has 0 usages → then delete it.
- **Make Data Store keys and searches:** always use `toString()` on phone numbers, so the stored value and the search value are both text.
- **Exporting data from Make without the API:** a temporary webhook scenario that ends in Webhook Respond with `TransformToJSON`; delete it afterwards.
