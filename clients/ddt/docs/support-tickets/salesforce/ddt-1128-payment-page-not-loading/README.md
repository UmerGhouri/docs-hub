# Payment page not loading

| | |
|---|---|
| **Ticket** | DDT-1128 |
| **Project** | Support tickets |
| **Status** | Done 2026-09-28 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-09-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Visualforce `Payment` page from the `fw1` payments package), web browser |
| **Area** | Taking client payments |
| **Related** | None recorded |
| **Code** | None (no change made) |

---

## 1. Summary

A staff member opening the Payment page in Salesforce got a browser error saying the page might be down or moved, so they could not take a payment. The support team checked and found the page working; the error was a brief browser problem. No change was made, and the advice was to hard-refresh the page if it happens again.

## 2. What was reported

On 2026-09-25 a staff member reported that opening the Payment page showed the browser message "webpage might be temporarily down or moved permanently" instead of the payment form. They could not collect a client payment. One user reported it; the ticket asked to confirm whether anyone else was blocked.

## 3. Scope

**In scope:** checking that the `Payment` Visualforce page loads.

**Out of scope:** any change to the page, the button that opens it, or domain settings. None was needed.

## 4. How it works (background)

```
Staff click the payment button in Salesforce (Lightning)
  -> Lightning opens the Visualforce page "Payment" in a frame
  -> the page is served from the managed payments package's own Visualforce web address
  -> the payment form loads
```

Managed-package Visualforce pages are served from an address that includes the package's namespace (here `fw1`), separate from the main Lightning address. That is normal.

## 5. Root cause

A brief browser glitch, according to the support team's reply. The page was working when checked.

**The ticket's first guess did not hold.** It suspected the `fw1` part of the page's address meant the link pointed at a sandbox or the wrong domain. `fw1` is the namespace of the managed payments package already installed in production, and the page loaded normally from that address.

Not recorded in the ticket: what the browser glitch was, or whether other users saw it.

## 6. Solution

No change. The support team confirmed the page loads and told the user to press **Ctrl+Shift+R** (hard refresh, which reloads the page without the browser's cached copy) if it happens again.

## 7. Testing

The support team opened the Payment page and found it working on 2026-09-28. Not recorded in the ticket whether the user confirmed a successful payment afterwards.

## 8. Step by step

1. **Open the Payment page yourself** from Salesforce. If it loads, the page and its address are fine.
2. **Check the page's address** before suspecting a misconfiguration: a `fw1` namespace in it is expected for this managed package.
3. **Tell the user** the page is working and to press Ctrl+Shift+R (hard refresh) if the error comes back.
4. **If it keeps failing**, reopen the ticket. Not recorded in the ticket what further checks would follow.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- The cause was not identified beyond "browser glitch", so it may recur.
- A hard refresh fixes cached-page problems only; it will not help if a network filter blocks the page's address.

## 11. Reuse for similar issues

**How to recognise one:** one user gets a generic "page down or moved" browser error on a page that works for support.

- **Reproduce it first** before changing anything.
- **An unfamiliar address is not proof of misconfiguration.** Managed-package pages use their own namespaced address.
- **Start with the browser:** a hard refresh reloads the page without the cached copy.
