# Audit rules

Rules can be managed one at a time or in bulk via CSV/XLSX upload. Both paths write to the
same list.

```mermaid
flowchart TD
    List["Audit rules list<br/>(GET /ai/audit-rules)"]

    List -->|"Add rule"| Blank["AuditRuleForm — blank<br/>rule type, description, config JSON, enabled"]
    List -->|"click a row"| Fetch["GET /ai/audit-rules/:id"]
    Fetch --> Prefilled["AuditRuleForm — prefilled"]

    Blank -->|pick rule type| Example["GET /ai/audit-rules/example-configs<br/>populates type list + 'Use this example'"]
    Example -->|"Use this example"| Blank

    Blank -->|Save| ValidJson{Config is<br/>valid JSON?}
    Prefilled -->|Save| ValidJson
    ValidJson -->|no| JsonErr["'This is not valid JSON.<br/>Fix it before submitting.'"]
    ValidJson -->|yes, new| Create["POST /ai/audit-rules"]
    ValidJson -->|yes, existing| Update["PUT /ai/audit-rules/:id"]
    Create -->|success| Saved["List invalidated, pane closes,<br/>notice: 'Rule saved'"]
    Update -->|success| Saved
    Create -->|error| SaveErr["Field errors + fallback:<br/>'The AI service did not accept<br/>the rule. Check that it is running.'"]
    Update -->|error| SaveErr

    Prefilled -->|Delete| Confirm{"window.confirm<br/>'Delete the {type} rule?<br/>This cannot be undone.'"}
    Confirm -->|OK| DeleteCall["DELETE /ai/audit-rules/:id"]
    DeleteCall -->|success| Removed["List invalidated, pane closes"]
    Confirm -->|Cancel| Prefilled

    List -->|"Upload rules /<br/>Upload rule updates"| BulkDialog["BulkUploadDialog<br/>(create or update mode)"]
    BulkDialog -->|pick file, submit| BulkCall["POST /ai/audit-rules/bulk/upload<br/>or .../bulk/upload/update"]
    BulkCall -->|success| Tracker["BulkJobStatus mounted above the list<br/>'Creating/Updating rules from {file}…'"]
    BulkCall -->|no file picked| PickErr["'Pick a file to upload.'"]

    Tracker --> Poll["GET /ai/audit-rules/bulk-jobs/:id<br/>every 2s"]
    Poll --> BulkStatus{status}
    BulkStatus -->|succeeded| BulkDone["'Rules created.' / 'Rules updated.'<br/>+ 'Refresh the list' button"]
    BulkStatus -->|failed| BulkFailed["job.error shown in red"]
    BulkStatus -->|60s elapsed, still pending| Stuck["'Still queued. The bulk<br/>worker may not be running.'"]
    BulkDone -->|"Refresh the list"| Manual["List query invalidated manually —<br/>it does not auto-refresh"]
```

## Details

- **The example-config shortcut only appears once a rule type is picked**, and only if
  that type actually has an `example` — it fills the config textarea, it doesn't submit.
- **Delete is edit-mode only** — there's no delete affordance from the blank/create form,
  and no bulk-delete path.
- **The bulk tracker's 60-second cap** exists because the upload jobs run in a separate
  worker process that may not be running at all; past that cap without a terminal status,
  the UI says so explicitly rather than spinning forever.
- **A successful bulk job does not auto-refresh the list** — "Refresh the list" is a
  deliberate manual action, not an oversight; the invalidate-on-success pattern used
  elsewhere (single-rule save) doesn't apply here because the bulk job is a *count*, not
  a returned row the page already has in hand.
