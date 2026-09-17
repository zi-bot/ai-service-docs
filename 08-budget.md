# Budget / spend & usage

The monthly cap, tenant-wide token totals, and a per-member usage table — three related
but independently-scoped pieces of the same screen.

```mermaid
flowchart TD
    Page["Spend and usage screen"]
    Page -->|internal staff only| TenantPicker["Tenant SearchPicker —<br/>'Your tenant — search another by name'"]
    TenantPicker -->|pick another tenant| Rescope["tenantId changes; selected member cleared,<br/>cap-edit pane closed, usage page reset to 1<br/>(a member id is tenant-scoped)"]

    Page --> Cap["'Monthly cap' tile<br/>GET /ai/tenants/:id/budget"]
    Cap -->|200| CapWords{is_active / hard_cap}
    CapWords -->|"!is_active"| NoCap["'No cap set. Jobs run<br/>without a spend limit.'"]
    CapWords -->|hard_cap| Hard["'Hard cap — jobs stop<br/>when the limit is reached'"]
    CapWords -->|soft cap| Soft["'Soft cap — jobs keep<br/>running past the limit'"]
    Cap -->|404| NoBudgetYet["Treated as 'no cap configured' —<br/>not an error; tile still renders,<br/>'Set monthly cap' button still shows"]
    Cap -->|other error| CapFailed["ErrorState in the cap tile only —<br/>rest of the page still renders"]

    Page --> Summary["'Total tokens this month' tile<br/>GET /ai/tenants/:id/token-usages/summary<br/>(from = month start, to = now)"]

    Page -->|"Set/Edit monthly cap"| BudgetForm["Monthly limit (USD), Hard cap,<br/>Active — pre-filled if a budget exists"]
    BudgetForm -->|Save| Upsert["PUT /ai/tenants/:id/budget<br/>(one endpoint for create and edit)"]
    Upsert -->|success| CapSaved["Budget query invalidated,<br/>pane closes, notice: 'Cap saved'"]

    Page --> MemberPicker["Member SearchPicker<br/>(disabled with no tenantId)"]
    MemberPicker -->|pick a member| Scoped["Heading: 'Usage for {name}'.<br/>Usage table switches to<br/>listMemberUsage — cap/summary<br/>tiles stay tenant-wide (noted inline)"]
    MemberPicker -->|clear| AllMembers["Heading: 'Usage for all members'"]

    Scoped --> Table["Usage table:<br/>When, Job, Type, Provider, Model, In/Out/Total"]
    AllMembers --> Table
    Table -->|empty| NoUsage["'No token usage yet.<br/>It appears here when jobs run.'"]
```

## Details

- **The 404-means-nothing-configured-yet pattern reappears here**: a budget 404 shows the
  no-cap copy and keeps the "Set monthly cap" action available, rather than treating a
  fresh tenant's lack of a budget as an error state.
- **Any other budget-fetch failure stays scoped to that one tile.** `ErrorState` replaces
  just the cap tile's contents; the summary tile, the member picker, and the usage table
  all keep working — a real fix for an earlier bug where any non-404 failure blanked the
  entire page.
- **Switching tenants resets the member filter.** A member id only means something within
  the tenant it was searched in; carrying it across a tenant switch would silently scope
  the usage table to a member from the wrong tenant.
- **The usage endpoints don't return the app's usual pagination envelope.** They answer
  `{data: {items, total}}` instead of `{data: [...], _metadata: {pagination}}` — the
  client reshapes this (`toUsagePage()` in `budget/api.js`) into the standard shape before
  it reaches the table, computing `total_pages` from `total`/`page_size` since the backend
  doesn't send it.
