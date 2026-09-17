# Admin setup: providers, model catalog, tenant config

The platform-admin layer: register an LLM provider, price its models, then point a
tenant's job types at a provider/model/temperature combination.

```mermaid
flowchart TD
    Providers["Providers list<br/>GET /ai/providers"]
    Providers -->|"Add provider"| NewProvider["Blank ProviderForm"]
    Providers -->|"click a row"| EditProvider["GET /ai/providers/:name<br/>→ prefilled ProviderForm<br/>+ ProviderModelsList nested below"]
    EditProvider --> Models["GET /ai/providers/:name/models<br/>(live from the provider itself)"]
    Models -->|empty| ModelsEmpty["'No models listed for this provider.<br/>Check the base URL and API key above.'"]

    NewProvider -->|Save| CreateProvider["POST /ai/providers"]
    EditProvider -->|Save| UpdateProvider["PUT /ai/providers/:name"]
    CreateProvider -->|success| ProviderSaved["Notice: 'Provider saved'"]
    UpdateProvider -->|success| ProviderSaved
    EditProvider -->|Delete, confirm| DeleteProvider["DELETE /ai/providers/:name"]

    Catalog["Model catalog list<br/>GET /ai/model-catalog<br/>(client-sorted by tier, no server pagination)"]
    Catalog -->|"Add"| NewPrice["Blank ModelPriceForm"]
    Catalog -->|"click a row"| EditPrice["GET /ai/model-catalog/:provider/:model<br/>→ prefilled (provider/model immutable in edit mode)"]
    NewPrice -->|Save| CreatePrice["POST /ai/model-catalog"]
    EditPrice -->|Save| UpdatePrice["PUT /ai/model-catalog/:provider/:model"]
    EditPrice -->|Delete, confirm| DeletePrice["DELETE /ai/model-catalog/:provider/:model"]

    TenantConfig["Tenant config screen<br/>GET /ai/config/types (job types)"]
    TenantConfig -->|internal staff| TCPicker["Tenant SearchPicker —<br/>replaces the old free-text tenant lookup"]
    TenantConfig --> TCList["GET /ai/tenants/:id/config —<br/>merged with every known job type;<br/>a type with no row of its own shows<br/>'Falls back to All jobs'"]
    TCList -->|"click a job type row"| TCForm["JobTypeConfigForm:<br/>Provider (select) →<br/>Model (GET /ai/models?provider=X)<br/>→ temperature, max_tokens,<br/>use_text_extraction, text_extractor"]
    TCForm -->|Save| TCUpsert["PUT /ai/tenants/:id/config/:jobType<br/>(create and edit are the same call)"]
    TCUpsert -->|success| TCSaved["Tenant's config query invalidated,<br/>notice: 'Config saved'"]
```

## Details

- **A provider's models are nested in its own edit pane**, not a separate screen — you
  can't manage a provider's model list without first opening that provider.
- **`api_key` never round-trips.** The field always starts blank; leaving it blank on
  save omits it from the request entirely so an existing key is preserved ("Leave blank
  to keep the current key.").
- **The model-catalog list has no server-side pagination or sort** — it fetches
  everything and sorts by tier client-side, unlike every other list in the app.
- **A stale saved provider/model still shows as an option** in the tenant config form even
  if it's since dropped out of the live provider/model lists, so the form never silently
  "loses" what's actually configured.
- **"All jobs" is a real row, not a UI label** — every tenant has (or can have) a
  job-type row named `all` that every other job type without its own row inherits at
  runtime. There's no explicit "remove override" action; the only way back to the
  fallback is presumably re-pointing that specific job type's row, which this form
  doesn't expose as a delete.
