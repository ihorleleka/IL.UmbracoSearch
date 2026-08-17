# IL.UmbracoSearch — NuGet consumer agent memory

Use this file as the implementation playbook when `IL.UmbracoSearch` is installed from NuGet and source code is unavailable. It records the public configuration, extension points, behavior contracts, and troubleshooting rules needed to build a search feature in an Umbraco application.

> [!IMPORTANT]
> This guide applies to **IL.UmbracoSearch 17.8.0.1+**. This is a breaking line: applications must use .NET 10 and Umbraco 17+. Umbraco 13-16 and .NET 8-9 are unsupported. `IIndexingConverter` computed-field methods and `IExternalIndexingConverter.GetIndexingModelsAsync` are asynchronous and return `Task`; converters in the same `Order` run concurrently, and order groups run sequentially.

## What this package is

`IL.UmbracoSearch` adds a search abstraction and extensible computed indexing to **existing Umbraco/Examine indexes**. It does not create Lucene indexes. The consuming application must have real Umbraco indexes (normally `ExternalIndex`, optionally `InternalIndex`) and configure the package to use them.

It provides:

- full-text search, typed filtering, sorting, boosts, facets, suggestions, and language-aware fields;
- custom indexing through `IIndexingConverter`;
- two mutually selected engine paths: Lucene or Azure AI Search;
- optional Azure hybrid keyword/vector search and optional chunk-level vectors;
- optional minimal HTTP endpoints and MCP search tools.

The main runtime service is `ISearchService`. The main customization seam is `IIndexingConverter`.

## Non-negotiable contracts

- Register **one appropriate engine feature**: `SearchOptions.Lucene` or `SearchOptions.Azure`.
- `SearchSettings.Indexes` contains the real Umbraco indexes this package may manage. In Azure mode it is also the lifecycle allow-list: do not configure or expect the package to provision/rebuild unrelated indexes.
- `DefaultIndexName` is used whenever a request omits `IndexName`.
- Azure requires a valid `LicenseToken` and fails initialization when it is invalid. Lucene does not require a license token.
- An absent or misspelled configured index can result in a logged error and empty results rather than an obvious startup failure. Include this in operational tests.
- Use typed index fields and typed filter helpers. The shape of an indexed field (scalar vs collection, multi-language, facetable/sortable) is part of the query contract.
- Do not rely on Lucene for hybrid/vector behavior; those are Azure-only capabilities.
- For Lucene full-text requests, supplied `SearchFields` are authoritative. If omitted, only declared fields with `isSearchable: true` are searched; configure at least one searchable field.
- The application that exposes HTTP or MCP endpoints owns authorization, authentication, rate limiting, tenancy, and public request limits.

## Install and register

Install the NuGet package in the Umbraco web application, then enable the desired feature using the attribute-based DI registration used by the package:

```csharp
builder.AddServiceAttributeBasedDependencyInjection(options =>
{
    options.AddFeature(SearchOptions.Lucene);
    // Or, for Azure AI Search:
    // options.AddFeature(SearchOptions.Azure);
});
```

Do not enable Azure-specific query/indexing behavior when only `SearchOptions.Lucene` is registered. Choose the engine from deployment requirements, not from an individual caller’s preference.

## Configuration

Place this under `SearchSettings` in application configuration. Keep tokens/keys in secret management, not committed settings.

```json
"SearchSettings": {
  "LicenseToken": "<license-token-required-for-azure>",
  "DefaultIndexName": "ExternalIndex",
  "Indexes": ["ExternalIndex", "InternalIndex"],
  "PreviewIndexes": ["InternalIndex"],
  "ReadOnly": false,
  "CustomIndexSuffix": "",
  "EnableBlueGreenIndexing": false,
  "DisableSwapDelay": false,
  "EnableTaxonomyFacetExpansion": false,
  "EnableTaxonomyFiltersExpansion": false,
  "Azure": {
    "ServiceUrl": "<azure-search-url>",
    "ApiKey": "<azure-search-key>",
    "UseHybridSearch": false,
    "UseChunkedVectorSearch": false,
    "ChunkedVectorMinTokens": 2000,
    "ChunkedVectorMaxTokens": 3000,
    "ChunkedVectorOverlapTokens": 200,
    "ChunkedVectorEmbeddingMaxConcurrency": 4,
    "BackgroundEmbeddingsProcessing": false,
    "DocumentBatchMaxActions": 100,
    "DocumentBatchMaxPayloadBytes": 14680064,
    "DocumentBatchFlushIntervalMilliseconds": 250,
    "DocumentBatchQueueCapacity": 2000,
    "DocumentBatchMaxRetries": 3,
    "BackgroundEmbeddingsMaxConcurrency": 4
  },
  "OpenAi": {
    "ServiceUrl": "<embedding-service-url>",
    "ApiKey": "<embedding-service-key>",
    "EmbeddingsDeploymentName": "text-embedding-3-large"
  },
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
```

