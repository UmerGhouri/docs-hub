# Support bot auto-assigns IT tickets

| | |
|---|---|
| **Ticket** | DDT-255 |
| **Project** | Support tickets |
| **Status** | Live since 2026-07-01 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-07-01 |
| **Resolved by** | Support team |
| **Systems** | Slack support bot, Linear, Claude (ticket classification) |
| **Area** | Turning Slack support requests into Linear tickets |
| **Related** | DDT-100 (syncing Slack message edits to Linear, same bot), DDT-260 (done reaction on close, same bot) |
| **Code** | Two commits on the support bot's own repository (not the ddt-salesforce repo) |

---

## 1. Summary

The Slack bot that turns support requests into Linear tickets now recognises **IT and office-hardware requests** and files them already assigned to the firm's IT support person, with an **IT** label and an `[IT]` title prefix. Everything else is still filed unassigned for the Salesforce team. The ticket was written as a record of work already deployed.

## 2. What was reported

- **2026-07-01:** a change record, not a fault report. Before it, every request the bot filed arrived unassigned, so IT requests waited in the same queue as Salesforce work until someone re-routed them by hand.

## 3. Scope

**In scope**

| Component | Change |
|---|---|
| `linear.find_user_by_email(email)` | New. Resolves an email to a Linear user id |
| `linear.create_issue` | Accepts `assignee_email` and passes the resolved `assigneeId` |
| `create_linear_ticket` tool schema | New `assignee_email` property |
| The classification prompt | Teaches the model which requests are IT |
| Label cache | Refreshes immediately on a miss, so a newly added label works without waiting out the 30-minute cache |

**Out of scope**

- **Salesforce requests.** These stay unassigned for the Salesforce team, including Salesforce login problems.
- **RingCentral, Heymarket and PR reviews.** Not IT; unchanged.

## 4. How it works (background)

```
Someone posts a request in the Slack support channel
  -> the bot asks Claude to classify it and draft a ticket
  -> Claude calls the create_linear_ticket tool (title, description, labels, now also assignee_email)
  -> the bot resolves labels and the assignee, and creates the Linear issue
  -> the Slack thread is linked to the Linear issue
```

## 5. Root cause

Not applicable: a new feature. The gap was that the bot had no way to assign a ticket, so all routing was manual.

## 6. Solution

### 6.1 What counts as IT

| IT (assigned, IT label) | Not IT (unassigned) |
|---|---|
| Office hardware: printers, monitors, keyboards, laptops | Salesforce admin and development |
| Office network, VPN, Wi-Fi | RingCentral and Heymarket operations |
| Operating system, laptop setup, software installs | Pull request reviews |
| Logins to SaaS tools such as Google, Slack, Microsoft 365 | Salesforce login problems |
| Small IT tasks: allow-listing an IP, adding a printer, restarting the router | |

**Salesforce login problems stay with the Salesforce team**, even though they are logins. If one turns out to be a single sign-on problem, the Salesforce team re-routes it.

### 6.2 How assignment works

The model returns an `assignee_email` with the ticket. The bot looks that email up in Linear, gets the user id, and creates the issue with that assignee.

### 6.3 The label cache fix

The bot caches Linear labels for 30 minutes. A brand-new label (such as **IT**) would have been unknown until the cache expired. On a cache miss it now refreshes immediately.

## 7. Testing

The ticket's acceptance checks:

- An IT request posted in Slack files with an `[IT]` title prefix, the IT label and the IT support person as assignee.
- The IT support person gets a Linear notification.
- "I can't log into Google" gets the IT label, not Salesforce.
- "I can't log into Salesforce" gets the Salesforce label.

The ticket does not record the results.

**Not covered:** the classification is done by a language model, so borderline requests can still be mislabelled. Nothing measures how often.

## 8. Step by step

To add a routing rule like this to the bot:

1. **Add a lookup** that turns an email into a Linear user id.
2. **Let ticket creation accept an assignee** by email, and pass the resolved id to Linear.
3. **Add the field to the tool schema** the model uses to file tickets.
4. **Describe the category in the prompt**, with clear "is" and "is not" lists. Put the tricky cases (Salesforce logins) on the "is not" side explicitly.
5. **Create the label in Linear** and make sure the bot's label cache picks it up.
6. **Test with one clear case from each side** of the boundary, in Slack.

## 9. Rollback

Remove the IT rules from the prompt and redeploy the bot. Tickets go back to being filed unassigned. The new code paths are harmless when no assignee is given.

## 10. Known limits

- **The assignee is one person.** If the IT support person changes, the email in the bot's routing must be updated.
- **Model judgement.** A request that mixes IT and Salesforce may land on either side.

## 11. Reuse for similar issues

**How to recognise one:** a category of requests always ends up with the same person, and someone re-routes them by hand.

- **Resolve the assignee at filing time** from an email, so the model never has to know Linear's internal ids.
- **Write the boundary down in the prompt**, with examples on both sides.
- **Bust caches on a miss** when the cached thing (labels, users) is something people add at any time.
