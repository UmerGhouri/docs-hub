# Text Messages panel rate-limit fix

| | |
|---|---|
| **Status** | Live since 2026-10-21 |
| **Reported by** | Pacific team, 2026-10-20 (panel showing a red error) |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, Platform Cache), RingCentral API |
| **Change log rows** | 70–74 |
| **Source** | `source/force-app` (in the private copy) |

---

## 1. Summary

The day the panel was opened to all users, it started failing with RingCentral **429 "Request rate exceeded"**. Every open lead tab was asking RingCentral directly, and all users and both orgs share one login that allows about 50 requests a minute.

We redesigned it so **panels never ask RingCentral**. One shared sync fills an org-wide cache, the panels read the cache, and the sync keeps itself under a budget of **40 requests a minute**.

## 2. Client report

_Screenshot in the private copy: 01-rate-limit-error.png (Rate limit error in the panel)._

## 3. Root cause (numbers)

| | Before |
|---|---|
| Requests per open lead tab | ~8 a minute (9 lines checked every 30 s, staggered) |
| On opening a lead | +9 (history from each line) |
| Users × tabs | 8 users with several tabs → far above ~50 a minute |
| Shared by | All users **and** both orgs (one JWT login) |

RingCentral counts per login, so one busy user could block everyone, including the sandbox.

## 4. Solution

Everything is in `RingCentralSmsController`, with shared state in Org cache `local.RingCentral` (raised from 1 MB to **5 MB**).

| Piece | How it works |
|---|---|
| **Shared texts per phone number** | Texts are stored in the cache under the lead's phone number (24 h). Every user and tab reads the same copy |
| **New texts: message-sync per line** | RingCentral `message-sync` (full sync once, then "only what's new since the last token"), **at most every 30 s per line, for all users together**. New texts are filed under each phone number |
| **Which lines every 30 s** | Only lines that matter to leads on screen: the **main line**, the lead's owner, and lines already texting the lead (plus the JWT owner's line in the sandbox). **All other lines every 5 minutes**, only with budget to spare |
| **History** | Older texts loaded **once per number per day**, **line by line**, so texts appear as they arrive; Refresh can reload history at most every 10 minutes |
| **Budget** | A per-minute request counter in the cache; stop at **40/min** (RingCentral blocks at ~50). Same budget in both orgs, because the sandbox is rarely used |
| **On 429** | Back off and show a small "RingCentral busy" note with the texts already loaded, not a red error |
| **Panel polling** | Every 10 s, only while the lead is on screen. **These polls make 0 RingCentral requests**; they read the cache |
| **Directory** | User / line list cached 24 h |

**Result:** about 18 requests a minute in total, plus up to 9 per newly opened lead (history), instead of ~8 per open tab.

_Screenshot in the private copy: 02-updated-with-seconds.png (Updated time with seconds)._

## 5. Why not the RingCentral bulk / batch API?
- RingCentral's batch endpoints still count against the same per-login limit, and they don't work across different users' mailboxes in one request.
- The real problem was **many tabs asking for the same data**. A shared cache fixes that; batching wouldn't.

## 6. Testing
- Apex tests use an in-memory map instead of Platform Cache (`memoryCache` when `Test.isRunningTest()`), with mock callouts for token, message-store, message-sync and 429.
- Validation: 11/11 tests, 91% coverage. Production quick deploy `[deploy id]`.
- Live: several leads open in several tabs. No 429; the footer time ticks every 10 s.

## 7. Rollback
Redeploy the backup `smsprod/backup/prod-before-shared-cache` (the 10-20 build). It will hit 429 again with many users.

## 8. Known limits
- **First open of a number is slower** when the budget is busy, because history loads line by line.
- **Deploys and cache eviction reset the sync.** Expect a few slow minutes after a deploy.
- **Ideas not done yet:** a 48 h cache time (for leads opened every other day), a morning warm-up for leads due today, and keeping only the newest 150 texts per number (cache item limit 100 KB).

## 9. Reuse for similar issues
- **Rate limit per login shared by many users** → shared Org cache plus one rate-budgeted sync. Never let each browser call the API.
- **Check how the vendor counts limits** (per app, per user, per login) before choosing a design. RingCentral counts per login, so the sandbox and production compete.
- **Show "busy, showing what we have"** instead of an error when a soft limit is hit.