| Setting | Consumer impact |
| --- | --- |
| `DefaultIndexName` | The default target for searches and suggestions without an explicit index. |
| `Indexes` | Existing Umbraco indexes this package can use/manage; Azure index lifecycle is restricted to this list. |
| `PreviewIndexes` | Indexes where preview/soft-delete semantics apply. |
| `ReadOnly` | Prevents package write/provisioning behavior; use for read-only application nodes. |
| `CustomIndexSuffix` | Distinguishes physical Azure indexes across environments. Keep it consistent per environment. |
| `EnableBlueGreenIndexing` | Azure rebuilds use alternate physical indexes before a swap; do not enable casually without an operational rebuild plan. |
| Taxonomy expansion flags | Opt into hierarchical taxonomy facets and compatible taxonomy `OR` filter rewriting. |
| `Azure.*` | Required Azure connection and throughput/vector settings when Azure feature is selected. |
| `OpenAi.*` | Required for generated embeddings when hybrid search is enabled. |
| `Analytics.*` | Optional Search Insights settings used when `SearchOptions.Analytics` is enabled. |

`ReadOnly`, server role, and `Indexes` are hard safety constraints for Azure operations. A search feature must not attempt to work around them.

## Fast decision guide

| Need | Use |
| --- | --- |
| Standard keyword search | Lucene or Azure + `ISearchService.SearchAsync<T>()` |
| Search by structured value | Typed `ISearchFilter` using the matching field shape |
| Search within a content type | `SearchParameters.Aliases` |
| Browse/filter UI counts | Facetable typed field + `FacetOn` |
| Hierarchical tags | `AddTaxonomyValue` + taxonomy fields/options |
| Search in a subtree | `SearchParameters.Root` |
| Local/relevance-first search | Lucene |
| Managed Azure search, hybrid semantic relevance | Azure + `UseHybridSearch` |
| Chunk-level vector matching | Azure + `UseChunkedVectorSearch` |
| Custom fields calculated from Umbraco content/media | `IIndexingConverter` |
| Result model needs a small known field set | `[SearchResultFieldProjection]` on a partial `SearchResultModelBase` model |
| Ready-made browser endpoint | `UseSearch` / `UseSuggestionsSearch` |
| Search for an AI/MCP client | `AddUmbracoSearchMcpTools` or `WithUmbracoSearchTools` |

## Build a normal query

Inject `ISearchService` and use `SearchParameters`:

```csharp
var parameters = new SearchParameters
{
    FullTextSearch = new FullTextSearch("umbraco"),
    Skip = 0,
    Take = 20,
    Aliases = ["contentPage"],
    LanguageIsoCode = "en-US",
    SearchOrderings =
    [
        ISearchOrdering.ByScore(OrderingType.Descending),
        ISearchOrdering.ByField(IndexingConstants.ComputedIndexFields.SearchDate, OrderingType.Descending)
    ]
};

var results = await searchService.SearchAsync<CommonSearchItemModel>(parameters);
```

`SearchParameters` contract:

- `IndexName`: optional; fallback is `DefaultIndexName`.
- `FullTextSearch`: keywords, optional typed field targeting/boosts, wildcard option, and Azure hybrid controls.
- `Skip` / `Take`: paging. Public endpoints should impose their own maximum `Take`.
- `Aliases`: Umbraco content type aliases.
- `Filters`: typed conditions or `And`/`Or`/`Not` filter trees.
- `SearchOrderings`: relevance or typed field ordering.
- `FacetOn`: requested facets.
- `ExtraBoostingOptions`: value-specific score boosts.
- `Root`: limits results to a node/key subtree.
- `LanguageIsoCode`: requested culture. It is normalized against available Umbraco languages; default language is handled as the base field.
- `IncludeExcludedFromSearch`: opt into normally excluded/soft-deleted content only when that is intentional.
- `EngineSpecific`: last-mile native engine overrides. Prefer the portable model; an override makes engine parity the caller’s responsibility.

