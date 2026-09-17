# Results

Where a succeeded job's output actually lives, and the one place raw model/tool output
gets rendered for a human to read.

```mermaid
flowchart TD
    Job["JobDetailPane — job succeeded,<br/>has a result_id"]
    Job -->|"Open the result"| Nav["/results/:id"]

    List["Results list<br/>GET /ai/results (paginated)"]
    List -->|empty| NoResults["'No results yet. A result appears<br/>here when a job succeeds.'"]
    List -->|"click a row"| Selected["Detail pane opens for that row"]
    Nav --> Selected

    Selected --> Fetch["GET /ai/results/:id"]
    Fetch --> Meta["Result id, job id, provider, model,<br/>'Kept until' expiry (or —), created"]
    Meta --> Extracted{result.extracted_output<br/>present?}
    Extracted -->|yes| ExtractedBlock["'Extracted output' section"]
    Meta --> Response{result.response != null?}
    Response -->|yes| ResponseBlock["'Response' section"]

    ExtractedBlock --> Pretty["PrettyBody: object, or a string that<br/>JSON.parse()s into one, renders as<br/>pretty-printed JsonBlock; anything else<br/>renders as plain prose"]
    ResponseBlock --> Pretty
    Pretty --> Copy["'Copy' button — copies exactly<br/>what's shown (stringified JSON or prose)"]

    Selected -->|close, arrived via URL id| Back["Navigate back to /results (replace)"]
```

## Details

- **Arriving from a job is the common path** — a `result_id` on a succeeded job is the
  only in-app link into this screen; there's no other cross-reference from, say, an audit
  finding or a chat message.
- **Deep-linking works both ways**: `/results/:id` seeds the detail pane directly from the
  URL even before any row is clicked in the list, and closing a pane that was opened this
  way navigates back to the bare `/results` list rather than leaving a stale id in the URL.
- **`PrettyBody`'s object-detection is the whole point of the component**: a response
  that's already a JS object, or a JSON-shaped string, is read as a record and
  pretty-printed; anything else — including a string that merely looks like JSON but
  fails to parse — falls back to prose. This is the exact bug surface fixed earlier
  (`String(object)` producing `[object Object]`) and the flow now handles both shapes on
  purpose.
