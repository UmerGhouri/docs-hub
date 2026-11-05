# Text Messages panel on the lead page

| | |
|---|---|
| **Status** | Live since 2026-10-20 (rate-limit redesign 2026-10-21) |
| **Requested by** | Client admin, 2026-10-20 |
| **Built by** | Support team |
| **Systems** | Salesforce (LWC, Apex, External/Named Credentials, Platform Cache), RingCentral API |
| **Area** | Lead record page |
| **Change log rows** | 55–69 (and 70–74 for the rate-limit fix) |
| **Source** | `source/force-app` (in the private copy) (production build of 2026-10-21) |

---

## 1. Summary

The team couldn't see a lead's full text conversation in Salesforce. SMS tasks are only logged while the line owner has the RingCentral panel open (see [the auto-log issue](../2026-10-16-ringcentral-sms-task-autolog/README.md)), and texts from other lines or from Mia were often missing.

We built a **Text Messages** panel on the Lead page that reads texts **live from RingCentral** for the lead's phone numbers, from **all** company lines, including photos. Nothing is stored in Salesforce.

## 2. Client request

> "We need to make sure all SMS with the lead are always visible in the lead record."

_Screenshot in the private copy: 01-client-request.png (Client request)._

## 3. Scope

**In scope**

| Component | Type |
|---|---|
| `rcSmsConversation` | LWC, Lead record page, right column above Activity |
| `RingCentralSmsController` (+ `RingCentralSmsControllerTest`) | Apex (`getThread(leadId, mode)`, `getAttachment(...)`) |
| `RingCentral_SMS` | External Credential (custom auth, principal `RC_Principal` with parameters ClientId / ClientSecret / JWT) + Named Credential → `https://platform.ringcentral.com` |
| `RingCentral_Media` | Named Credential → media host (photos/files are only served there); reuses the same login |
| `RingCentral` | Platform Cache partition (Org, 5 MB) |
| `RingCentral_SMS_Viewer` | Permission set: Apex class + credential principal access |

**Out of scope:** sending texts (read-only); RingCentral's own SMS task logging (unchanged).

## 4. Design

### 4.1 Authentication
- RingCentral developer app **"Salesforce SMS Viewer"**: private, **JWT** grant, scopes **Read Messages** and **Read Accounts**.
- The JWT belongs to a **super admin** (the developer), so the app can read **every** user's mailbox (9 SMS lines).
- Client ID, Client Secret and JWT are pasted into **Setup → Named Credentials → External Credentials → RingCentral_SMS → RC_Principal**. **They are not in the code or this repo.**
- Apex calls `callout:RingCentral_SMS/restapi/oauth/token` with grant type `jwt-bearer`, building the Basic header from the credential merge fields.

_Screenshot in the private copy: 02-ringcentral-developer-apps.png (RingCentral developer console, where the app is registered (Apps → Register App))._

### 4.2 What the panel shows
- Chat bubbles with date dividers, sender name + time, failed-text marker.
- **Sender = the RingCentral user who owns the line.** Mia's texts show as the client admin (main line); the client chose this over a "Mia" label.
- **Line filter**: "All lines" or one line, shown as name + number.
- **Photos (MMS)**: newest 6 load automatically, older on click; click to enlarge and download. PDFs and other files show as download links. Max 1.6 MB per file. The controller checks the file belongs to this lead's conversation first.
- Texts from **30 days before the lead was created** onwards, on `MobilePhone` and `Phone`.

_Screenshot in the private copy: 03-panel-header.png (Panel header)._ _Screenshot in the private copy: 04-line-filter.png (Line filter)._

### 4.3 Refresh behaviour (current build)
- The panel checks for new texts **every 10 seconds, only while the lead is on screen and the tab is visible** (`visibilitychange`), and on return to the tab.
- These checks read the **shared cache**, not RingCentral. RingCentral is only asked by the shared sync. Details in the [rate-limit doc](../2026-10-21-text-panel-rate-limit/README.md).
- Footer: "Updated h:mm:ss AM"; a **Refresh** button forces an update.

## 5. Build history

| Date | Change |
|---|---|
| 10-20 | v1: reads only the JWT owner's mailbox; manual refresh |
| 10-20 | Client admin's texts missing → read **every** user's mailbox (super admin JWT); line names cached 24 h |
| 10-20 | Auto-update every 30 s; count and "auto every 30s" removed from the display |
| 10-20 | Photos and files via `RingCentral_Media` |
| 10-20 | Production: deployed for the developer only (visibility rule), then opened to all 8 active users |
| 10-21 | Rate-limit redesign (shared cache, budget); "Updated" shows seconds |

## 6. Deploy

```
sf project deploy validate --source-dir force-app --target-org pacific --test-level RunSpecifiedTests --tests RingCentralSmsControllerTest
sf project deploy quick --job-id <validation id> --target-org pacific
```
After deploying:
1. **Setup → External Credentials → RingCentral_SMS → RC_Principal:** enter ClientId, ClientSecret, JWT (first deploy only).
2. Assign permission set **RingCentral_SMS_Viewer** to every user who should see the panel. **New users need it too.**
3. Lightning App Builder → Lead record page → drag **Text Messages (RingCentral)** into the right column → Save / Activate.

Production deploys: `[deploy id]` (10-20, 8/8 tests, 92%), `[deploy id]` (10-21, 11/11 tests, 91%).

## 7. Testing
- A real lead loads 11 texts across three loan officers' lines, plus 4 replies from the lead.
- Two-way: texts sent from a phone to the sandbox line appear in the panel within the refresh interval.
- MMS: an image shows "queued" until RingCentral finishes delivering it, then shows normally.

## 8. Rollback
Redeploy `Lead_record_page.prod-before-sms-panel.xml` (removes the panel from the page), or remove the permission set from users. The components can stay; without the page entry nothing runs.

## 9. Known limits
- **One RingCentral login for all users and both orgs**, with about 50 requests a minute. See the rate-limit doc.
- **Cache item limit 100 KB**: about 190 texts per phone number. A lead with more would currently error. Idea: keep only the newest 150.
- **Deploys clear the cache.** The first opens after a deploy are slower for a few minutes.
- **If the JWT owner's RingCentral user is deleted or demoted, the panel stops.** Generate a new JWT from another super admin and paste it into the External Credential.

## 10. Reuse for similar issues
- **Calling an outside API from Salesforce:** External Credential (custom parameters) + Named Credential, so secrets never sit in code.
- **Many users reading the same rate-limited API:** read from a shared Org Platform Cache and let one sync fill it (see the rate-limit doc).
- **Pilot first:** release to one user with a page visibility rule plus the permission set, then open it to everyone.
