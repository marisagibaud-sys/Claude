# archive-inbox/

Ingest queue for the **Position Archive** artifact — the searchable record of
historical **New Position** and **Job Description** emails.

Drop one `.json` file per email here (the Power Automate flow does this
automatically; see the recipe below). A scheduled Claude task sweeps this
folder on weekday afternoons, files each email into the archive artifact —
extracting the structured components (Title, PL/CC/JC, Grade, Salary Range,
FLSA, PPT Level, License, FTE Leader, Direct Supervision, FTE Hours,
Reports to, JD Status) from the body — and the processed files are then moved
to `archive-inbox/processed/` by a daily housekeeping task.

## File format

Any `.json` file directly in this folder, shaped like this (only `subject`
is required):

```json
{
  "subject": "New Position — IRB Coordinator",
  "from": "Dana Reyes <dana@fabrikam.com>",
  "received": "2021-03-09T14:02:00Z",
  "category": "New Position",
  "id": "outlook-message-id",
  "imid": "internet-message-id",
  "body": "Title: IRB Coordinator\nGrade: GR 17\n… full announcement / JD text …",
  "bodyType": "html"
}
```

- `category` decides the record kind: `New Position` → New Position;
  `Job Description Review/Update` (or anything containing "description") →
  Job Description. Anything else is guessed from the subject.
- `body` is the full email body. When `bodyType` is `"html"` the sweep strips
  the markup to text first — the components table survives this as
  `Label: value` lines, which is what the extractor reads.
- `id` / `imid` are de-duplication keys — the same email never creates two
  records, so overlapping backfill runs are safe.

## Power Automate recipe (ongoing feed)

Mirrors the Triage Volume Board flow in the main README, with two changes:
the Graph query selects the body, and files land here instead of `inbox/`.
Build it as the board does — **one flow per category** (or one flow with two
GET + Apply-to-each pairs), which keeps each query identical to the shape
already proven in this tenant:

1. **Find categorized mail** — *Office 365 Outlook → Send an HTTP request*.
   Method `GET` in the **Method dropdown** (never typed into the URI field);
   the **URI field gets exactly one line**, no backticks, straight quotes:

   Flow A (New Position):

   ```
   https://graph.microsoft.com/v1.0/me/messages?$filter=categories/any(c:c eq 'New Position') and not(categories/any(c:c eq 'Archived'))&$select=id,internetMessageId,subject,from,receivedDateTime,categories,body
   ```

   Flow B (Job Description Review/Update): the same line with
   `'Job Description Review/Update'` in place of `'New Position'`.

2. **Apply to each** item in `body(...)?['value']` — **Compose** the payload
   (the `category` value is simply hardcoded per flow — `"New Position"` in
   Flow A, `"Job Description Review/Update"` in Flow B):

   ```json
   {
     "subject": "@{item()?['subject']}",
     "from": "@{item()?['from']?['emailAddress']?['name']} <@{item()?['from']?['emailAddress']?['address']}>",
     "received": "@{item()?['receivedDateTime']}",
     "category": "New Position",
     "id": "@{item()?['id']}",
     "imid": "@{item()?['internetMessageId']}",
     "body": "@{item()?['body']?['content']}",
     "bodyType": "@{item()?['body']?['contentType']}"
   }
   ```

3. **HTTP** (premium connector; a consumption-plan Azure Logic App works the
   same without the premium license):

   ```
   PUT https://api.github.com/repos/marisagibaud-sys/Claude/contents/archive-inbox/@{guid()}.json
   Headers:
     Authorization: Bearer <your fine-grained token>
     Accept: application/vnd.github+json
   Body:
     {
       "message": "archive email",
       "branch": "claude/email-archive-positions-v00ubh",
       "content": "@{base64(string(outputs('Compose')))}"
     }
   ```

4. **Mark it handled** — *Send an HTTP request*:

   ```
   PATCH https://graph.microsoft.com/v1.0/me/messages/@{item()?['id']}
   Body: { "categories": @{union(item()?['categories'], createArray('Archived'))} }
   ```

The fine-grained token is the same one the board flow uses (scoped to this
repo, *Contents: Read and write*).

### If the *Send an HTTP request* step fails

**"URI path is not a valid Graph endpoint … Invalid resource, Allowed
values: me,users"** — this comes from the Outlook connector's own URI
parser, *before* anything is sent to Graph, and it means the URI field
didn't parse to `…/me/messages`. The `$filter` contents can never cause
this error (a bad filter comes back as a Graph 400 instead). Fix the field,
not the filter:

1. Delete everything in the URI field and re-paste the one-line URI from
   step 1 above. It must **start with** `https://graph.microsoft.com/` —
   no `GET ` prefix (the method lives in the dropdown), no backticks, no
   surrounding quotes, no line break anywhere in the middle.
2. Check the quotes around the category name are straight (`'New Position'`)
   — pasting through Word/OneNote/Teams can curl them.
3. Still failing? Retype the URI by hand up to `…/me/messages`, then paste
   only the query string after `?`.

**A Graph 400/error mentioning the filter** — the category name in the
`$filter` must match the Outlook category exactly, including capitalization
and the `/` in `Job Description Review/Update`.

## Backfilling the legacy history

Two ways, safe to combine:

- **Bulk drag & drop** — in Outlook, select any number of historical emails,
  drag them into a desktop folder (they save as `.msg`), then drop the whole
  batch anywhere on the archive page. Subject, sender, original date, full
  body, and the components are all read from the files.
- **One-time flow run** — run each flow with a date window added to its
  `$filter`, e.g. for Flow A:

  ```
  $filter=receivedDateTime ge 2019-01-01T00:00:00Z and receivedDateTime lt 2020-01-01T00:00:00Z and categories/any(c:c eq 'New Position')
  ```

  Run it a year at a time (Graph pages at 10 by default — follow
  `@odata.nextLink`, or add `$top=100`). Duplicates are skipped, so re-runs
  are harmless.
