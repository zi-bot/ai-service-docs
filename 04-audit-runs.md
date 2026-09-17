# Audit runs

Running a rule set (and/or the model) against a batch of claims, watching it complete,
then reading what it found.

```mermaid
flowchart TD
    List["Audit runs list<br/>(GET /ai/jobs/audit)"]
    List -->|"Start audit run"| Dialog["Start audit run dialog<br/>GET /ai/audit-rules/example-configs<br/>for the checks checklist"]

    Dialog --> Mode{Mode}
    Mode -->|Filter| FilterMode["source (audit_claim/claim) +<br/>FilterBuilder over claim fields"]
    Mode -->|File| FileMode["title, description,<br/>file_upload_id (pasted in)"]

    FilterMode --> Checks["Pick ≥1 check<br/>+ include_exclusions"]
    FileMode --> Checks
    Checks -->|no checks picked| ChecksErr["'Pick at least one<br/>check to run.'"]
    Checks -->|Start| SubmitFilter["POST /ai/jobs/audit<br/>{source, checks, filters, include_exclusions}"]
    Checks -->|Start, file mode| SubmitFile["POST /ai/jobs/audit/upload<br/>{file_upload_id, title, description, checks, ...}"]

    SubmitFilter -->|success| Queued["List invalidated, dialog closes.<br/>No auto-open — the run row<br/>appears once the list refetches."]
    SubmitFile -->|success| Queued
    SubmitFilter -->|error| SubmitErr["ApiError message, or<br/>'The AI service did not accept<br/>the audit run. Check that it is running.'"]

    Queued -->|user selects the row| Detail["AuditRunDetailPane<br/>GET /ai/jobs/audit/:runId"]
    Detail --> Running{run.status is<br/>queued/processing?}
    Running -->|yes| Poll["usePollingJob(run.job_id) —<br/>same WebSocket + polling-fallback<br/>mechanism as FWA jobs"]
    Poll --> Watching["'Checking every 2 seconds.'"]
    Poll -->|reaches terminal status| Refetch["Run-detail query invalidated →<br/>run.status refetched from the<br/>audit-run endpoint itself"]
    Refetch --> Running

    Running -->|no| Findings["FindingsTable mounts<br/>GET /ai/jobs/audit/:runId/findings"]
    Findings -->|no findings| Clean["'This run found no anomalies.'"]
    Findings -->|rows returned| Rows["claim_no, member_identity, anomaly_type,<br/>severity Stamp, found-by (Rule/Model)"]
    Rows -->|click a row| Expand["Row expands inline —<br/>full finding.detail text.<br/>No separate page."]
```

## Details

- **Two ways to define the input batch**: a filter over claim fields (built with the same
  [`FilterBuilder`](../design/2026-09-12-ai-console-design-brief.md) used on the Jobs
  list), or a pre-uploaded file referenced by `file_upload_id` — that upload itself
  happens outside this flow.
- **No auto-open of the new run.** Starting a run returns a *job*, not a run row with an
  id the UI already knows how to open; the new row surfaces once the list invalidates and
  refetches, and the user picks it manually.
- **The run's own status and the underlying job's status are two different reads.**
  `usePollingJob` watches the job; once that job goes terminal, the run-detail query is
  invalidated so the *run* record (which the findings table depends on) gets refetched
  from its own endpoint rather than assumed from the job's status.
- **Findings have no drill-down page** — clicking a row expands it in place. There's
  nothing to navigate to beyond the detail text already returned with the row.
