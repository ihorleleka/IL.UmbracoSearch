# Clicks and front end

## Click endpoint

Map the consumer-owned endpoint once at startup:

```csharp
app.UseSearchAnalyticsClicks();   // POST /api/search/analytics/click (pass a path argument to change it)
```

The host owns its authentication, transport security, rate limiting and consent policy; chain `RequireRateLimiting(...)` or similar on the returned `RouteHandlerBuilder`.

JSON body:

| Field | Meaning |
| --- | --- |
| `trackingReference` | From `SearchResults.Analytics.TrackingReference` of the search that displayed the result. |
| `nodeKey` | Umbraco content **GUID** (never the numeric id). |
| `position` | 1-based position shown to the user. The stored position is the one recorded for the displayed result. |
| `idempotencyKey` | Unique per click attempt (for example a new GUID); makes retries safe. |

Outcomes: `202 Accepted` (recorded or queued), `204 No Content` (duplicate click), `400 Bad Request` (invalid, expired or non-displayed click). A click is valid only for a tracking reference the server issued, within `TrackingReferenceLifetime`, for a node that was actually displayed in that search. Rejected clicks have no side effects. The endpoint records clicks fire-and-forget, so a client disconnect does not abort recording.

If the search was not persisted yet when the click arrives, the click is accepted and resolved shortly afterwards by the background node, so do not retry on `202`.

## npm helper

```bash
npm i @ihorleleka/umbraco-search-analytics
```

```ts
import { trackSearchResultClick } from '@ihorleleka/umbraco-search-analytics';

const sent: boolean = await trackSearchResultClick(
  { endpoint: '/api/search/analytics/click', consent: () => hasAnalyticsConsent() },
  {
    trackingReference: search.analytics.trackingReference,
    nodeKey: item.nodeKey,
    position: index + 1,
    idempotencyKey: crypto.randomUUID()
  });
```

- `consent()` is checked first; without consent nothing is sent and the call resolves to `false`.
- The helper prefers `navigator.sendBeacon`, then falls back to `fetch` with `keepalive`; it never throws and resolves to `true` when the click was sent.
- The package also exports the request/response types (`SearchAnalyticsRequest`, `SearchAnalyticsResponse`, `SearchResultClick`) for front ends that call the search endpoint over HTTP.

## Requesting analytics from the front end

- For `UseSearch`, set `SearchRequest.Analytics` (`captureAnalytics`, `analyticsSessionToken`, `siteKey`); the response carries `analytics.trackingReference` and `analytics.analyticsSessionToken`.
- Suggestion requests work the same way through `SuggestionsSearchRequest.Analytics`; the response is the `SuggestionResult` envelope with `Analytics`, `TotalItemCount` and `Items`.
- Store the returned session token (for example in memory or per-visit storage) and send it with later searches so they form one session. Tokens do not expire, are never shown in reports, and a value that is not a GUID is replaced by a new one.

## SSR and hydration

A tracking reference is generated per executed search. If the same logical search runs on the server and again on the client, analytics records two executions. Reuse the server-rendered results during hydration, or set `Analytics = new AnalyticsRequest { CaptureAnalytics = false }` on the non-user-visible call. Do the same for replayed or system searches.
