# Send the website visit history with each form, one touch row per visit


| | |
|---|---|
| **Ticket** | DDT-821 |
| **Project** | Marketing Attribution |
| **Status** | Live since 2026-11-02 |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-08-24 |
| **Resolved by** | Support team, with the website developer |
| **Systems** | Salesforce (Apex REST, custom field, permission set), firm website (Cloudflare Worker) |
| **Area** | Attribution: website visits before a form submission |
| **Related** | DDT-811, DDT-813, DDT-814, DDT-820, DDT-1131, DDT-1134, DDT-1361 |
| **Code** | PR #501 and #502 in the ddt-salesforce repo; PR #293, #298 and #321 in the website repo |


---


## 1. Summary

The website already tracked each visitor's ordered visit history (source, time, click ids) but sent it to Salesforce only as an unused text blob. Now, after a form creates a Lead, the site posts the visitor's earlier visits to a Salesforce endpoint, which lands each visit as its own `Inbound_Contact__c` row with its own time, source and click id. The visit on which the form was submitted is left out, because the Lead flow (DDT-814) already logs it. The Salesforce half went live on 2026-09-25; the website half shipped switched off on 2026-10-26 and was switched on after a live test on 2026-10-30.

## 2. What was reported

- 2026-08-24 (DDT-821): website half, post the full ordered touch history with the form; Salesforce half, land it as one Inbound Contact row per touch with its own `Occurred_On__c`, source, click id and channel Web Form, and the raw history on one row only. Done when one form from a browser with three prior visits makes three rows in order, and the Lead's referral panel still shows its captured referral.
- 2026-09-25 (firm leadership): omit the submitting visit; it already lands as its own touch.

## 3. Scope

**In scope**

- Salesforce: `WebsiteTouchIntake` (a thin REST wrapper), `WebsiteTouchService`, their tests, a new field `Touch_Key__c`, additions to the website integration permission set, and a deploy note.
- Website: a session marker, the background post from the five form routes, click-id shape checks, a `prior_visits` count in the audit log line, and a feature switch.

**Out of scope**

- Using the first ad click when the last visit has none (DDT-820).
- Capturing more click-id types in the site's own cookies (DDT-1134).
- A referral question on the website forms; none of the five forms has one.

## 4. How it works (background)

The site's tracker adds a history entry only when the traffic source changes; back-to-back visits from the same source merge into one entry. So "one row per touch" means one row per change of source, which is what crediting needs. Salesforce buckets each source name through `Channel_Mapping__mdt`. DDT-811 ensures touch rows (which have no referrer type) can never replace a captured referral on the referral panel.

## 5. Root cause

Not applicable as a fault: new capability. The design risks the ticket named were that many rows per Lead could hijack a captured referral (closed by DDT-811 being live first), and that `Raw_Payload__c` is a long text area on a searchable object, so anything in it becomes searchable across the org.

## 6. Solution

**Salesforce half (PR #501, live 2026-09-25):**

- The service checks the Lead exists, accepts 1 to 50 touches with ISO 8601 times and a source, sorts them by time, and writes `Channel_Type__c = Web Form`, the Google click id to `GCLID__c`, and other click ids to `Click_Id_Other__c` with `Click_Id_Type__c`.
- `Raw_Payload__c` holds the typed touch list only (never form fields) and only on the last row.
- `Touch_Key__c`, a new unique External Id built from Lead, time and source; the service upserts on it, so a retry creates nothing. `Call_Session_Key__c` could not be reused because it is 18 characters and reserved for calls.
- The service enforces the integration user's object and field access before writing, and the permission set changes were additions only (create and read on Inbound Contact, no edit or delete).

**Website half (PR #293, #298, #321):**

- A first-party session marker (a gap over 30 minutes starts a new session) lets the site leave out the submitting session.
- After the Lead exists, the form routes post the earlier visits in the background, so the form never waits on or fails because of the post. Batches up to 50, capped at 100 per submission; retries on 422, 500, network errors and timeouts within a 25-second deadline; no retry on 400 or 403.
- Click ids are checked for shape before sending, and anything that looks like a phone number or email is dropped, so personal data cannot reach `Raw_Payload__c` through a crafted link.
- If no earlier visit is left, nothing is sent; the Lead flow's single row is then both first and last.
- Each audit line logs `prior_visits`, the denominator DDT-1361's gate needs.
- The post shipped behind a switch set to off; PR #321 turned it on.

## 7. Testing

- Salesforce tests: shuffled order, click ids, retry, bad input, a 50-touch bulk, access refusal, and referral safety (a Lead with a captured referral keeps its referral fields after three web touches, and conversion still carries it forward). Production check-only validation: 6 of 6 components, 54 of 54 tests.
- Live test, 2026-10-30, on a throwaway production Lead: a test browser made three visits (a Google ad landing with a test click id, a Facebook landing with a test click id, a newsletter link) more than an hour before submitting. Read back: three web rows in visit order; no row for the submitting visit; click ids on the right rows; Google Paid resolved to Paid Search and Facebook Ad to Paid Social; the raw history on one row only with no personal data; the first flag on the Google row; a referral added by hand through the panel after the rows landed still showed after refresh; no endpoint errors. The switch stayed on, and the test Lead was to be removed afterwards.

## 8. Step by step

1. Confirm the referral-protection fix (DDT-811) and the fields (DDT-813) are live in production.
2. Build the endpoint: validate, sort, map click ids, upsert on a unique per-touch key, raw history on one row only.
3. Agree the contract with the website side, including leaving out the submitting visit.
4. Ship the website post behind an off switch; add the audit count before switching on.
5. Run one real test submission with three prior visits from different sources, read the rows back, and test the referral panel.
6. Switch on; if the read-back fails, a one-line change turns it back off.

## 9. Rollback

Set the website switch back to off (a one-line change on the website side); the Salesforce endpoint then receives nothing. Rows already written stay, and can be found by their `Touch_Key__c` prefix. The test plan named this route; it was not needed.

## 10. Known limits

- A visit whose source is "Other" (for example an email newsletter) maps to Unknown; a mapping for email visits was left as a follow-up.
- A Lead that found the site and submitted in one sitting has no earlier visits, so it carries no web touch rows. This limits how high DDT-1361's coverage gate can go.
- Consecutive visits from one source are merged into a single row by the tracker.
- Referrals cannot be captured on the website forms today.

## 11. Reuse for similar issues

- Give every externally posted record a deterministic unique key and upsert on it, so the caller can retry freely.
- Post side data after the main record is saved and in the background, never in the user's form path.
- Strip and shape-check anything that will land in a searchable long-text field.
- Ship integration changes behind a switch, then prove them with one real, read-back test before switching on.
- Avoid double counting by agreeing which system logs the "current" event.
