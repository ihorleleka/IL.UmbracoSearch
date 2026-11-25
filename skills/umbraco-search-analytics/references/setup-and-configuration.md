# Setup and configuration

## Options (`SearchSettings:Analytics`)

Options bind from `SearchSettings:Analytics`; `UmbracoSearch:Analytics` is accepted only when the former section is absent.

```json
{
  "ConnectionStrings": { "UmbracoSearchAnalytics": "Server=...;Database=SearchAnalytics;..." },
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

| Option | Default | Meaning |
| --- | --- | --- |
| `Enabled` | `true` | `false` stops all capture on this node (preview/test servers). |
| `EnableBackgroundProcessing` | `true` | `false` skips background work on this node (see below). |
| `CaptureQueryText` | `true` | Store query text (after the sanitizer runs) so it can be reported. |
| `QueueCapacity` | 2000 | Bounded in-process capture queue; a full queue drops captures and logs a rate-limited warning. |
| `FlushInterval` | 1 second | How often captured batches are written. |
| `TrackingReferenceLifetime` | 1 hour | How long a tracking reference accepts clicks. |
| `RawEventRetention` | 90 days | Raw events older than this are purged; session insights and window reports depend on retained raw events. |
| `ConnectionString` | none | Explicit connection string override. |
| `UseInMemoryStorage` | `false` | Allows the in-memory store for tests and local experiments only. |
| `SynonymFieldNames` | `["searchTitle"]` | Azure fields that receive the generated synonym map. |
| `SynonymMapPrefix` | `us-` | Prefix of the shared Azure synonym map and its cleanup scope. |

`CaptureAnalytics` is not an appsetting; it is the per-request flag on `AnalyticsRequest`.

## Storage

- SQL Server. Connection string resolution order: `SearchSettings:Analytics:ConnectionString`, then `ConnectionStrings:UmbracoSearchAnalytics`, then `ConnectionStrings:umbracoDbDSN`. With none, startup fails unless `UseInMemoryStorage` is `true`.
- The package uses its own migrations-history table and does not touch Umbraco tables.
- Pending migrations are applied automatically at startup on nodes where the feature is active and `EnableBackgroundProcessing` is `true`. Never hand-write migration files; generate schema changes with `dotnet ef`.
- The legacy `DailyAggregates` table is no longer populated or read by reports; permanent data deletion still clears it.
- Replaceable seams (registered in DI): `IAnalyticsSanitizer`, `IAnalyticsConsentProvider`, `IAnalyticsStore`, `IAnalyticsCaptureQueue`, `IAnalyticsClock`. Custom implementations must not store IP addresses or user names. The default sanitizer redacts emails and long numbers and normalizes and lower-cases query text.

## Multi-server deployments

| Node | `Enabled` | `EnableBackgroundProcessing` | What runs |
| --- | --- | --- | --- |
| Master / backoffice | `true` | `true` | Migrations, retention, historical summaries, pending-click processing, capture |
| Delivery | `true` | `false` | Capture and click recording only |
| Preview / test | `false` | irrelevant | Nothing |

`EnableBackgroundProcessing = false` skips startup migrations, retention purge, historical summary sync and pending-click processing. Keep it `true` on exactly one designated backoffice node. A delivery node's clicks that arrive before their search is stored are parked as pending and resolved by the master node.

## When the feature is not enabled

If the package is referenced but `SearchOptions.Analytics` is not added, it is inert: no database migration, no workers, no DI services resolved. `MapSearchAnalyticsManagement()` maps no routes and `UseSearchAnalyticsClicks()` maps only a stub that answers `404 Not Found`.
