# IL.UmbracoSearch.Analytics

Optional Search Insights for [IL.UmbracoSearch](https://www.nuget.org/packages/IL.UmbracoSearch). It decorates `ISearchService` to record searches and consented result clicks in SQL, and adds a **Search** section to the Umbraco backoffice: overview, queries, content, sessions, Azure synonyms and settings. It never changes query or result behavior.

Targets **.NET 10 with Umbraco 17.8.1+ (<18)** and **.NET 11 with Umbraco 18.3.1+ (<19)**, matching the main package. Umbraco 13-16 and .NET 8-9 are not supported.

## Enable

Add `SearchOptions.Analytics` next to **exactly one** engine:

```csharp
builder.AddServiceAttributeBasedDependencyInjection(options =>
{
    options.AddFeature(SearchOptions.Lucene);      // or SearchOptions.Azure
    options.AddFeature(SearchOptions.Analytics);
});
```

(`using IL.AttributeBasedDI.Extensions;`.) Any other combination fails at startup. Without `SearchOptions.Analytics` the package is inert.

## Map the endpoints

```csharp
app.UseSearchAnalyticsClicks();                                           // POST /api/search/analytics/click
app.MapSearchAnalyticsManagement(authorizationPolicy: "SearchInsights");  // /umbraco/api/search-analytics
```

Define the `SearchInsights` policy in your host (for example backoffice authentication scheme plus an authenticated user). Without a policy the management endpoints still require a signed-in backoffice user. Your application owns authentication, rate limiting and consent for the click endpoint.

## Record searches and clicks

Capture is requested per search:

```csharp
var results = await searchService.SearchAsync<CommonSearchItemModel>(new SearchParameters
{
    FullTextSearch = "umbraco",
    Take = 10,
    Analytics = new AnalyticsRequest { AnalyticsSessionToken = sessionToken }
});
```

`results.Analytics` returns a `TrackingReference` (use it for clicks) and an `AnalyticsSessionToken` (send it with later searches to form a session). Post consented clicks from the browser with the npm helper:

```bash
npm i @ihorleleka/umbraco-search-analytics
```

```ts
import { trackSearchResultClick } from '@ihorleleka/umbraco-search-analytics';

await trackSearchResultClick(
  { endpoint: '/api/search/analytics/click', consent: () => hasAnalyticsConsent() },
  { trackingReference, nodeKey: item.nodeKey, position: index + 1, idempotencyKey: crypto.randomUUID() });
```

If the same search runs during server rendering and again on hydration it is counted twice: reuse the server results or set `CaptureAnalytics = false` on the repeat.

## Configure

Options live under `SearchSettings:Analytics`; the connection string comes from `ConnectionStrings:UmbracoSearchAnalytics` (falling back to `umbracoDbDSN`).

```json
"SearchSettings": { "Analytics": { "Enabled": true, "EnableBackgroundProcessing": true, "RawEventRetention": "90.00:00:00" } }
```

On delivery servers of a multi-server setup set `EnableBackgroundProcessing` to `false`; on preview/test servers set `Enabled` to `false`. Migrations are applied automatically on the node that runs background processing.

## Learn more

| Topic | Page |
| --- | --- |
| All options, storage, migrations, multi-server nodes | [setup and configuration](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search-analytics/references/setup-and-configuration.md) |
| Click endpoint, npm helper, SSR/hydration, consent | [clicks and front end](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search-analytics/references/clicks-and-frontend.md) |
| Backoffice endpoints, paging and CSV, retention, Azure synonyms | [management and reports](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search-analytics/references/management-and-reports.md) |

AI coding agents: use the [`umbraco-search-analytics` skill](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search-analytics/SKILL.md).