Use `CommonSearchItemModel` for a conventional result. A feature-specific result model inherits `SearchResultModelBase` and reads defined fields with `ValueFor<T>(fieldDefinition)`.

## Result field projections

The package provides two concrete common result bases. `CommonSearchItemModel` is the ordinary all-fields model; `CommonSearchResultProjection` exposes the same common properties but is the projected base for a projection hierarchy. Neither is abstract, and neither is required: consumers can define their own `SearchResultModelBase`-derived bases. Mark any custom base intended to participate in a projection hierarchy as `partial` with `[SearchResultFieldProjection]`.

Use a result field projection when a feature-specific result model consumes a small, known set of index fields. Mark the model as `partial` and apply `[SearchResultFieldProjection]`:

```csharp
[SearchResultFieldProjection]
public partial class ProductSearchResult : SearchResultModelBase
{
    public string? Name => ValueFor(ProductFields.Name);
    public int Stock => ValueFor(ProductFields.Stock);
    public string[] Tags => ValueFor(ProductFields.Tags) ?? [];
}
```

The source generator discovers fields used by `ValueFor(...)`, `ValueForAs(...)`, and marked helper accessors, then generates the model's static `RequiredIndexFields` property, fulfilling the static `ISearchResultFieldProjection` contract. For custom helper methods that retrieve a field, mark the helper with `[SearchResultFieldAccessor]` so the generator can include its field argument.
Do not manually implement `ISearchResultFieldProjection` or declare `RequiredIndexFields`; use the attribute on a partial model.
Projection is direct-only: an unmarked derived model does not inherit its base model's projection.
A projected model in a downstream assembly composes the `RequiredIndexFields` member from its nearest metadata base. Use the package's partial `[SearchResultFieldProjection]` `CommonSearchResultProjection` as that base; `CommonSearchItemModel` remains an ordinary unprojected model.
Every hierarchy level intended as a reusable projection base must itself be `partial` and directly marked with `[SearchResultFieldProjection]`. If marked `A` inherits unmarked, non-partial `B`, the generator instead composes from the next marked ancestor `C`; B's fields are included only while B is source-visible in A's compilation. Mark B when its fields must flow across project/package boundaries.

The projection is the model's field-retrieval contract. It lets search providers request only the fields this model uses, including localized and invariant field-name fallbacks. An unmarked model keeps the normal all-fields behavior.

`ValueFor` remains lazy: scalar and multi-value fields are materialized when the model accesses them. Prefer the declared field type (`IndexFieldDefinition<int>`, `IndexFieldDefinition<string[]>`, and so on) over `ValueForAs<T>`; reserve `ValueForAs<T>` for serialized or custom values. Ensure every projected field is registered by an indexing converter and stored by the selected backend's index schema.

## Filtering, sorting, facets, and boosts

### Filtering

The helper must match the index-field type:

```csharp
// Scalar field, e.g. IndexFieldDefinition<string> or IndexFieldDefinition<int>
ISearchFilter.FilterFor(ProductFields.Status, "published");

// Collection field, e.g. IndexFieldDefinition<List<string>> or string[]
ISearchFilter.FilterForCollectionField(ProductFields.Tags, "sale");

// Dynamic fallback when a field is only known at runtime
ISearchFilter.FilterFor((IIndexFieldDefinition)field, "value");
```

- `FilterFor(...)` on a collection field triggers analyzer `ILUS0002`.
- `FilterForCollectionField(...)` on a scalar field triggers analyzer `ILUS0003`.
- Compose different fields with `And`, `Or`, and `Not`; do not attempt to encode unrelated field logic in one leaf.
- Preserve the field’s underlying type. Treating numeric/date fields as tokenized strings produces backend-specific and fragile behavior.

### Sorting, boosting, facets

- Sort using `ISearchOrdering.ByScore(...)` or `ISearchOrdering.ByField(...)`. A sort field must have been declared sortable.
- Add `FacetOn` only for fields declared facetable. Facets are not a substitute for arbitrary group-by behavior.
- Use `ExtraBoostingOptions` for explicit relevance policy (field + value + boost). Confirm that the chosen behavior is meaningful on the selected engine.
- Suggestions use `SuggestionSettings` and follow the same index and language rules as normal search.

## Implement custom indexing

Implement `IIndexingConverter` in the consuming application. A concrete implementation must carry a supported DI registration attribute so it is discovered:

