# IL.UmbracoSearch.Analytics

Optional Search Insights for IL.UmbracoSearch. This package targets **Umbraco
17 / .NET 10 only**.

## Enable Analytics

Enable `SearchOptions.Analytics` with exactly one engine:

```csharp
builder.Services.AddUmbracoSearch(SearchOptions.Lucene | SearchOptions.Analytics);
// or
builder.Services.AddUmbracoSearch(SearchOptions.Azure | SearchOptions.Analytics);
```

Analytics decorates `ISearchService`; it does not change query/result semantics.
Completed searches may receive a `TrackingReference`. Capture is non-blocking and
stores only GUID `UmbracoNodeKey` values for impressions/clicks.

Per-call capture opt-out is available through `SearchParameters.CaptureAnalytics`.
Set it to `false` on individual searches that must not be captured (for example,
server-generated/internal searches).

## Consumer setup checklist

1. Enable Analytics with exactly one engine.
2. Configure analytics storage.
3. Map click tracking endpoint (`UseSearchAnalyticsClicks`).
4. Map backoffice management endpoints (`MapSearchAnalyticsManagement`).
5. Apply a backoffice authorization policy for Search Insights access.
6. Install and use the npm click helper package in your front end.

## Configuration

The package binds options from `UmbracoSearch:Analytics`.

```json
{
  "ConnectionStrings": {
    "UmbracoSearchAnalytics": "Server=...;Database=SearchAnalytics;..."
  },
  "UmbracoSearch": {
    "Analytics": {
      "Enabled": true,
      "CaptureQueryText": true,
      "QueueCapacity": 2000,
      "FlushInterval": "00:00:01",
      "TrackingReferenceLifetime": "01:00:00",
      "RawEventRetention": "90.00:00:00",
      "EnableAzureTelemetry": false,
      "AzureTelemetryImportInterval": "00:05:00",
      "SynonymFieldNames": ["searchTitle"]
    }
  }
}
```

Supported `UmbracoSearch:Analytics` options:

- `Enabled`
- `CaptureQueryText`
- `QueueCapacity`
- `FlushInterval`
- `TrackingReferenceLifetime`
- `RawEventRetention`
- `ConnectionString` (explicit override)
- `UseInMemoryStorage` (tests/local experiments only)
- `EnableAzureTelemetry`
- `AzureTelemetryImportInterval`
- `SynonymFieldNames`

`CaptureAnalytics` is not an appsetting; it is a per-request flag on
`SearchParameters` (and the default HTTP `SearchRequest`) for selective capture.

Storage defaults:

- SQL Server is default.
- Connection resolution order:
  1. `UmbracoSearch:Analytics:ConnectionString`
  2. `ConnectionStrings:UmbracoSearchAnalytics`
  3. `ConnectionStrings:umbracoDbDSN`
- If none are set, startup fails unless `UseInMemoryStorage=true`.

## Database migrations

Generate analytics migrations with EF tooling (`dotnet ef`); do not hand-author
migration files. Analytics uses its own migrations history table and does not
modify Umbraco tables.

## Click endpoint and npm helper

Map the consumer-owned click endpoint:

```csharp
app.UseSearchAnalyticsClicks();
```

Default path: `/api/search/analytics/click`

Install helper package:

```bash
npm i @ihorleleka/umbraco-search-analytics
```

Use the helper:

```ts
import { trackSearchResultClick } from '@ihorleleka/umbraco-search-analytics';

await trackSearchResultClick(
  { endpoint: '/api/search/analytics/click', consent: () => hasAnalyticsConsent() },
  {
    trackingReference: search.trackingReference,
    nodeKey: item.nodeKey,
    position: index + 1,
    idempotencyKey: crypto.randomUUID()
  });
```

Click payload:

- `trackingReference`: from the search response
- `nodeKey`: Umbraco content GUID (not numeric node ID)
- `position`: 1-based position shown to the user
- `idempotencyKey`: unique per click attempt

Endpoint outcomes:

- `202 Accepted` for accepted click
- `204 No Content` for duplicate click
- `400 Bad Request` for invalid/expired/non-displayed click

The host owns endpoint authentication, transport security, rate limiting, and consent policy.

## Backoffice management endpoints

Map Search Insights management endpoints:

```csharp
app.MapSearchAnalyticsManagement(authorizationPolicy: "SearchInsights");
```

Default base path: `/umbraco/api/search-analytics`

If no policy is supplied, endpoints still require an authenticated Umbraco
backoffice user via `BackOfficeAuthenticationType`.

Example policy wiring:

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("SearchInsights", policy =>
        policy.AddAuthenticationSchemes(Constants.Security.BackOfficeAuthenticationType)
              .RequireAuthenticatedUser());
});
```

## SSR and hydration guidance

`TrackingReference` is generated per executed search call. If the same logical
search is executed in SSR and then re-executed during hydration, analytics will
record two executions.

To avoid duplicate analytics with the current implementation, reuse SSR search
results during hydration instead of issuing a second identical search call, or
set `CaptureAnalytics = false` on non-user-visible/server-generated searches.

You can also suppress analytics for specific replay/internal searches by setting
`SearchParameters.CaptureAnalytics = false` for those calls.

## Azure synonyms

Synonym management is Azure-only. Publication creates immutable versioned Azure
synonym maps and refreshes active/indexing index definitions; it does not upload
or reindex content documents.

## Azure operational telemetry

If you provide `IAzureSearchTelemetryReader` implementations and enable
`UmbracoSearch:Analytics:EnableAzureTelemetry`, telemetry imports run in a
background service (default every 5 minutes), never on the request path.
