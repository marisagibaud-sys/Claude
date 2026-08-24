# Triage Volume Board

A shared drag-and-drop tracker for categorizing email and Microsoft Teams volume.
The board is published as a Claude artifact — one page the team opens, drops
messages onto, and reads volume trends from. Every change is saved back into the
artifact itself, so everyone with access sees the same board.

`index.html` is the entire app (no build step). Publishing it as a Claude artifact
with the `artifact` and `downloads` capabilities turns on shared saving and CSV
export. **No connectors are used** — Claude has no access to any mailbox or file
store; automated items are *pushed* into this repo by a flow the mailbox owner
controls (see below).

## What it does

- **Drop emails on a column** — drag `.eml` / `.msg` files (saved or dragged out of
  Outlook) onto a category column and the subject, sender, and received date are
  parsed out of the file. Dropping selected text works too.
- **Paste Teams messages** — copy a message in Teams, press `Ctrl+V` on the board,
  confirm the parsed fields, pick a category.
- **Categorize by dragging** — cards move between columns; each column maps to an
  Outlook category (editable via the column's `⋯` menu).
- **Attribution** — cards carry the initials of whoever added them (asked for once,
  remembered per device); automated items are marked `auto`.
- **Volume view** — stat tiles plus an 8-week stacked chart of items per week per
  category, with per-column weekly counts. **Export CSV** downloads the raw rows.
- **Outlook category automation** — apply a category in Outlook and the message
  lands on the board without any dragging (setup below).

Default columns mirror the team's Outlook categories: Promotion, Other, Market
Adjustment, Reorg, Job Evaluation, New Position, Research, Data Request, and
Benchmarking (neutral, matching its uncolored Outlook chip), plus an
Uncategorized bucket for anything that doesn't match.

## The Outlook category automation

Design constraint: the mailbox owner does **not** grant Claude any access to
Outlook or Microsoft 365. So the data flows one way, pushed from the Microsoft
side into this repo:

```
Outlook category applied
        │  (Power Automate, every 15 min — runs entirely in your tenant)
        ▼
this repo: inbox/<guid>.json          one small file per categorized email
        │  (scheduled Claude task, weekdays every 2 hours)
        ▼
Board column whose "Outlook category" field matches; file moves to inbox/processed/
```

Claude only ever reads what the flow chose to send: subject, sender, received
date, category, message id. The scheduled task de-duplicates by message id, files
non-matching categories into Uncategorized, and updates the published board.

### One-time setup

1. **GitHub token** — create a fine-grained personal access token scoped to this
   repository only, permission *Contents: Read and write*
   (github.com → Settings → Developer settings → Fine-grained tokens). This is
   what lets the flow write files here; it grants nothing else.

2. **Power Automate** — a scheduled cloud flow, every 15 minutes:

   - **Find categorized mail** — *Office 365 Outlook → Send an HTTP request*:

     ```
     GET https://graph.microsoft.com/v1.0/me/messages?$filter=categories/any(c:c eq 'Promotion') and not(categories/any(c:c eq 'Logged'))&$select=id,subject,from,receivedDateTime,categories
     ```

     Swap `'Promotion'` for whichever category, or run one flow per category —
     the board routes each file by its `category` value either way.

   - **Apply to each** item in `body(...)?['value']`:

     1. **Compose** the payload:

        ```json
        {
          "subject": "@{item()?['subject']}",
          "from": "@{item()?['from']?['emailAddress']?['name']} <@{item()?['from']?['emailAddress']?['address']}>",
          "received": "@{item()?['receivedDateTime']}",
          "category": "@{first(item()?['categories'])}",
          "id": "@{item()?['id']}"
        }
        ```

     2. **HTTP** (premium connector):

        ```
        PUT https://api.github.com/repos/marisagibaud-sys/Claude/contents/inbox/@{guid()}.json
        Headers:
          Authorization: Bearer <your fine-grained token>
          Accept: application/vnd.github+json
        Body:
          {
            "message": "log categorized email",
            "branch": "claude/email-teams-drag-drop-tracker-2qgei7",
            "content": "@{base64(string(outputs('Compose')))}"
          }
        ```

     3. **Mark it handled** — a second *Send an HTTP request*:

        ```
        PATCH https://graph.microsoft.com/v1.0/me/messages/@{item()?['id']}
        Body: { "categories": @{union(item()?['categories'], createArray('Logged'))} }
        ```

   > **No Power Automate premium?** The plain HTTP action needs a premium
   > license. If that's not available, an Azure **Logic App** (consumption plan)
   > has the same Office 365 Outlook connector plus a free built-in HTTP action,
   > and costs pennies at this volume — same steps, same recipe.

3. **Claude's scheduled sweep** — already configured: a recurring task runs on
   weekdays every 2 hours, checks `inbox/`, merges new items into the published
   board, and moves processed files to `inbox/processed/`. When the inbox is
   empty it does nothing.

### Inbox file format

Any `.json` file in `inbox/` with this shape (only `subject` is required):

```json
{
  "subject": "Comp review — Fabrikam req",
  "from": "Dana Reyes <dana@fabrikam.com>",
  "received": "2026-08-24T14:02:00Z",
  "category": "Promotion",
  "id": "unique-message-id"
}
```

`category` matches a column's *Outlook category* field (or its name),
case-insensitively. `id` is the de-duplication key — the same id never creates
two cards.

## How shared state works

The board's state (categories, items, meta) lives in the
`<script id="vb-state">` block of `index.html`. The page never serializes its
live DOM: it rebuilds the complete document from its own parse-time source plus
current state (`assemble()` in the code) and publishes that as the new artifact
version — compare-and-set, so concurrent editors can't clobber each other. The
scheduled sweep does the same merge server-side. Escaping note: `</` inside the
state JSON is written as `<\/` (a legal JSON escape) so no subject line can break
out of the script block.

## Parsers included

| Input | What's extracted |
|---|---|
| `.eml` | Subject (RFC 2047 decoded), From, Date, Message-ID |
| `.msg` (Outlook) | Subject, sender, received time via a minimal MS-CFB reader |
| Pasted/dropped text | `From:` / `Subject:` / `Sent:` header lines, or first line as the summary; Teams paste patterns detected and split into author + message |
| Inbox JSON | `subject`, `from`, `received`, `category`, `id` |

Parsers and `assemble()` are reachable in the browser console as `window.__vb`.
