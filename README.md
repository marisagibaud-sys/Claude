# Triage Volume Board

A shared drag-and-drop tracker for categorizing email and Microsoft Teams volume.
The board is published as a Claude artifact — one page the whole team opens, drops
messages onto, and reads volume trends from. Every change is saved back into the
artifact itself, so everyone sees the same board.

`index.html` is the entire app (no build step). Publishing it as a Claude artifact
with the `artifact`, `mcp` (Dropbox), and `downloads` capabilities turns on shared
saving, Outlook auto-import, and CSV export.

## What it does

- **Drop emails on a column** — drag `.eml` / `.msg` files (saved or dragged out of
  Outlook) onto a category column and the subject, sender, and received date are
  parsed out of the file. Dropping selected text works too.
- **Paste Teams messages** — copy a message in Teams, press `Ctrl+V` on the board,
  confirm the parsed fields, pick a category.
- **Categorize by dragging** — cards move between columns; each column maps to an
  Outlook category (editable via the column's `⋯` menu).
- **Volume view** — stat tiles plus an 8-week stacked chart of items per week per
  category, with per-column weekly counts. **Export CSV** downloads the raw rows
  for deeper analysis.
- **Outlook category automation** — apply a category in Outlook and the message
  lands on the board without any dragging (setup below).

Default columns mirror the team's Outlook categories: Promotion, Other, Market
Adjustment, Reorg, Job Evaluation, New Position, Research, Data Request, and
Benchmarking (neutral, matching its uncolored Outlook chip), plus an
Uncategorized bucket for anything that doesn't match.

## The Outlook category automation

Artifacts can't receive webhooks, so the bridge is a folder in Dropbox: Power
Automate drops one small JSON file per categorized message into a watched folder,
and the board (using the viewer's Dropbox connector) stages new files and imports
them into the matching column with one click.

```
Outlook category applied
        │  (Power Automate, every 15 min)
        ▼
Dropbox: /Claude/Email Tracker/Inbox/<message-id>.json
        │  (board watches the folder via the Dropbox connector)
        ▼
Board column whose "Outlook category" field matches
```

### One-time setup

1. **Dropbox** — create the folder `/Claude/Email Tracker/Inbox` (the path is
   editable in the board's *Automation* drawer).
2. **Power Automate** — create a scheduled cloud flow:
   - **Trigger:** Recurrence, every 15 minutes.
   - **Action:** *Office 365 Outlook → Send an HTTP request*:

     ```
     GET https://graph.microsoft.com/v1.0/me/messages?$filter=categories/any(c:c eq 'Promotion') and not(categories/any(c:c eq 'Logged'))&$select=id,subject,from,receivedDateTime,categories
     ```

     (Swap `'Promotion'` for whichever categories you track, or run one flow
     per category.)
   - **Apply to each** result (`body('Send_an_HTTP_request')?['value']`):
     1. *Dropbox → Create file* in the folder above, file name
        `@{item()?['id']}.json`, content:

        ```json
        {
          "subject": "@{item()?['subject']}",
          "from": "@{item()?['from']?['emailAddress']?['name']} <@{item()?['from']?['emailAddress']?['address']}>",
          "received": "@{item()?['receivedDateTime']}",
          "category": "@{first(item()?['categories'])}",
          "id": "@{item()?['id']}"
        }
        ```
     2. A second *Send an HTTP request* to mark the message handled so the next
        run skips it:

        ```
        PATCH https://graph.microsoft.com/v1.0/me/messages/@{item()?['id']}
        Body: { "categories": @{union(item()?['categories'], createArray('Logged'))} }
        ```
3. **On the board** — open *Automation*, confirm the folder is being watched, and
   give each column the Outlook category it maps to (the `⋯` menu). Files whose
   `category` doesn't match any column land in Uncategorized.

The board de-duplicates by Dropbox file id and by message id, so re-runs and
overlapping flows are safe. Any `.json`, `.eml`, or `.txt` file dropped in the
folder — by a flow or by hand — is importable; JSON gets the richest parsing.

### Notes

- Auto-import runs with the *viewer's* Dropbox connector (added in claude.ai
  Settings → Connectors) and needs read access to the watched folder. Teammates
  without the connector can still use every drag-and-drop feature.
- Because the page calls a connector, the artifact can be shared with specific
  people or your organization, but not published fully publicly.
- Import is one click rather than fully silent by design: saving the board is a
  shared write, so a person confirms it.

## Parsers included

| Input | What's extracted |
|---|---|
| `.eml` | Subject (RFC 2047 decoded), From, Date, Message-ID |
| `.msg` (Outlook) | Subject, sender, received time via a minimal MS-CFB reader |
| Pasted/dropped text | `From:` / `Subject:` / `Sent:` header lines, or first line as the summary; Teams paste patterns detected and split into author + message |
| Automation JSON | `subject`, `from` (string or Graph shape), `received`, `category`/`categories`, `id` |

Parsers are reachable in the browser console as `window.__vb` for testing.
