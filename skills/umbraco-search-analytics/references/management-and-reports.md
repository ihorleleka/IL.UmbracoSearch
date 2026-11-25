# Backoffice management, reports and synonyms

## Management endpoints

```csharp
app.MapSearchAnalyticsManagement(authorizationPolicy: "SearchInsights");   // base path /umbraco/api/search-analytics
```

- Without a policy the endpoints still require an authenticated Umbraco backoffice user. Define the policy in the host, for example `AddAuthenticationSchemes(Constants.Security.BackOfficeAuthenticationType).RequireAuthenticatedUser()`.
- Routes (all under the base path): `GET overview`, `index-info` (Azure only), `queries`, `content`, `content-item`, `sessions`, `session-insights`, `sites`; `POST data/delete`; synonym routes (below). The Search section of the backoffice calls these; call them directly only for custom tooling.
- Reports take a reporting window (`fromUtc`, `toUtc`) and optional `indexName`, `languageIsoCode`, `siteKey`. Session and refinement views use raw events and are therefore limited by `RawEventRetention`; they never expose session tokens, and refinement analysis samples at most the latest 10,000 matching events (marked as sampled).
- Paging on `queries`, `content` and `sessions`: `page` is at least 1 and `pageSize` is 1-100 (out-of-range values are clamped). Adding `csv=true` to `queries` or `content` exports every matching row.
- CSV cells that start with `=`, `+`, `-`, `@`, a tab or a carriage return are prefixed with `'` so spreadsheet tools treat visitor query text as text.

## Retention and permanent deletion

- Raw events (impressions, executions, clicks, pending clicks) older than `RawEventRetention` are purged daily on the node that runs background processing. Daily summaries stay, so trends remain after raw details expire; days whose raw detail expired before summaries existed are shown as availability gaps, not as zeroes.
- `POST data/delete` permanently removes collected analytics data (raw interactions, summaries, sync state, availability gaps, retention audits). The body must contain the exact confirmation `DELETE`, otherwise a validation problem is returned. Synonym mappings are kept.

## Azure synonyms

Azure only; Lucene has no synonym support here, and the endpoints answer `501` without the Azure feature.

- Editors manage mappings in the Search section (routes `synonyms`, `synonyms/{id}`, `synonyms/{id}/enabled`, `synonyms/publish`, `synonyms/publications`, `synonyms/publications/delete`). Equivalent terms publish as `a,b`, directional ones as `a => b`; source terms must be unique across mappings.
- Invalid mappings (empty terms, a term on both sides) return `400` with a validation problem keyed `mapping`; a duplicate source returns `409`; an unknown id returns `404`; publishing without index write permissions or without a managed index returns `409`.
- Publishing creates or updates one shared map named `${SynonymMapPrefix}library-managed-all` (prefix `us-` by default) in place, updates each managed index definition once to reference it, then deletes other maps with that prefix, so searches keep their synonyms throughout. It uploads or reindexes no documents.
- `SynonymFieldNames` (default `searchTitle`) lists the searchable fields that use the map.
- Environments that share one Azure Search service must use different `SynonymMapPrefix` values (for example `staging-us-` and `production-us-`), because maps are deleted by prefix.
