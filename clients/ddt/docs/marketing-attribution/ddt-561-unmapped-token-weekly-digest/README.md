# Weekly digest of marketing sources the dictionary cannot map


| | |
|---|---|
| **Ticket** | DDT-561 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-09-15 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-07-28 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex / custom metadata / email), Slack |
| **Area** | Attribution dictionary upkeep (`Channel_Mapping__mdt`) |
| **Related** | DDT-555, DDT-557, DDT-560, DDT-574, DDT-947, DDT-1078, DDT-1080 |
| **Code** | PR #388, #430, #433, #437 and #439 in the ddt-salesforce repo |


---


## 1. Summary

The attribution model is only as good as its channel dictionary. Every new campaign, lead vendor or search source that nobody maps lands in "Unknown" or on a pattern guess, and nothing says so. This ticket built a weekly digest that lists source tokens with no dictionary row, sorted by volume and revenue, with draft mapping rows ready to approve. It also lists records where two referral sources disagree and sends a mid-week alert when an unmapped token spikes. It named an owner (firm leadership now, a named successor in firm operations later) and added a bucket reference and a campaign-launch checklist. A bug in the first live email was fixed on 2026-09-22.

## 2. What was reported

- 2026-07-28: build a weekly digest of tokens resolving to basis "Pattern Guess" or "None", with volume and revenue and drafted dictionary rows; a section for `Attribution_Conflict__c`; a mid-week threshold alert; add "create the mapping row" to the campaign-launch checklist; document the buckets.
- 2026-08-09: ownership decided. Firm leadership owns the dictionary now with a named successor; the digest goes to both, with reasoning, and logs who decided each mapping and when.
- 2026-09-14: build assigned to the support team; it must not take a new scheduled-job slot.

## 3. Scope

**In scope**

- New Apex `UnmappedTokenDigest` and `UnmappedTokenDigestTest`.
- `GlobalMinuteScheduler`: one wrapped call added; no existing line changed.
- New `Attribution_Config__mdt` settings: a digest on/off switch, recipients, alert threshold and lookback days.
- New `Channel_Mapping__mdt` fields: `Decided_By__c`, `Decided_On__c`, `Decision_Note__c`.
- Docs: the channel buckets reference and the campaign-launch checklist.

**Out of scope**

- Writing mapping rows automatically: the digest proposes them as text; a person applies them.
- Records stamped before their Campaign or how-heard arrived: a re-scoring problem, fixed in DDT-1078.

## 4. How it works (background)

The resolver (DDT-555) looks a token up in `Channel_Mapping__mdt`. With no row it falls back to name patterns, which reach only 6 of the 15 channel buckets; everything else becomes `Unknown`. The org is near its limit of 100 scheduled Apex jobs, so recurring work rides a shared every-minute dispatcher, `GlobalMinuteScheduler`, at no slot cost.

## 5. Root cause

Not applicable for the build. For the 2026-09-22 bug: the first live digest listed five tokens as unmapped and drafted rows for them, but all five already had rows. Two were deliberate decisions (a main office line treated as a "door", and manually created leads treated as excluded). The digest never asked the dictionary. It picked any record with basis "None" or "Pattern Guess" and re-derived the token with an old precedence order. Under the current rule, basis "None" can also mean the engine skipped a door or excluded token on purpose.

## 6. Solution

- **Rides the dispatcher.** The digest runs Monday 07:11 Chicago time; the spike alert runs daily at 13:11. Why: a direct `System.schedule()` throws when no slot is free, and the job would deploy green and never send. The call is wrapped in try/catch, placed above the other riders, and pinned to the Chicago time zone.
- **Proposes, does not apply.** Deploying a custom metadata record writes the whole record and blanks any field left out. A digest that applied its own drafts could silently erase curated values.
- **Email for the digest, Slack for the alert.** A long list of draft rows suits email; a spike is a short notice.
- **Decision log on each row** so the successor's progress can be read from data.
- **Fix (PR #433):** the digest now picks the token the same way the runner does (`AttributionTokenSelector`) and reports it only when the dictionary lookup has no row. PR #439 corrected the wording of the empty-digest sentence. PR #430 and #437 switched the digest on and set its recipients.

## 7. Testing

- Production check-only before first deploy: 11 of 11 components, 42 tests, 0 failures; the digest class at 87.3% coverage. The first run had failed at 74.8% because no test called the alert path; a real test was added.
- What tests cannot prove: email delivery, the Slack call, and that the dispatcher fires. These were checked after deploy.
- First live email, 2026-09-22: wrong (five false gaps). After the fix, five new tests; 33 of 33 green in the partial-copy sandbox and production. A re-send the same day returned "No unmapped tokens in the window", the expected answer.
- Recipients were limited to one developer until the email read correctly, then widened.

## 8. Step by step

1. Ask the dictionary itself whether a token has a row; never infer a gap from the basis field.
2. Use the same token-selection code as the resolver.
3. Rank gaps by volume and revenue so the most costly gaps come first.
4. Emit draft rows as text with the reasoning; let a person apply them in Setup.
5. Host the schedule on the shared dispatcher, wrapped, with a pinned time zone.
6. Send to one reader first; widen once a live email reads correctly.
7. Add "create the mapping row before the first click" to the campaign-launch checklist.

## 9. Rollback

Untick the digest switch on `Attribution_Config__mdt`. The dispatcher call becomes a no-op and logs a warning; nothing else changes. The feature writes no records, so there is nothing to undo.

## 10. Known limits

- The digest runs at minute 11, which would not survive a planned reduction of dispatcher minutes. A test flags this; the gates would move to minute 12.
- Lead vendors still share one bucket with no per-vendor split, which limits vendor renewal decisions.
- A test showed a Lead's `Referral_Source__c` being cleared on save by some other automation; not investigated in this ticket.

## 11. Reuse for similar issues

- A model that maps raw values to categories needs an owner and a regular list of what it failed to map.
- Report a gap only after asking the source of truth; a proxy field can carry deliberate decisions.
- Never let a job auto-deploy custom metadata records; propose and let a person apply.
- On an org near its scheduled-job limit, host new recurring work on an existing dispatcher.
