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

Captured searches also return an `AnalyticsSessionToken`. Store it in the caller
and pass it as `SearchParameters.AnalyticsSessionToken` on a later search to
group those executions as one **Search session**. The package does not expire
session tokens; omit the value to begin a new session. `TrackingReference`
remains distinct for every execution and must still be used for click tracking.

## Session insights

The Search section includes an anonymous **Sessions** view. It reports activity
only within the selected reporting window: searches per session, refined-session
rate, paginated journeys, and frequent query-to-query refinements. It never
returns or displays `AnalyticsSessionToken` values.

Journey rows are paginated and only include recent query/click activity. Refinement
analysis is limited to the most recent 10,000 matching raw events and indicates
when a wide reporting window is sampled. These insights depend on raw retained
events and are not available beyond `RawEventRetention`.

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

The package binds options from `SearchSettings:Analytics`.
For backward compatibility, `UmbracoSearch:Analytics` is also accepted when
`SearchSettings:Analytics` is not present.

```json
{
  "ConnectionStrings": {
    "UmbracoSearchAnalytics": "Server=...;Database=SearchAnalytics;..."
  },
  "SearchSettings": {
    "Analytics": {
      "Enabled": true,
      "EnableBackgroundProcessing": true,
      "CaptureQueryText": true,
      "QueueCapacity": 2000,
      "FlushInterval": "00:00:01",
      "TrackingReferenceLifetime": "01:00:00",
      "RawEventRetention": "90.00:00:00",
      "SynonymFieldNames": ["searchTitle"]
    }
  }
}
```

Supported `SearchSettings:Analytics` options:

- `Enabled` — set to `false` on preview/test servers to disable all capture on that node
- `EnableBackgroundProcessing` — set to `false` on delivery servers in a multi-server setup so only the designated master backend runs periodic tasks (retention purge); capture and ingestion are unaffected
- `CaptureQueryText`
- `QueueCapacity`
- `FlushInterval`
- `TrackingReferenceLifetime`
- `RawEventRetention`
- `ConnectionString` (explicit override)
- `UseInMemoryStorage` (tests/local experiments only)
- `SynonymFieldNames`

`CaptureAnalytics` is not an appsetting; it is a per-request flag on
`SearchParameters` (and the default HTTP `SearchRequest`) for selective capture.

Storage defaults:

- SQL Server is default.
- Connection resolution order:
  1. `SearchSettings:Analytics:ConnectionString`
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

## Multi-server deployments

In a load-balanced or multi-server setup, use `Enabled` and `EnableBackgroundProcessing`
to control per-node behaviour:

| Node type | `Enabled` | `EnableBackgroundProcessing` | What runs |
|---|---|---|---|
| Master / backoffice server | `true` (default) | `true` (default) | Migrations, retention, capture |
| Delivery server | `true` (default) | `false` | Capture only |
| Preview / test server | `false` | *(irrelevant)* | Nothing |

`EnableBackgroundProcessing = false` skips **all** server-side processing on that node:
database migrations and retention purge. Only the master
(backoffice) server should run with it set to `true`.

Example delivery server appsettings:

```json
{
  "SearchSettings": {
    "Analytics": {
      "EnableBackgroundProcessing": false
    }
  }
}
```

Example preview/test server appsettings:

```json
{
  "SearchSettings": {
    "Analytics": {
      "Enabled": false
    }
  }
}
```

## Safety when the feature is not activated

If `IL.UmbracoSearch.Analytics` is referenced but `SearchOptions.Analytics` is
never added during service registration, the package is inert:

- The database migration hosted service exits without running.
- No background workers start (capture queue, retention).
- `UseSearchAnalyticsClicks()` and `MapSearchAnalyticsManagement()` map no
  routes and resolve nothing from the DI container.

## Azure synonyms

Synonym management is Azure-only. Publication creates immutable versioned Azure
synonym maps and refreshes active/indexing index definitions; it does not upload
or reindex content documents.

## Local Storybook preview for Search section FE

Use the local Storybook workspace to evolve Search-section UI and fake-data
states without publishing a new package release.

Workspace path:

- `src/UmbracoSearch.Analytics.Storybook/`

Run locally:

```bash
cd src/UmbracoSearch.Analytics.Storybook
npm install
npm run storybook
```

Build preview artifacts:

```bash
npm run build-storybook
```

The preview uses a Storybook-only fake-data provider. Runtime backoffice
behavior remains unchanged: production continues to use authenticated Umbraco
fetch flow and existing management endpoints.