```csharp
[Service]
public sealed class ProductIndexingConverter : IIndexingConverter
{
    public IEnumerable<IIndexFieldDefinition> GetIndexFieldDefinitions() =>
        [ProductFields.Sku, ProductFields.Tags];

    public Task AddContentComputedFieldsAsync(
        IPublishedContent? content,
        IndexingModel item,
        IndexingItemContext context,
        CancellationToken cancellationToken = default)
    {
        item.SetOrOverrideComputedValue(ProductFields.Sku, content?.Value<string>("sku") ?? string.Empty);
        item.SetComputedValue(ProductFields.Tags, content?.Value<IEnumerable<string>>("tags")?.ToArray() ?? []);
        return Task.CompletedTask;
    }

    public Task AddMediaComputedFieldsAsync(
        IPublishedContent? media,
        IndexingModel item,
        IndexingItemContext context,
        CancellationToken cancellationToken = default)
    {
        // Add media fields only when this feature needs them.
        return Task.CompletedTask;
    }
}
```

Define public fields with `IndexFieldDefinitionFactory`, not raw field-name strings:

```csharp
public static class ProductFields
{
    public static readonly IndexFieldDefinition<string> Sku =
        IndexFieldDefinitionFactory.ForString("productSku", isRaw: true, isSortable: true);

    public static readonly IndexFieldDefinition<string[]> Tags =
        IndexFieldDefinitionFactory.ForStringArray("productTags", isFacetable: true, isRaw: true);
}
```

Implementation rules:

- `GetIndexFieldDefinitions()` declares the schema contract. Register every field that queries/results/facets use.
- `RunForIndexes()` may limit a converter to particular index names. An empty list means all relevant indexes.
- `Order` controls converter layering. Use it only when an override order is intentional and documented in the application.
- `SetComputedValue` preserves an existing value; `SetOrOverrideComputedValue` intentionally wins. Choose based on feature ownership.
- Keep field names, generic types, collection/scalar shape, raw/tokenized mode, sorting, faceting, and language variance stable. Any change requires a compatible migration/reindex plan and query review.
- Analyzer `ILUS0001` reports concrete converters without the necessary DI registration attribute.

## Language-aware data

- Mark a field multi-language when values truly vary per culture. The package expands it into culture-specific fields during indexing.
- Query with `SearchParameters.LanguageIsoCode`; do not manually append culture strings to field names.
- In advanced field-aware code, use the field definition’s `LanguageInvariantFieldName(...)` helper rather than constructing names manually.
- Test the default culture and at least one non-default culture for every multi-language search feature: population, full text, filters, facets, and result projection as applicable.

## Taxonomy

Use taxonomy when a value has meaningful hierarchy, not for generic flat labels:

```csharp
item.AddTaxonomyValue("Category", "SubCategory", "Tag");
```

This stores the complete path in `TaxonomyTags` (`Category__SubCategory__Tag`) and category paths in `TaxonomyCategories` (`Category__SubCategory`).

- Facet on `TaxonomyTags` to obtain tag values.
- Filter `TaxonomyCategories` to drill down through category levels.
- With both `EnableTaxonomyFacetExpansion` and `EnableTaxonomyFiltersExpansion` enabled, render the hierarchical `Facet.Values` / nested `FacetOption.Values` structure and use `ChildFacetsIndexFieldName` for the next level.
- Under those same settings, an `OR` filter spanning different taxonomy branches is regrouped by category path. Do not duplicate that normalization in UI/controller code.
- Custom taxonomy fields are supported when the application needs separate taxonomy domains.

## Azure, hybrid, and chunk vectors

Use Azure only when its operational requirements are available: service credentials, a managed-index allow-list, lifecycle ownership, and monitoring.

### Azure lifecycle

- The package may provision/rebuild Azure equivalents only for names in `SearchSettings.Indexes`.
- `CustomIndexSuffix` and optional blue/green naming are managed internally. Do not derive or swap physical index names in application search code.
- When `EnableBlueGreenIndexing` is on, rebuilds may create the alternate index, switch active search/indexing clients, and delete the inactive index after a successful swap. Build a recovery/observability plan before enabling it.
- `ReadOnly` application nodes must not execute write/rebuild flows.
- For preview indexes, keep exclusion semantics correct for both base and language-specific fields.

### Hybrid vectors

Enable only after conventional search works:

