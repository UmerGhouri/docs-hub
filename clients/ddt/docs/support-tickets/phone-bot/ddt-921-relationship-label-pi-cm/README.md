# Lost-call alert says Case Management instead of PI Case Management

| | |
|---|---|
| **Ticket** | DDT-921 |
| **Project** | Support tickets |
| **Status** | Live since 2026-09-04 |
| **Priority** | Medium |
| **Reported by** | Firm operations, 2026-09-02 |
| **Resolved by** | Support team |
| **Systems** | Phone Bot (Python), Salesforce (read-only queries) |
| **Area** | The "Relationship" line on Phone Bot lost-call alerts for emergency-contact callers |
| **Related** | None recorded |
| **Code** | Commits `4f6581c` and `70908fb` in the Phone Bot repo (`driverdefenseteam-sf-ops-bot`); no PR number recorded |

---

## 1. Summary

When an emergency contact of a PI client called and the call was lost, the alert said the caller "belongs with Case Management", the defense-side team. It should have said **PI Case Management**. The team name was **hard-coded** in the alert, so it could never say anything else. The alert now checks whether the client has a PI Matter and names the right team.

## 2. What was reported

On 2026-09-02 the operations lead reported a lost-call alert for a caller who was the emergency contact of a PI client. The call had correctly routed to PI, but the Relationship line read "emergency contact for (client) — belongs with Case Management". Her point: the routing was right, the wording was wrong, and the wording is what staff read to decide who follows up.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| `app/core/phonebot_lost_calls.py` | Both versions of the Relationship line use a team name instead of a hard-coded "Case Management" |
| `app/clients/salesforce.py` | The emergency-contact lookup also reports whether the client is a PI client (`_client_has_pi_matter`) |
| `tests/test_phonebot_pi_case_management.py` | New tests (section 7) |

**Out of scope**

- **Call routing.** It was already correct.
- **Salesforce configuration.** No org change; the bot only reads.

## 4. How it works (background)

```
Lost call from a number Salesforce knows
  → Phone Bot looks the caller up in Salesforce (cached for one hour)
  → if the caller is an emergency contact, it finds the client they belong to
  → the alert adds: "Relationship: emergency contact for (client) — belongs with (team)"
```

## 5. Root cause

The team name was a literal string in both branches of that line, the one used when the client's name is known and the one used when it is not. Nothing was being looked up.

Finding the right signal for "is this a PI client" took some checking, recorded so nobody repeats it. **None of these carry it:**

- **Account record types** have no PI or defense split.
- **The relationship record** only holds words like Spouse or Co-worker.
- **Opportunity Case Type** is entirely defense work (Petty, DUI, Aggravated Speeding, SOS hearings). The client in the report had no Opportunity at all.

**What does carry it:** the **Matter** (`Matter__c`), linked to the client account by `Account__c`. Every Matter in the org is PI. The first fix used the PI incident object (`PI_Incident__c`) instead; it was corrected the next day because there are more incidents (342) than Matters (296). An incident can be logged for a PI enquiry that was never retained, so an incident alone does not make someone a client.

## 6. Solution

```python
_DEFAULT_CMT_TEAM = "Case Management"
_PI_CMT_TEAM = "PI Case Management"

def _client_has_pi_matter(sf, account_id) -> bool:
    for soql in (
        f"SELECT Id FROM Matter__c WHERE Account__c = '{escaped}' LIMIT 1",
        f"SELECT Id FROM PI_Incident__c WHERE Client__c = '{escaped}' LIMIT 1",
    ):
        try:
            if sf.query(soql).get("records"):
                return True
        except SalesforceError:
            return False
    return False
```

The Relationship line then reads "belongs with PI Case Management" when this returns true, and "belongs with Case Management" otherwise.

Why it's done this way:

- **Matter first, incident as a fallback.** The Matter is what says the firm took the case. The incident query only runs when there is no Matter, so it is usually one query.
- **Both branches fixed,** including the line used when the client's name can't be resolved.
- **Failure falls back to "Case Management"** rather than dropping the line. A missing lookup must never cost the alert.
- **Inside the existing one-hour cache,** so it costs about one extra query per emergency-contact caller per hour.

## 7. Testing

Fifteen new tests across the two commits; 880 passing at closure. The second commit's six cover the signal itself: a Matter settles it in one query; an incident alone still counts; the Matter is checked first; a query failure reports the default team instead of raising.

**Live check:** re-run against the reported caller, the line now reads "emergency contact for (client) (Co-worker) — belongs with PI Case Management".

**Not covered:** a non-PI emergency-contact caller checked live. The tests cover the default branch.

## 8. Step by step

1. Find where the alert text is built and check whether the team name is looked up or written in.
2. Decide which Salesforce record actually says "PI client". Check record types, relationship roles and Opportunity case types before assuming; here only the Matter did.
3. Add a lookup that checks the Matter first and the PI incident second, and returns false on any failure.
4. Use its result in both branches of the Relationship line.
5. Add tests for each path, including the failure path.
6. Re-run against the reported caller and check the line.
7. Deploy the Phone Bot.

## 9. Rollback

Revert the two commits and redeploy the Phone Bot. Safe; PI emergency contacts would be sent to the defense team again.

## 10. Known limits

- **"Every Matter is PI" is a fact about the org today.** If Matters are ever used for another practice area, this check will call those clients PI.
- **A PI enquiry that was never retained and has no Matter** still counts as PI if it has an incident. That is the fallback's deliberate trade-off.
- **If the Matter query fails,** the bot reports the default team without trying the incident query.

## 11. Reuse for similar issues

**How to recognise one:** an alert or message always shows the same team, label or value no matter the input.

- Look for a hard-coded string before looking for bad data.
- When you need a "which practice area" signal, check which record is authoritative. Here Opportunities are defense only and Matters are PI only.
- Prefer the record that means "retained" over one that means "enquired".
- Enrichment in an alert should degrade to a safe default, never remove the line or block the alert.
