# Cost predictions

One member at a time, a whole book at once, or the config that shapes every prediction's
risk-tier cutoffs.

```mermaid
flowchart TD
    List["Cost predictions list<br/>GET /ai/cost-predictions<br/>filters: Member (SearchPicker), Batch ID<br/>(free text, no directory), Risk tier, Status"]

    List -->|"Predict a member"| PredictForm["Member ID, Horizon, Period start/end,<br/>Include an explanation"]
    PredictForm -->|Submit| PredictCall["POST /ai/cost-predictions"]
    PredictCall -->|success| OpenNew["List invalidated;<br/>pane switches straight to the<br/>new prediction's detail"]
    PredictCall -->|error| PredictErr["Field errors + fallback message"]

    List -->|"click a row"| Detail["PredictionDetailPane<br/>GET /ai/cost-predictions/:uuid"]
    OpenNew --> Detail
    Detail --> Status{status}
    Status -->|completed| Range["'Likely between {p10} and {p90},<br/>most likely {p50}' + Low/Most likely/High table"]
    Status -->|insufficient_data| NotEnough["'Not enough claim history<br/>to predict this member.'"]
    Range --> Divergence{llm_divergence_flag?}
    Divergence -->|yes| Wide["'The model and the statistical<br/>estimate disagree on this member.<br/>Treat the range as wide.'"]

    List -->|"Run a batch"| BatchForm["Horizon, Period start/end, Batch size"]
    BatchForm -->|Submit| BatchCall["POST /ai/cost-predictions/batch"]
    BatchCall -->|success| BulkTracker["Pane closes; BulkJobStatus<br/>mounted above the list"]
    BulkTracker --> BulkPoll["GET /ai/.../bulk-jobs/:id every 2s,<br/>60s cap — same tracker as Audit rules"]
    BulkPoll -->|succeeded| BatchDone["'The batch's predictions are ready.'<br/>+ 'Refresh the list' (manual invalidate)"]
    BulkPoll -->|failed| BatchFailed["job error shown"]

    List -->|"Prediction settings"| Config["CostPredictionConfigForm<br/>GET /ai/cost-prediction-config"]
    Config -->|404| Fresh["Treated as 'not configured yet' —<br/>fields seed blank, not an error"]
    Config -->|fields: lookback months, full-credibility<br/>months, annual trend, tier cutoffs| SaveConfig["PUT /ai/cost-prediction-config"]
    SaveConfig -->|success| ConfigSaved["Pane closes, notice: 'Settings saved'"]
```

## Details

- **Member is a `SearchPicker`, Batch ID is free text.** There's a member directory to
  search against (`searchMembers`); there's no equivalent directory for batch ids, so that
  field keeps the draft/commit-on-Enter-or-blur pattern instead.
- **A 404 on the config endpoint means "nothing saved yet," not a failure** — the form
  opens with blank fields and the first save is effectively a create, same 404-as-empty
  convention as the [budget](./08-budget.md) flow.
- **`insufficient_data` suppresses the figures block entirely** rather than showing
  zeroed-out numbers — a "no answer" is shown as no answer, not a misleading zero.
- **The batch tracker is the same `BulkJobStatus` component** used by Audit rules' bulk
  upload — one shared tracker, reused wherever a job produces a count rather than a single
  row.
