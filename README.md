# AI Console — flow documentation

This is a screen-by-screen account of how a user moves through the AI Console (this
frontend) and what each action does against the backend: which endpoint it calls, what
happens on success, what happens on failure, and any loading/empty states along the way. this document is the current source of truth.

## How to read these diagrams

Every flow is a Mermaid `flowchart TD`. The convention across all of them:

- **Rectangles** — a screen, pane, or dialog the user sees.
- **Rounded rectangles** — a state within a screen (loading, empty, terminal status).
- **Diamonds** — a decision point (success vs. error, a status branch).
- Solid arrows are user actions or automatic transitions; arrow labels name the API call
  (`METHOD /path`) where one is made.
- A loop-back arrow into a "polling" node means "repeats every 2s until a terminal
  condition," not a single request.

## Index

| Flow | Covers |
|---|---|
| [Authentication](./01-authentication.md) | Login, session bootstrap, token refresh, logout, route guarding |
| [FWA jobs](./02-fwa-jobs.md) | Submit an FWA analysis job, watch it complete (WebSocket + polling fallback), open the result |
| [Audit rules](./03-audit-rules.md) | CRUD on rules, bulk CSV upload/update and its job tracker |
| [Audit runs](./04-audit-runs.md) | Start a run against a rule set, watch it complete, review findings |
| [OCR document intake](./05-ocr.md) | Claim intake, lab result OCR, plain text extraction |
| [Chat](./06-chat.md) | Sessions, sending a message, partial-answer and retry handling |
| [Cost predictions](./07-cost-predictions.md) | Predict a member, run a batch, prediction settings |
| [Budget / spend & usage](./08-budget.md) | Monthly cap, tenant/member token usage |
| [Admin setup](./09-admin-setup.md) | Providers, model catalog, per-tenant job-type config |
| [Results](./10-results.md) | Viewing a job's stored output |

## The shape every flow shares

Three patterns repeat across almost every feature, so they're named once here rather than
re-explained ten times:

1. **Draft → apply, not per-keystroke.** Free-text ID lookups (Batch ID, an old-style
   tenant/member field) commit on Enter or blur, never on every keystroke. Anywhere a real
   directory exists to search instead (tenants, members), a `SearchPicker` replaces the
   free text — see the [Budget](./08-budget.md) flow for where that came from.
2. **Async job = submit → poll/watch → terminal state.** Every "start a thing" action
   (an FWA job, an audit run, a bulk upload, a batch prediction, an OCR document) returns
   a job immediately and settles later. The generic AI-jobs table (`GET /ai/jobs/{id}`)
   is watched via `usePollingJob` — a WebSocket (`GET /ws/ai/jobs/{id}?token=`) that
   pushes updates, with 2-second HTTP polling as an always-on fallback if the socket
   never connects or drops early. Domain-specific job tables (OCR, bulk rule uploads,
   batch predictions) use their own dedicated polling hooks instead, since their payloads
   don't fit the generic job shape.
3. **A pane, not a modal, for CRUD.** Every list screen (Jobs, Audit rules, Providers,
   Model catalog, Cost predictions, ...) opens a detail pane docked to the right for
   create/edit/view, keeping the list visible and selectable behind it. Confirmation
   dialogs (`window.confirm`) are the only true modal interruption, reserved for delete.
