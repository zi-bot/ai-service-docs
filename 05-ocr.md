# OCR document intake

Three panels on one screen, sharing a single `activeJob` slot so only one job's status
pane shows at a time.

```mermaid
flowchart TD
    Page["Documents screen — three panels:<br/>Claim intake · Lab results · Read text now"]

    Page --> Intake["Claim intake:<br/>file + member identity number/type"]
    Intake -->|oversize for the job type's cap| SizeErr["Inline size error —<br/>no request sent"]
    Intake -->|submit| IntakeCall["POST /ai/jobs/ocr/intake<br/>FormData(file, member_personal_identity, ...)"]
    IntakeCall -->|success| ActiveIntake["activeJob = {id, kind: 'intake'}<br/>form resets"]
    IntakeCall -->|error| IntakeErr["Field errors + fallback:<br/>'AI service did not accept the document.'"]

    Page --> Lab["Lab results: file + optional claim_id"]
    Lab -->|submit| LabCall["POST /ai/jobs/lab/ocr<br/>FormData(file, claim_id?)"]
    LabCall -->|success| ActiveLab["activeJob = {id, kind: 'lab'}"]

    ActiveIntake --> OcrPane["Shared OcrJobPane"]
    ActiveLab --> OcrPane
    OcrPane --> OcrPoll["usePollingOcrJob:<br/>GET /ai/jobs/ocr/:id (or /ai/jobs/lab/ocr/:id)<br/>every 2s — no WebSocket, this endpoint<br/>carries the extraction payload the<br/>generic job-status socket doesn't"]
    OcrPoll --> OcrStatus{status}
    OcrStatus -->|queued/processing| Checking["'Checking every 2 seconds.'"]
    Checking --> OcrPoll
    OcrStatus -->|succeeded| Artifact["GET .../{id}/artifact<br/>rendered via RecordViewer<br/>('Extracted claim' / 'Extracted lab results')"]
    OcrStatus -->|failed| OcrFailed["poll.data.error shown inline"]
    OcrStatus -->|needs_review| NeedsPerson["'This document needs<br/>a person to check it.'"]

    Page --> Extract["Read text now: file<br/>+ optional 'Use OCR' checkbox"]
    Extract -->|submit| ExtractCall["POST /ai/text-extraction<br/>or /ai/text-extraction/ocr"]
    ExtractCall -->|success, synchronous| ExtractDone["Renders immediately:<br/>source label, extracted text, Copy button —<br/>no job, no polling"]
```

## Details

- **File-size caps are shown before upload**, sourced from `GET /ai/jobs/types` per job
  type, so an oversized file is rejected client-side rather than after a slow upload.
- **The artifact fetch is gated on `succeeded`** — it isn't attempted for `failed` or
  `needs_review`, since there's nothing structured to show yet (or ever, for a failure).
- **"Read text now" is the odd one out**: no job is created at all, the extraction is
  synchronous and the result renders on the same request/response.