1. Register `SearchOptions.Azure`.
2. Configure Azure credentials, `OpenAi` embedding settings, and `Azure.UseHybridSearch=true`.
3. Request hybrid search with `new FullTextSearch(query, useHybridSearch: true)` where appropriate.
4. In an indexing converter, set source text with `item.SetVectorContent(content)`.

For chunk-level vectors, set `Azure.UseChunkedVectorSearch=true` and optionally provide a different source/chunking policy:

```csharp
item.SetVectorContent("short searchable summary");
item.SetVectorContentPrecise("long form content for chunk matching");
// Or choose per-item bounds:
item.SetVectorContentPreciseManual(content, minTokens: 1000, maxTokens: 2000, overlapTokens: 150);
```

Vector behavior:

- Average vectors are the baseline. Precise chunk content falls back to the average content, then normal search content, when omitted.
- Azure multi-vector quota limits chunks (effectively 100); the package may increase chunk size and log a warning.
- `BackgroundEmbeddingsProcessing=true` queues average and chunk generation durably. Results are eventually consistent: an immediately published document may not yet participate in vector matching.
- If changing vector persistence from the consuming application, use the package’s supported persistence configuration and apply EF Core migrations through the application’s normal deployment process. Do not delete embedding data to “fix” a relevance issue.

## Optional HTTP endpoints

The package can map minimal APIs:

```csharp
app.UseSearch();
app.UseSuggestionsSearch();
```

Default paths are `/api/search` and `/api/search/suggest`. Options can change the path, constrain/map request parameters, or customize output:

```csharp
app.UseSearch(options =>
{
    options.Path = "/api/site-search";
    options.SearchParametersOverride = static (_, _, parameters, _) =>
    {
        parameters.Aliases = ["contentPage"];
        parameters.Take = Math.Min(parameters.Take, 50);
        return ValueTask.FromResult(parameters);
    };
});
```

Unknown fields in the built-in request mapper are ignored. That is not validation: validate and cap untrusted public inputs in the consuming application. Add authentication/authorization, rate limiting, caching, and tenant restrictions there as well.

## Optional MCP tools

Expose search to an MCP client using either:

```csharp
builder.Services.AddUmbracoSearchMcpTools().WithHttpTransport();
// or add package tools to an existing MCP builder:
builder.Services.AddMcpServer().WithHttpTransport().WithUmbracoSearchTools();
app.MapMcp("/searchMcp");
```

The package tools are `fetchAvailableIndexes` and `search`. Default MCP `search` accepts `(indexName, q, top_k)`, clamps `top_k` to `1..50`, and returns common search-item data. Configure `McpSearchOptions` to constrain the default query or map result data; replace/decorate `IUmbracoSearchMcpService` for an intentionally different contract.

MCP transport, authentication, authorization, client identity, rate limits, and exposure of index names remain application responsibilities.

## Troubleshooting sequence

1. Confirm the enabled feature matches the intended engine.
2. Check application startup logs and license configuration.
3. Confirm `DefaultIndexName`, request `IndexName`, and `Indexes` all match existing Umbraco indexes exactly.
4. Check the requested content is actually published/indexed and that a custom converter is DI-discovered.
5. Validate field declaration versus usage: scalar/collection, correct generic type, facetable/sortable/raw/multi-language settings.
6. Check culture resolution and indexed values for the requested language.
7. For empty taxonomy results, check whether filtering tags versus category paths and whether expansion flags match UI expectations.
8. For Azure, check `ReadOnly`, managed-index allow-list, service credentials, server role, suffix/blue-green configuration, and batch logs.
9. For hybrid results, check both `UseHybridSearch` and OpenAI settings, then distinguish missing source text from asynchronous background embedding delay.
10. For an endpoint/MCP issue, inspect application-owned mapping overrides, authorization, limits, and request validation before assuming a package issue.

## Definition of done for a consumer feature

- The chosen engine supports every advertised feature.
- The app registers the correct feature and has valid, secret-managed configuration.
- Custom fields have typed definitions, a registered converter, correct population, and queries that match their shape.
- Default index and explicit index paths are tested.
- Language-aware behavior is tested for default and non-default culture where used.
- Filter, facet, sort, and result-model behavior are exercised against representative content.
- Azure lifecycle constraints and vector consistency expectations are represented in operations/tests when Azure is enabled.
- Public HTTP/MCP surfaces impose app-owned authorization, rate and pagination limits, and input constraints.
- Consumer-facing examples/configuration are kept with the application and aligned to the installed package version.
