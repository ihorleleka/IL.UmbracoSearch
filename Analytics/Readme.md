# IL.UmbracoSearch.Analytics

Optional Search Insights for IL.UmbracoSearch. This package targets **Umbraco
17 / .NET 10 only**. Install it and enable `SearchOptions.Analytics` together
with exactly one search engine.

```csharp
builder.Services.AddUmbracoSearch(SearchOptions.Lucene | SearchOptions.Analytics);
// or
builder.Services.AddUmbracoSearch(SearchOptions.Azure | SearchOptions.Analytics);
```

Analytics decorates `ISearchService`; it does not change its query or result
semantics. Completed searches receive a `TrackingReference` and are captured
through a bounded in-process queue, so persistence errors never fail a search.
Only GUID `UmbracoNodeKey` values are stored for impressions and clicks.

## Storage and privacy

SQL Server is the default provider. Set `ConnectionStrings:UmbracoSearchAnalytics`
to use a separate database; otherwise the package uses Umbraco's `umbracoDbDSN`.
`UseInMemoryStorage` is solely for tests/local experiments. Configure a custom
`IAnalyticsStore`, `IAnalyticsSanitizer`, or `IAnalyticsConsentProvider` to
replace the defaults. Raw events are retained for 90 days by default; query text
can be disabled with `CaptureQueryText`. Do not store an IP address or a user
name in custom implementations; search text may itself contain personal data.

```json
{
  "ConnectionStrings": {
    "UmbracoSearchAnalytics": "Server=...;Database=SearchAnalytics;..."
  },
  "UmbracoSearch": {
    "Analytics": {
      "RawEventRetention": "90.00:00:00",
      "CaptureQueryText": true,
      "EnableAzureTelemetry": false
    }
  }
}
```

## Database migrations

Analytics schema changes are generated only with the EF `dotnet` tool. The
`Microsoft.EntityFrameworkCore.Design` references are deliberately commented
out in the package project: temporarily enable the matching target-framework
reference, run `dotnet ef migrations add <Name> --framework net10.0` (and
`dotnet ef database update --framework net10.0` where appropriate), then
comment the reference again before packaging. Do not hand-author migration
files. The package uses its own EF migrations history table, so analytics
migrations do not share Umbraco's history.

Before updating a production schema, take the normal SQL backup and run the
migration against a staging copy. Analytics records are disposable according to
the configured retention policy; the package does not modify Umbraco tables.

Map the consumer-owned click endpoint explicitly:

```csharp
app.UseSearchAnalyticsClicks();
```

The host remains responsible for endpoint authentication, rate limiting, and
consent. The npm helper is published as `@ihorleleka/umbraco-search-analytics` and
posts `{ trackingReference, nodeKey, position, idempotencyKey }` only when the
consumer-supplied consent callback returns true.

```ts
import { trackSearchResultClick } from '@ihorleleka/umbraco-search-analytics';

await trackSearchResultClick(
  { endpoint: '/api/search/analytics/click', consent: () => hasAnalyticsConsent() },
  { trackingReference: search.trackingReference, nodeKey: item.nodeKey, position: index + 1, idempotencyKey: crypto.randomUUID() });
```

`nodeKey` must be the Umbraco content GUID—not a numeric node ID. Expired,
unknown, duplicate, or non-displayed click events are safely rejected.

Map the backoffice management routes separately. They require an authenticated
user by default; pass the host's backoffice Search policy to restrict access to
the appropriate Umbraco user groups:

```csharp
app.MapSearchAnalyticsManagement(authorizationPolicy: "SearchInsights");
```

## Azure synonyms

Synonym management is Azure-only. Publication creates an immutable versioned
Azure synonym map and refreshes the active/indexing physical index definitions;
it never uploads or reindexes documents. A failed publication is recorded while
the previously active map remains assigned. Configure `SynonymFieldNames` to
select the Azure searchable fields that receive the map.

## Azure operational telemetry

Where Azure Monitor or Log Analytics already has useful Search operational
metrics, register an `IAzureSearchTelemetryReader` and set
`UmbracoSearch:Analytics:EnableAzureTelemetry` to `true`. The package invokes
readers only from a background import service (every five minutes by default),
never while serving a search. Readers use the host's existing Azure credentials
and return aggregates, so the overview can show Azure request/failure/latency
figures separately from interaction analytics without double counting them.
