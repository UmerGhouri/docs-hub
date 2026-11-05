# Calls in Activity but missing from the new Call Log

| | |
|---|---|
| **Ticket** | DDT-752 |
| **Project** | Support tickets |
| **Status** | Live since 2026-08-19 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-08-17 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (Apex, LWC, custom metadata, Lightning record pages), RingCentral |
| **Area** | Client (Person Account) record page: the Call Log related list |
| **Related** | DDT-643 (root-cause fix for unlinked call sessions), DDT-748 (Days Since Last Outreach) |
| **Code** | PR #193 in the ddt-salesforce repo; PR #201 recorded the page placement |

---

## 1. Summary

A user's calls to a client showed in the client's Activity but not in the new Call Log. The Call Log lists `Call_Session__c` records linked to the client, and many RingCentral call sessions are saved without that link. The workaround was a "Link RC Calls" button; the user asked not to have to click it every time.

We added a hidden component to the Client page that runs a callout-free version of the linking pass when the page opens and refreshes the Call Log if it linked anything. It only fills empty links and never moves a call already linked to another client. This is a fallback: it repairs clients someone opens, not the underlying linking gap.

## 2. What was reported

Calls made that day to one client appeared in Activity but not on the new Call Log. The support team first showed the "Link RC Calls" button, which tags the missing calls to the client. The user asked whether there was a way not to click it every time, and said he wanted call history to live on the client or matter record.

In production on 2026-08-19, **14,919 of 42,450** call sessions (35.1%) had no client (`Account__c`) set, so a Client page could not show them.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `LinkRcCallsAction` | Apex class | Extracted `resolvePhones()`; new `autoLinkForRecord()`; a fill-only mode for the page-load path |
| `RC_Settings` | Apex class | New `autoLinkOnPageLoad()` reading the kill switch |
| `RC_Setting__mdt.Auto_Link_Disabled__c` | Custom metadata field (Checkbox) | New; kill switch, unchecked by default |
| `linkRcCallsAuto` | LWC | New headless component for Account record pages |
| `LinkRcCallsActionTest`, `RC_SettingsTest` | Apex tests | 10 and 2 new tests |
| `linkRcCallsAuto/__tests__` | Jest | 8 new tests |
| Client Lightning record pages | Flexipages | Component placed on all five assigned pages |
| RC Setting page layout | Layout | Kill switch added so admins can see it |

**Out of scope**

- **The root cause.** Sessions are left unlinked when created in bulk (see section 5). The fix for that is DDT-643, a separate change. Reports, list views, Days Since Last Outreach and the historical backlog stay wrong until it lands.
- **The RingCentral capture pipeline and triggers.** Unchanged.
- **The manual "Link RC Calls" button.** Unchanged; it still covers calls RingCentral never delivered as sessions.
- **A team-wide view of calls**, which the user also asked about. Not part of this change.

## 4. How it works (background)

```
RingCentral call
  -> RC_SessionBuilder creates Call_Session__c records in bulk
  -> RC_SessionMatcher tries to link each session to a Client / Contact / Lead by phone
  -> Client page "Call Log" = the Call_Sessions__r related list, keyed on Call_Session__c.Account__c
Activity timeline = Task records, which are created separately
```

A Client in this org is a Person Account. A session with no `Account__c` is in Activity (through the Task) but not in the Call Log.

## 5. Root cause

The two views read different records. Activity shows Tasks; the Call Log shows `Call_Session__c` records only when `Account__c` is set.

Why so many sessions are unlinked: `RC_SessionMatcher` spends one SOSL query per session, SOSL is capped at 20 per transaction, and `RC_SessionBuilder` saves 50 to 169 sessions per transaction. So the matcher gives up partway through every large batch and leaves `Linked_Party_Type__c` **null**. The nightly reconcile only looks at sessions marked `'None'`, so it never sees them. Measured over 14 days for DDT-643: 3% unlinked in batches of 2 to 19, **50% unlinked in batches of 50 or more**.

The ticket also asked whether the cause was a display filter on the Call Log or missing data. It was missing links, not a filter.

## 6. Solution

### 6.1 A separate, callout-free Apex entry point

The button's `linkForRecord` pulls 48 hours of RingCentral call log per click. Running that on every Client page view would invite rate-limit errors, which this org has already had. `autoLinkForRecord` uses SOQL and one bulk update only. It links sessions that **exist** but are unlinked (the user's case); it cannot create a session RingCentral never delivered. The test `auto_makesZeroCallouts` stops anyone later routing it through `linkForRecord`.

### 6.2 A separate headless LWC

`linkRcCallsAuto` runs from `connectedCallback()` on Account record pages and shows nothing. It is a separate bundle from the button because the two want opposite behaviour: the button is loud, this must be silent.

### 6.3 Refresh the Call Log only when something changed

The related list renders at the same time as the component, so on the load where linking happens the Call Log is already on screen and still empty. The component fires a `RefreshEvent` **only** when it linked at least one session. The refresh re-runs the component, the second pass links nothing, and no further refresh fires, so it cannot loop.

### 6.4 Read-only users do nothing

`autoLinkForRecord` checks that the user can update `Call_Session__c` first and returns `'no-access'` without querying or writing. Otherwise every page a read-only user opened would quietly pile up failed updates.

