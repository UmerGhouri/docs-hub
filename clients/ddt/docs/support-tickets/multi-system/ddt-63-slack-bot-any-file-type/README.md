# Support Slack bot accepts attached files of any type

| | |
|---|---|
| **Ticket** | DDT-63 |
| **Project** | Support tickets |
| **Status** | Live since 2026-05-26 |
| **Priority** | Medium |
| **Reported by** | Firm staff, 2026-05-26 |
| **Resolved by** | Support team |
| **Systems** | Slack bot, Linear, downstream file storage |
| **Area** | Support intake: requests sent to the bot in Slack |
| **Related** | None recorded |
| **Code** | Not in the `ddt-salesforce` repo. Where the bot's code lives is not recorded in the ticket. |

---

## 1. Summary

The firm's Slack bot only handled plain text. Staff wanted to send it anything they'd naturally attach: a screenshot of an error, a PDF, a CSV. This was a feature request.

The bot was extended to accept any file type Slack supports, keep the file's details, pass it on to the right downstream system, and tell the user clearly when a file can't be accepted.

## 2. What was reported

The request listed the expected uses:

- A screenshot of a Salesforce error, attached to a Linear ticket.
- A PDF, such as a court document or signed agreement, routed to the right case file.
- A CSV, stored or forwarded for processing.

It also asked that the bot not fail silently on unsupported or oversized files, and that it only use files from authorised users and channels.

## 3. Scope

**In scope** (from the ticket's acceptance criteria and the resolution note)

| Capability | Result per the resolution note |
|---|---|
| Accept any file type Slack supports | Done |
| Preserve file name, sender, timestamp and channel | Done |
| Route the file to the right downstream system | Done |
| Clear message for unsupported or oversized files | Done |
| Only accept files from authorised users and channels | Done |

**Out of scope / not recorded**

- **The answers to the open questions**: where each kind of file ends up, the size limit, whether every file gets an acknowledgement, which file types are rejected, and whether it covers DMs, channel mentions or both. **Not recorded in the ticket.**

## 4. How it works (background)

```
User posts a message with a file to the bot in Slack
  -> Slack sends the bot a file-share event
  -> the bot downloads the file using its own Slack credentials
  -> the bot decides where it goes (e.g. a Linear ticket attachment)
  -> the bot replies to the user
```

Slack delivers uploads to bots as file-share events, and the file itself has to be downloaded with the bot's own token; it is not inside the event. A bot that only subscribes to message text never sees the file.

## 5. Root cause

Not applicable: this was a new capability, not a fault. The bot had only been built for text.

## 6. Solution

Per the resolution note, the bot now:

- accepts any file type Slack supports;
- keeps the file name, sender, timestamp and channel with the file;
- routes the attachment to the right downstream system;
- handles unsupported or oversized files with a clear message to the user instead of failing silently;
- checks that the sender and channel are authorised before using the file.

The implementation details are **not recorded in the ticket.**

## 7. Testing

**Not recorded in the ticket.** The ticket was closed as done on 2026-05-26 and the resolution note was added on 2026-06-02.

## 8. Step by step

The build steps are **not recorded in the ticket.** To check the behaviour, work through the acceptance criteria:

1. **Send the bot an image, a PDF and a CSV.** Each should be accepted.
2. **Check the downstream copy** of each file and confirm the file name, sender, timestamp and channel came with it.
3. **Send a file that is too large or of a refused type.** The bot should reply with a clear message, not stay silent.
4. **Send a file from a user or channel that is not authorised.** The bot should not use it.

## 9. Rollback

Not recorded in the ticket.

## 10. Known limits

- **The routing rules, size limit and blocked types are not recorded**, so what the bot does with a given file has to be read from the bot itself.
- **The code is outside this repo**, so this page can't point at it.

## 11. Reuse for similar issues

**How to recognise one:** a bot or integration "ignores" attachments while handling text normally.

- **Check the event subscription first.** Text and files arrive as different events.
- **Files need a separate, authenticated download.** The event only describes the file.
- **Refuse loudly.** A file that is too large or the wrong type should get a reply saying so, never silence.
- **Write the routing rules down** where staff can see them, so people know where their file will end up.
