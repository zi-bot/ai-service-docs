# FWA jobs

Submitting and watching a Fraud/Waste/Abuse (FWA) analysis job — the app's most-watched
async flow, and the shape every other "start a job" flow in this app follows.

```mermaid
flowchart TD
    List["Jobs list<br/>(GET /ai/jobs, or POST /ai/jobs/filter<br/>when filters are applied)"]
    List -->|"Submit FWA job"| Dialog["'Analyze Fraud, Waste and Abuse (FWA)' dialog<br/>payload (JSON), reference ID (optional),<br/>include_exclusions checkbox"]
    Dialog -->|"Use example claim"| Filled["Payload textarea filled with a<br/>realistic example claim — Reference ID<br/>left untouched, it's optional"]
    Filled --> Dialog

    Dialog -->|Submit FWA job| Validate{Payload is<br/>valid JSON?}
    Validate -->|no| JsonErr["'This is not valid JSON.<br/>Fix it before submitting.'"]
    JsonErr --> Dialog
    Validate -->|yes| Submit["POST /ai/jobs<br/>{payload, reference_id?, include_exclusions, tenant_uuid}"]

    Submit -->|success| Opened["List invalidated + refetched;<br/>JobDetailPane opens for the new job"]
    Submit -->|ApiError| ApiErr["Inline error: the API's message"]
    Submit -->|other failure| GenericErr["'The AI service did not accept<br/>the job. Check that it is running.'"]
    ApiErr --> Dialog
    GenericErr --> Dialog

    Opened --> Watch["Checking every 2 seconds.<br/>This pane updates on its own."]
    Watch --> Socket["WebSocket:<br/>GET /ws/ai/jobs/:id?token=&lt;access_token&gt;"]
    Socket -->|message received| Live["Cache patched directly;<br/>2s polling backs off"]
    Socket -->|error / closes early| Fallback["2s polling resumes<br/>(GET /ai/jobs/:id)"]
    Live --> Watch
    Fallback --> Watch

    Watch --> Terminal{Status reaches a<br/>terminal state}
    Terminal -->|succeeded, has result_id| ResultLink["'Open the result' link<br/>→ /results/:result_id"]
    Terminal -->|failed| ShowError["job.error shown inline"]
    Terminal -->|needs_review| NeedsReview["Status stamp only —<br/>no further automatic action"]
```

## Details

- **The dialog's title reads "Analyze Fraud, Waste and Abuse (FWA)"**; the button that
  opens it and the submit button both still read "Submit FWA job" — the title was renamed
  for clarity without touching the action language (the brief's "an action keeps its name
  end to end" rule applies to the button, not necessarily a longer descriptive heading).
- **Reference ID is optional and doesn't need to match anything.** If it happens to
  resolve to a real claim already on file, the backend reads that claim's own data and
  ignores the typed payload; if it doesn't resolve, the typed payload is used as-is. There
  is no claim list in this UI to pick one from, which is why the field stays free-text
  and unrequired rather than a picker.
- **"Use example claim"** only fills the payload — a realistic claim shape matching what
  the backend itself builds from a real record (diagnosis/procedure codes, amounts,
  dates) — specifically so someone with no claim data on hand can see a working shape.
  It never touches Reference ID, since filling that with a made-up value would misrepresent
  the example as tied to a real claim.
- **WebSocket-first, polling always as a safety net.** `usePollingJob` opens the socket
  and the polling query at the same time. The socket is an accelerator: while it's
  delivering messages, `refetchInterval` returns `false` so the job isn't fetched twice.
  If the socket never connects, errors, or closes before a terminal status, polling was
  never actually turned off and picks the slack back up automatically — no user-visible
  difference either way beyond the fresher update cadence.
- **Getting to the result.** A `result_id` on a succeeded job is the only link into the
  [Results](./10-results.md) flow — there's no other way to browse to a specific result
  from a job.
