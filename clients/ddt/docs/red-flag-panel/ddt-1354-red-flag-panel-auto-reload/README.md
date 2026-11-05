# Red Flag Panel shows stale flags until a page reload


| | |
|---|---|
| **Ticket** | DDT-1354 |
| **Project** | Red Flag Panel |
| **Status** | Live since 2026-10-20 (reached production) |
| **Priority** | Medium |
| **Reported by** | Firm leadership, 2026-10-16 (after a staff report in DDT-1346) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (LWC) |
| **Area** | Matter red flags: record page |
| **Related** | DDT-794, DDT-1346 |
| **Code** | PR #757 in the ddt-salesforce repo |


---


## 1. Summary

After a payment was made and a court appearance added on a matter, the panel kept showing "There has been no payment made yet" and "No Court Appearances exist" until the page was reloaded. The data was already correct; the panel was showing a cached result. The panel now reloads its flags on its own: when the user returns to the tab, when the record page refreshes, when the matter record changes, and from a new refresh button next to a "Checked" time.

## 2. What was reported

- 2026-10-16: DDT-1346 was opened because `NO_PAYMENT_YET` and `NO_APPEARANCES` stayed on a matter after the payment and the appearance. A reload cleared them. Make the panel reload by itself, show how fresh the flags are, and add a refresh button.

## 3. Scope

**In scope**

- The `redFlagPanel` component and its Jest tests.

**Out of scope**

- Live push of changes (change data capture or streaming): more build for a problem of one user on a stale page.
- Apex, formula, permission set or scheduled job changes.

## 4. How it works (background)

The panel reads flags through a cacheable wire. Before this change it only reloaded after its own inline Fix save. A payment happens on the invoice screens, often in another tab, and appearances are often added from the related list, so the panel never heard about either change.

## 5. Root cause

In `redFlagPanel.js`, `refreshApex` only ran after the panel's own save. On the reported matter, the payment and the appearance were both in the data well before the report; the formula was already clear, and only the panel's cached result was stale.

## 6. Solution

| Trigger | How |
| -- | -- |
| Returning to the tab | `visibilitychange`, reload when visible. Switches under 2 seconds apart share one reload; a reload already running is joined, never doubled |
| The record page's refresh | `registerRefreshHandler` from `lightning/refresh`, so a standard Save, New or refresh also reloads the panel |
| The matter changing | `getRecord` on the codes field and last-modified date; the first value is ignored, a later different one reloads |
| The user | Refresh icon in the header beside "Checked 2:14 PM", in the user's Salesforce time zone |

An inline Fix save counts as a reload. Both listeners are removed in `disconnectedCallback`. A document-level listener was used for tab visibility because there is no platform replacement for it, and two existing components already use the same pattern.

## 7. Testing

- 7 new Jest tests, 20 of 20 pass. Ten pieces were removed one at a time (listener, 2-second gap, handler binding, host element, running-reload join, required field, record-change reload, button, both removals); every removal fails at least one test.
- Check-only to production: the bundle compiles; controller test 21 of 21. That runs no browser code.
- Live in production on 2026-10-19 on a test matter, changing the client in a second tab: returning to the tab cleared "How Heard" on its own (6 to 5 flags, Checked time moved); the refresh button cleared "Language" (5 to 4). A non-admin user saw the panel, Checked time and button. Test values were put back.
- Not tested live: a real payment, because it would go through the live payment system (the tab-return test covers the same reload path); and a court appearance added from the related list. Firm leadership confirmed and closed the ticket with DDT-1346, to be reopened if staff report a stale panel on that path.

## 8. Step by step

1. Confirm the data is right and only the component is stale.
2. List where the data changes: other tabs, related lists, standard save.
3. Add a reload for each: tab visibility, the page refresh handler, a record wire.
4. Debounce and join reloads so they never stack.
5. Show when the data was last checked and give a manual refresh.
6. Remove listeners on disconnect; test each trigger and prove each test can fail.
7. Check live on a real record page, not just a validate deploy.

## 9. Rollback

Redeploy `redFlagPanel` from the previous commit.

## 10. Known limits

- Switching console sub-tabs inside one browser tab does not fire the visibility event; the refresh button covers that.
- Adding an appearance from the related list was not seen live; the tab return and the refresh button are the fallback.
- The "Checked" line only shows while there are flags, because the panel renders nothing when clean.

## 11. Reuse for similar issues

- Signs of the same problem: a component built on a cacheable wire shows old values until a hard reload, while the record data is correct.
- Combine tab visibility, the page refresh handler and a record wire, and add a visible "checked at" with a refresh button.
- Always clean up document listeners and refresh handlers when the component is removed.
