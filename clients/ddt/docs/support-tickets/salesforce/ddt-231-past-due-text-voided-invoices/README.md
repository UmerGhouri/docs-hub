# Past-due text firing on voided invoices

| | |
|---|---|
| **Ticket** | DDT-231 |
| **Project** | Support tickets |
| **Status** | Live since 2026-06-25 |
| **Priority** | High |
| **Reported by** | Firm staff, 2026-06-25 |
| **Resolved by** | Support team |
| **Systems** | Salesforce (scheduled Flow), fw1 billing invoices, Heymarket SMS |
| **Area** | Collections: the automatic "2 days past due" text from the attorney to the client |
| **Related** | None |
| **Code** | None. The flow was edited in Flow Builder in production. Its new version reached the repo in the 2026-07-03 prod backfill. |

---

## 1. Summary

A daily scheduled flow texts a client, in the attorney's name, when an invoice is two days past due. It also texted clients whose invoice had been **voided**, because voiding an invoice does not clear its balance or its overdue flag, and the flow never checked the void flag. The flow's start condition now requires the invoice not to be voided.

## 2. What was reported

On 2026-06-25 a client received the 2-days-past-due text for an invoice that had been voided, so they were told they owed money they did not owe. The ticket gave one example court case. How many other clients received the same text is not recorded.

## 3. Scope

**In scope**

| Component | Type | Change |
|---|---|---|
| `Past_Due_2_Day_SMS_From_Attorney` | Scheduled flow on `fw1__Invoice__c` | Start condition gains `fw1__Is_Voided__c = false` |

**Out of scope**

- **The message text and the send action.** Unchanged.
- **Other collection messages** (demand-letter flows and similar). The ticket named only this text, and no other flow was changed.

## 4. How it works (background)

```
Scheduled flow, daily, on fw1__Invoice__c records that match the start condition:
  fw1__Is_Overdue__c = true
  fw1__Balance_Due__c > 0
  fw1__Opportunity__c is not null
  Is_2_Days_Past_Due__c = true          (formula: fw1__Due_Date__c = TODAY() - 2)
  -> Get the Court Case (Opportunity): Hire_Status__c = Hired, Primary_Attorney__c not null
  -> Get the client Account, the attorney User, and the attorney's Heymarket team mapping
  -> Decisions: hired with a primary attorney? phone present? team mapping found?
  -> Heymarket "send message" action, from the attorney's default inbox, to the client
```

## 5. Root cause

The start condition checked that the invoice was overdue and had a balance, but **not that it was still a live invoice**. A voided fw1 invoice keeps its balance due and its overdue flag, so on the day its due date was two days back it matched like any other.

**Evidence.** Voided invoices still look overdue and unpaid. On 2026-11-05 this returned **877**:

```sql
SELECT COUNT(Id)
FROM fw1__Invoice__c
WHERE fw1__Is_Voided__c = true
  AND fw1__Balance_Due__c > 0
  AND fw1__Is_Overdue__c = true
```

Every one of those would pass the old start condition on the right day.

## 6. Solution

Two filters were added to the flow's start condition:

```xml
<filters>
    <field>fw1__Is_Stop_Collection__c</field>
    <operator>EqualTo</operator>
    <value><booleanValue>false</booleanValue></value>
</filters>
<filters>
    <field>fw1__Is_Voided__c</field>
    <operator>EqualTo</operator>
    <value><booleanValue>false</booleanValue></value>
</filters>
```

The support team's note on the ticket describes only the void check. The stop-collection filter appears in the same diff, between the 2026-05-22 snapshot of the flow and the 2026-07-03 backfill of production. Whether it was added in this change or separately is **not recorded**.

**Why filter in the start condition and not in a decision later:** records that fail the start condition never start an interview, so a voided invoice costs nothing and cannot reach the send action by any path.

Production shows the flow last modified on **2026-06-25 at 15:08 UTC**, four minutes before the support team posted that the change was done.

## 7. Testing

**Not recorded in the ticket.** The ticket asked for a check against the example case and a regression check that genuine past-due invoices still trigger the text; neither result is recorded.

**Not covered:** there is no automated test for this flow in the repo.

## 8. Step by step

1. **Find the flow.** Setup → Flows → **Past Due 2-Day SMS From Attorney**.
2. **Open the active version** and click the Start element.
3. **Add the condition** `Is Voided` equals `False` (and `Is Stop Collection` equals `False` if it is missing), keeping the condition logic as **All conditions are met**.
4. **Save as a new version and activate it.**
5. **Check the scope** with the query in section 5, and with the same query using `fw1__Is_Voided__c = false` to see what still qualifies.
6. **Record it in git.** Retrieve the flow from production and open a PR the same day, so the repo carries what production runs.

## 9. Rollback

In Setup → Flows, activate the previous version. Immediate, but it brings the bug back.

## 10. Known limits

- **Other "dead" invoice states.** The ticket also asked about paid and cancelled invoices. A paid invoice has no balance due, so the existing condition already excludes it. Any other state that leaves a balance and the overdue flag set will still be texted.
- **Other collection automations.** Demand-letter flows on invoices were not reviewed under this ticket; each needs its own void check.
- **The fix was made directly in production** and reached git only through a bulk backfill. Rebuild from the repo's current version, not from an older copy.

## 11. Reuse for similar issues

**How to recognise one:** a client is contacted about money they do not owe, or a reminder fires on a record that was cancelled, voided or reversed.

- **Find which flags the managed package leaves set** when a record is voided or cancelled. Do not assume voiding zeroes the balance.
- **Put the "is it still live" check in the start condition** of a scheduled or record-triggered flow, so excluded records never start.
- **Count the affected population** with a query like the one in section 5 before and after the change.
- **Check the sibling automations** that read the same object for the same missing condition.