### 6.5 The kill switch is stored inverted

The field is `Auto_Link_Disabled__c`, not "enabled". A new checkbox on an existing custom metadata record reads `false`, and a default of `true` is not applied to existing records. A positive "enabled" flag would have shipped the feature switched off; the first sandbox run caught this with 8 failing tests. Leave the box unchecked for normal use; check it to turn the feature off org-wide without a deploy.

### 6.6 The page-load path only fills blanks

Added in review. Shared household and employer numbers are common, and in production 957 Client accounts (2.8%) share a normalised phone with another client. If the page-load path re-pointed links like the button does, just **viewing** client B would move client A's calls to B, and the active Last Outreach flow on `Call_Session__c` would move the outreach date with them. Two people viewing each other's clients would swap them back and forth with nothing on screen to show it. So:

- **Button:** unchanged. A link pointing at another client is moved to this one, because a person asked for it.
- **Page load:** fills empty links only. `auto_sharedPhone_doesNotRepointExistingClient` and `button_sharedPhone_stillRepoints` pin the difference.

### 6.7 Placement on every assigned page

The component was first placed only on the org default Client page. App-default page assignments override the org default, so most users were on pages without it. It was then added, in production, to all five Client record pages that are actually assigned; the six unassigned pages were left alone. The pages were retrieved from production and checked for the component, rather than assuming the edit took.

## 7. Testing

- **Apex:** production validation passed with 55 of 55 tests and 6 of 6 components, using `LinkRcCallsActionTest`, `RC_SettingsTest` and `LinkRcCallsBuildQueueableTest`. The new tests cover zero callouts, idempotency, read-only users, fill-only versus button behaviour on a shared phone, and a 200-session bulk case to confirm queries do not grow with session count.
- **Jest:** 8 of 8 for the new bundle, including "does not dispatch RefreshEvent when nothing changed".
- **Sandbox end-to-end:** created an unlinked session, pointed a client's phone at it, opened the client page with no click. `Account__c` was set to the client and the last-modified user was the person who opened the page. Reloading the page changed nothing (no second write).
- **Live:** the support team confirmed in the ticket that calls now attach to the client when the page opens, with no button click.

Two findings from validation worth knowing:

- Six older tests failed in one sandbox and passed in production because that sandbox had the unmerged DDT-643 trigger deployed, which links sessions as they are created. Those tests will need updating when DDT-643 merges.
- `System.runAs` counts as a DML statement, so a DML-count assertion around a `runAs` block reads 1, not 0. The test was changed to rely on returned counts instead.

**Not covered:** page-load time before and after on a client with a long call history.

## 8. Step by step

1. Confirm the gap: for the client, compare Tasks in Activity with `Call_Session__c` records where `Account__c` is the client. Look for sessions whose `External_Party_Number__c` matches the client's phone but whose `Account__c`, `Contact__c` and `Lead__c` are blank.
2. Short-term for one client: click "Link RC Calls" on the client page.
3. To deploy the automatic version: deploy the six components in section 3, validated with the three test classes in section 7.
4. Confirm the kill switch is off by query, since it is not visible until the layout is updated: `SELECT DeveloperName, Auto_Link_Disabled__c FROM RC_Setting__mdt WHERE DeveloperName = 'Default'` should return `false`.
5. Add **Auto Link Disabled** to the RC Setting page layout in Setup. Do not deploy the layout file from git; it does not match production.
6. Find which Client record pages are actually assigned (org default, app default, by profile). Add `linkRcCallsAuto` to each one in App Builder, starting from production's current version of the page.
7. Retrieve the pages afterwards and confirm the component is on them.
8. Open a client with known unlinked calls and confirm they appear without a click.

## 9. Rollback

- **Fastest:** check `Auto_Link_Disabled__c` on the Default RC Setting record. The component stops linking at once, with no deploy.
- **Full:** remove `linkRcCallsAuto` from the five pages, then redeploy the previous Apex. Links already written stay, and they are correct links.

## 10. Known limits

- **Only clients someone opens are repaired.** Reports, list views and Days Since Last Outreach still miss unlinked sessions until DDT-643's fix lands.
- **Calls RingCentral never delivered** as sessions still need the manual button.
- **The phone match uses a leading-wildcard `LIKE`,** which scans the whole call session table on each page load. If Client pages slow down, the planned fix is an indexed, normalised phone field on `Call_Session__c` (needs a backfill).
- **Linking from the Client page fills `Account__c` only.** `Contact__c` and `Linked_Party_Type__c` are left as they were, so a report filtered on `Linked_Party_Type__c = 'Contact'` will not count these rows. The manual button behaves the same way.

## 11. Reuse for similar issues

**How to recognise one:** "it's in Activity but not in the Call Log" (or any two views of calls that disagree).

- Find which record each view reads. Activity reads Tasks; the Call Log reads `Call_Session__c` by `Account__c`.
- Count the sessions with no party linked before blaming the component.
- Anything that runs on page load must be cheap: no callouts, bulk DML, and a no-op on a second run.
- An automatic path must not override a choice an earlier link made. Fill blanks; leave re-pointing to a deliberate click.
- Name a new checkbox kill switch in the "disabled" direction, so its default leaves the feature on.
- Place a page component on every page that is actually assigned, not just the org default.
