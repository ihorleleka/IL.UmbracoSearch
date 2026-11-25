---
name: umbraco-search-analytics
description: Use when adding or operating Search Insights with the IL.UmbracoSearch.Analytics NuGet package in an Umbraco 17/18 site - SearchOptions.Analytics, SearchParameters.Analytics / AnalyticsRequest, tracking references and session tokens, consent-gated click tracking (UseSearchAnalyticsClicks, the @ihorleleka/umbraco-search-analytics npm helper), the backoffice Search section (MapSearchAnalyticsManagement), analytics storage and migrations, multi-server flags, or Azure synonym management.
---

# IL.UmbracoSearch.Analytics

Optional Search Insights for `IL.UmbracoSearch`: it decorates `ISearchService` to record search and suggestion executions and consented result clicks, stores them in SQL, and adds a Search section to the Umbraco backoffice (overview, queries, content, sessions, synonyms, settings). It never changes query or result behavior. Requires the main package (see the `umbraco-search` skill); same framework pairing (.NET 10 with Umbraco 17.8.1+, .NET 11 with Umbraco 18.3.1+).

## Non-negotiable contracts

- Enable `SearchOptions.Analytics` together with **exactly one** engine (`Lucene` or `Azure`). Any other combination throws `InvalidOperationException` at startup and on every search.
- Capture is per request and opt-in: it happens only when `SearchParameters.Analytics` (or `SuggestionsSearchRequest.Analytics`) is supplied with `CaptureAnalytics = true`. Otherwise `SearchResults.Analytics` is `null`. Set `CaptureAnalytics = false` on server-generated, internal or replayed searches.
- Every captured search gets its own `TrackingReference`; use it for click tracking. Keep the returned `AnalyticsSessionToken` in the caller and send it back on later searches to group them into one session. `SiteKey` (a GUID) partitions data by website.
- Impressions and clicks use Umbraco content **GUID node keys** only, never numeric ids.
- The application owns the public click endpoint's authentication, rate limiting, transport security and **consent**. Send a click only after consent.
- The same logical search executed twice (for example during SSR and again on hydration) is counted twice: reuse SSR results or disable capture on the second call.
- Analytics failures never fail a search; a full capture queue drops captures.
- Create schema changes only with `dotnet ef`; never edit migration files.

## Minimum setup

```csharp
builder.AddServiceAttributeBasedDependencyInjection(options =>
{
    options.AddFeature(SearchOptions.Lucene);     // or Azure - exactly one engine
    options.AddFeature(SearchOptions.Analytics);
});

builder.Services.AddAuthorization(options =>
    options.AddPolicy("SearchInsights", policy =>
        policy.AddAuthenticationSchemes(Constants.Security.BackOfficeAuthenticationType)
              .RequireAuthenticatedUser()));

app.UseSearchAnalyticsClicks();                                           // POST /api/search/analytics/click
app.MapSearchAnalyticsManagement(authorizationPolicy: "SearchInsights");  // /umbraco/api/search-analytics
```

Request analytics on the searches the user sees and read the response:

```csharp
var results = await searchService.SearchAsync<CommonSearchItemModel>(new SearchParameters
{
    FullTextSearch = "umbraco",
    Take = 10,
    Analytics = new AnalyticsRequest { AnalyticsSessionToken = sessionToken, SiteKey = siteKey }
});
var trackingReference = results.Analytics?.TrackingReference;     // for clicks
var nextSessionToken = results.Analytics?.AnalyticsSessionToken;  // reuse on the next search
```

Then add a connection string (`ConnectionStrings:UmbracoSearchAnalytics`, falling back to `umbracoDbDSN`) and send consented clicks from the front end. Only the designated backoffice server should run background work; see `references/setup-and-configuration.md`.

## Where to read next

- `references/setup-and-configuration.md` - options, storage, migrations, multi-server nodes, behavior when the feature is off.
- `references/clicks-and-frontend.md` - click endpoint, npm helper, payload and outcomes, SSR/hydration, consent.
- `references/management-and-reports.md` - backoffice endpoints and authorization, report paging and CSV, sessions, Azure synonyms, retention and data deletion.
