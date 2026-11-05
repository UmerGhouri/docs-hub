# RingCentral October 2026 developer update impact

| | |
|---|---|
| **Ticket** | DDT-1406 |
| **Project** | Support tickets |
| **Status** | Done 2026-10-23 |
| **Priority** | No priority |
| **Reported by** | Firm operations, 2026-10-23 |
| **Resolved by** | Support team |
| **Systems** | RingCentral (Salesforce package, REST API), Salesforce |
| **Area** | Phones: the RingCentral integration with Salesforce |
| **Related** | None recorded |
| **Code** | Not applicable: review only, no change made |

---

## 1. Summary

Firm operations forwarded RingCentral's October 2026 developer newsletter and asked whether anything in it affected the firm or could help it. Each item was checked against the firm's RingCentral setup. None affected it and none needed action. Firm operations reviewed the findings and agreed.

## 2. What was reported

On 2026-10-23 firm operations asked for someone to investigate the newsletter's changes. The ticket asked for three things:

- check App Connect 2.0, the main product change;
- check whether it affects the RingCentral setup in Salesforce or any RingCentral API the firm calls;
- note anything else worth acting on.

## 3. Scope

**In scope:** every item in the newsletter, compared with how the firm uses RingCentral.

**Out of scope:** any change. None was needed.

## 4. How it works (background)

How the firm uses RingCentral, as relevant to the review:

- **Calling inside Salesforce** runs through RingCentral's own Salesforce package.
- **The firm's own sync code** in Salesforce calls RingCentral's REST API: call log, live call events (webhooks), recordings, extensions, RingSense call insights, and sign-in.
- **The AI assistant's RingCentral connector** is the firm's own self-hosted server calling the RingCentral REST API directly. It does not use RingCentral's hosted MCP server.

## 5. Root cause

Not applicable: this was a review of a vendor update, not a fault.

## 6. Solution

The findings, item by item:

| Newsletter item | What it is | Effect on the firm |
|---|---|---|
| **App Connect 2.0** | RingCentral's browser extension for logging calls into a CRM. It supports 19 CRMs, and Salesforce is not one of them. | None. It touches neither the Salesforce package nor the firm's sync code, and changes none of the APIs the firm calls. |
| **Admin MCP (preview)**, and RingCentral splitting its hosted MCP server into Phone, Chat and Admin servers, retiring the old single server | Lets an AI assistant read RingCentral admin data (users, call stats, availability). | None. The firm's connector is self-hosted and calls the REST API directly. Sign-in, call log and RingSense insights were checked live on 2026-10-23 and all worked. |
| **Embeddable 3.0 (preview)** | A phone widget for websites. | Not needed: calling runs inside Salesforce. |
| **Events** | A meetup on App Connect plugins; a recorded demo of bringing RingCentral data into Claude; a webinar on agentic voice AI. | The meetup is not relevant. The demo covers something the firm already does. The webinar is optional background for the AI receptionist. |
| **Blog posts** | IVR, ServiceNow, video and conference calls. | The IVR post may be worth reading for the AI receptionist. The others cover products the firm does not use. |

**Recommendation:** no action needed. Firm operations reviewed it on 2026-10-23, agreed, and the ticket was closed.

## 7. Testing

The one live check: sign-in, call log and RingSense insights through the AI assistant's RingCentral connector, run on 2026-10-23, all working after RingCentral retired its old single hosted MCP server.

**Not covered:** nothing was changed, so nothing else was tested.

## 8. Step by step

To review a vendor update like this:

1. **List every item** in the update.
2. **For each, write down what it is** in one line, from the vendor's own description.
3. **Map it against how the firm uses that vendor**: which package, which APIs, which connectors.
4. **Check live anything that could break silently**, such as a connector whose server the vendor has retired or renamed.
5. **Post the findings on the ticket** with a recommendation, and get the requester to review before closing.

## 9. Rollback

Not applicable: nothing was changed.

## 10. Known limits

- **A point-in-time review.** Two items were previews. If either becomes generally available, or RingCentral changes an API the sync code calls, review again.
- **App Connect** would need a fresh look if RingCentral adds Salesforce to its supported CRMs.

## 11. Reuse for similar issues

**How to recognise one:** a vendor newsletter, release note or deprecation notice forwarded with "does this affect us?".

- **Start from what the firm actually uses**, not from the vendor's headline feature. The headline item here applied to none of it.
- **Name the APIs your code calls** and check each one against the notice.
- **Look for retirements**, not just new features. A retired endpoint or server is what breaks things.
- **Verify live** the one or two things that could fail silently, and say what you checked.
