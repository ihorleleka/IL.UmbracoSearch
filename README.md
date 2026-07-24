[![NuGet version (IL.AttributeBasedDI)](https://img.shields.io/nuget/v/IL.UmbracoSearch.svg?style=flat-square)](https://www.nuget.org/packages/IL.UmbracoSearch/)
# IL.UmbracoSearch

A comprehensive search solution for Umbraco, supporting both Lucene and Azure Search, with extensible indexing and flexible search parameters.

## Table of Contents

- [IL.UmbracoSearch](#ilumbracosearch)
  - [Table of Contents](#table-of-contents)
  - [Features](#features)
  - [Quick Start](#quick-start)
  - [Configuration](#configuration)
    - [1. Service Registration](#1-service-registration)
    - [2. AppSettings](#2-appsettings)
    - [Configuration Details](#configuration-details)
  - [Basic Usage](#basic-usage)
    - [The `SearchParameters` Object](#the-searchparameters-object)
  - [Minimal API Endpoints](#minimal-api-endpoints)
  - [MCP Tools](#mcp-tools)
  - [Advanced Usage](#advanced-usage)
    - [Advanced SearchApiController Example](#advanced-searchapicontroller-example)
    - [Multi-Language Support](#multi-language-support)
  - [Customizing the Index](#customizing-the-index)
    - [The `IIndexingConverter` Interface](#the-iindexingconverter-interface)
    - [Defining Index Fields](#defining-index-fields)
    - [Auto-Discovery with DI Registration Attributes](#auto-discovery-with-di-registration-attributes)
    - [Example: Custom Indexing Converter](#example-custom-indexing-converter)
  - [Customizing Search Results](#customizing-search-results)
    - [`CommonSearchItemModel`](#commonsearchitemmodel)
    - [Custom Models with `ValueFor<T>`](#custom-models-with-valuefort)
  - [Built-in Indexing Constants](#built-in-indexing-constants)

## Features

- **Full-text search:** Search for keywords in the content of the documents.
- **Filtering:** Filter search results by various criteria, such as document type, date range, and tags.
- **Sorting:** Sort search results by relevance, date, or any other field.
- **Faceting:** Get a count of the number of documents that match each value of a field.
- **Hybrid search (Azure only):** Combine keyword search with vector search for more relevant results.
- **Granular hybrid search (Azure only):** Optionally store chunk-level vectors per item (multi-vector field) for finer vector matching.
- **Extensible:** Add your own custom fields to the search index.
- **Optional Search Insights:** On Umbraco 17, install `IL.UmbracoSearch.Analytics` and add `SearchOptions.Analytics` alongside exactly one engine to capture search and consented click metrics. See the analytics package README for SQL setup, the backoffice Search section, Azure-only synonyms, and the npm click helper.

## Quick Start

Here's the simplest way to get started.

1.  **Install the package** and configure your services and `appsettings.json` (see [Configuration](#configuration)).
2.  **Inject `ISearchService`** into your controller or service.
3.  **Perform a search:**

```csharp
// Inject the search service
private readonly ISearchService _searchService;

public MyController(ISearchService searchService)
{
    _searchService = searchService;
}

// Perform a basic search
public async Task<IActionResult> Search(string query)
{
    var searchParameters = new SearchParameters
    {
        FullTextSearch = new FullTextSearch(query),
        Take = 20
    };

    var results = await _searchService.SearchAsync<CommonSearchItemModel>(searchParameters);

    // 'results' now contains the top 20 hits for the query.
    // You can access properties like results.Items, results.Total, etc.

    return View(results);
}
```

## Configuration

> [!NOTE]
> This library attaches to existing Umbraco indexes (e.g., `ExternalIndex`, `InternalIndex`). It does not create new indexes.

### 1. Service Registration

In your `Program.cs`, register the services by calling `AddServiceAttributeBasedDependencyInjection`.

```csharp
// Program.cs
builder.AddServiceAttributeBasedDependencyInjection(options =>
{
    options.AddFeature(SearchOptions.Lucene);
    // or options.AddFeature(SearchOptions.Azure);
});
```

### 2. AppSettings

Add a `SearchSettings` section to your `appsettings.json`.

<details>
<summary><strong>Example `appsettings.json`</strong></summary>

```json
"SearchSettings": {
  "LicenseToken": "YOUR_LICENSE_TOKEN",
  "DefaultIndexName": "ExternalIndex",
  "Indexes": [
    "ExternalIndex",
    "InternalIndex"
  ],
  "ReadOnly": false,
  "PreviewIndexes": ["InternalIndex"],
  "CustomIndexSuffix": "Dev|Staging|DeveloperMachine|None",
  "EnableBlueGreenIndexing": false,
  "DisableSwapDelay": false,
  "EnableTaxonomyFacetExpansion": false,
  "EnableTaxonomyFiltersExpansion": false,
  "Azure": {
    "ServiceUrl": "YOUR_AZURE_SEARCH_SERVICE_URL",
    "ApiKey": "YOUR_AZURE_SEARCH_API_KEY",
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
    "ApiKey": "YOUR_OPENAI_API_KEY",
    "ServiceUrl": "YOUR_OPENAI_SERVICE_URL",
    "EmbeddingsDeploymentName": "text-embedding-3-large"
  }
}
```

</details>

### Configuration Details

- **LicenseToken:** Required when using the Azure feature; Lucene does not require a license token.
- **DefaultIndexName:** The default index to use if not specified in a search.
- **CustomIndexSuffix:** Allows to specify custom suffix applied to all of your configured indexes. Optional.
- **Indexes**: A list of search index names to be used. Defaults to `["ExternalIndex"]`.
- **PreviewIndexes**: A list of search index names where soft deletion is enabled.
- **Azure:** Azure Search service credentials.
- **OpenAi:** OpenAI credentials for vector embeddings.
- **Azure.UseHybridSearch:** Enables vector + keyword hybrid search.
- **Azure.UseChunkedVectorSearch:** Enables chunk-level multi-vector indexing in Azure in addition to the averaged vector.
- **Azure.ChunkedVectorMinTokens / MaxTokens / OverlapTokens:** Controls chunk granularity used for chunk-level vectors.
- **Azure.ChunkedVectorEmbeddingMaxConcurrency:** Max parallel OpenAI embedding requests per indexed item when chunk vectors are generated.
- **Azure.BackgroundEmbeddingsProcessing:** Enables durable background processing for both average and chunk vectors. Azure document writes are batched and become eventually consistent. Defaults to `false`; when enabled, both vector fields may be unavailable briefly after indexing.
- **Azure.DocumentBatchMaxActions / DocumentBatchMaxPayloadBytes / DocumentBatchFlushIntervalMilliseconds:** Bounds each Azure merge-or-upload batch by action count, serialized payload size, and maximum wait time. Defaults are `100`, `14680064` (14 MiB), and `250` ms.
- **Azure.DocumentBatchQueueCapacity:** Maximum accepted-but-not-yet-sent document writes. When full, indexing waits for capacity instead of creating unbounded in-memory work. Defaults to `2000`.
- **Azure.DocumentBatchMaxRetries:** Retry limit for transient Azure indexing failures (`408`, `429`, and `5xx`). Defaults to `3`; permanent action failures are logged and reported by rebuild flush.
- **Azure.BackgroundEmbeddingsMaxConcurrency:** Maximum number of independent durable vector jobs processed concurrently. Defaults to `4`.
- **ReadOnly:** Disallows runtime to create or modify existing index in any way.
- **EnableBlueGreenIndexing:** Allows to enable b/g indexing behavior when rebuilding index, so that old functional index temporary remains available for search operations.
- **DisableSwapDelay:** System will delay index swap to allow for indexing to be finalized (1 min per 1000 items of delay); you can disable this behavior with this flag (probably for dev/test purpose).
- **EnableTaxonomyFacetExpansion:** Enables hierarchical taxonomy facet expansion for `TaxonomyTags` values. Keep disabled if you want to work with raw facet values only.
- **EnableTaxonomyFiltersExpansion:** When taxonomy facet expansion is enabled, rewrites taxonomy `OR` filters so values from different category paths are split into separate filter groups.

## Basic Usage

To perform a search, inject `ISearchService` and call `SearchAsync` with a `SearchParameters` object.

```csharp
var searchParameters = new SearchParameters
{
    FullTextSearch = new FullTextSearch("umbraco", useHybridSearch: true),
    Skip = 0,
    Take = 10,
    Aliases = new[] { "contentPage" },
    SearchOrderings = new List<ISearchOrdering>
    {
        ISearchOrdering.ByScore(OrderingType.Descending),
        ISearchOrdering.ByField(IndexingConstants.ComputedIndexFields.SearchDate, OrderingType.Descending)
    },
    Filters =
    [
        ISearchFilter.FilterForCollectionField(
            IndexingConstants.ComputedIndexFields.NodeTypeAlias,
            "contentPage"),
        ISearchFilter.FilterFor(
            IndexingConstants.ComputedIndexFields.UmbracoNodeIdInt,
            1234),
        ISearchFilter.FilterForCollectionField(
            IndexingConstants.ComputedIndexFields.TaxonomyTags,
            "Category__SubCategory__TagName")
    ],
    FacetOn = new List<FacetOn>
    {
        new FacetOn(IndexingConstants.ComputedIndexFields.TaxonomyTags)
    },
    LanguageIsoCode = "en-US"
};

var searchResults = await searchService.SearchAsync<CommonSearchItemModel>(searchParameters);
```

### Taxonomy Indexing and Faceting

The taxonomy helpers let you index a full taxonomy path once and then facet/filter on category levels separately from final tag values.

#### Indexing taxonomy values

Use `AddTaxonomyValue` during indexing. Each call accepts path segments and stores:
- full value in `TaxonomyTags` as `"__"`-joined path (for example `Category__SubCategory__Tag`)
- category-only path in `TaxonomyCategories` (for example `Category__SubCategory`)

```csharp
indexingObject
    .AddTaxonomyValue("Category", "SubCategory", "TagName")
    .AddTaxonomyValue("Category", "SubCategory", "TagName2");
```

You can also target custom taxonomy fields without changing existing calls:

```csharp
indexingObject.AddTaxonomyValue(
    taxonomyTagsField: CustomIndexingConstants.ProductTags,
    taxonomyCategoriesField: CustomIndexingConstants.ProductTagGroups,
    "Category", "SubCategory", "TagName");
```

#### Enabling hierarchical facet expansion

Set the following appsetting to `true`:

```json
"SearchSettings": {
  "EnableTaxonomyFacetExpansion": true,
  "EnableTaxonomyFiltersExpansion": true
}
```

When enabled, the shared search pipeline expands taxonomy facets into a hierarchy:
- the root `Facet` is always bound to `TaxonomyCategories`
- `Facet.ChildFacetsIndexFieldName` tells you which field to use for the next level
- intermediate category nodes use `TaxonomyCategories`
- terminal tag nodes use `TaxonomyTags`

When `EnableTaxonomyFiltersExpansion` is also enabled, taxonomy filters with `FilteringBehavior.Or` are normalized by category path before execution. For example:

```csharp
ISearchFilter.FilterFor(
    IndexingConstants.ComputedIndexFields.TaxonomyTags,
    FilteringBehavior.Or,
    "Category1__Tag1",
    "Category2__Tag2",
    "Category2__Tag3")
```

is internally rewritten into two `OR` filters:
- `["Category1__Tag1"]`
- `["Category2__Tag2", "Category2__Tag3"]`

This keeps same-category values grouped together while avoiding a single broad `OR` across unrelated taxonomy branches.

The top-level facet now exposes `Values` instead of a flat `Values` collection, and nested `FacetOption.Values` can be used to render sub-categories and tags.

Example hierarchy for `Category__SubCategory__TagName` and `Category__SubCategory__TagName2`:

```json
[
  {
    "Label": "Category",
    "IndexFieldName": "computedTaxonomyCategories",
    "Value": "Category",
    "HitsCount": 2,
    "ChildFacetsIndexFieldName": "computedTaxonomyCategories",
    "Values": [
      {
        "Label": "SubCategory",
        "Value": "Category__SubCategory",
        "HitsCount": 2,
        "ChildFacetsIndexFieldName": "computedTaxonomyTags",
        "Values": [
          {
            "Label": "TagName",
            "Value": "Category__SubCategory__TagName",
            "HitsCount": 1
          },
          {
            "Label": "TagName2",
            "Value": "Category__SubCategory__TagName2",
            "HitsCount": 1
          }
        ]
      }
    ]
  }
]
```

```csharp
var searchParameters = new SearchParameters
{
    FacetOn = [new FacetOn(IndexingConstants.ComputedIndexFields.TaxonomyTags)],
    Filters =
    [
        // Drill down by category path
        ISearchFilter.FilterForCollectionField(IndexingConstants.ComputedIndexFields.TaxonomyCategories,
            "Category__SubCategory")
    ]
};
```

### The `SearchParameters` Object

This object allows you to define your query:

- **FullTextSearch:** The full-text search query.
- **Skip/Take:** For pagination.
- **Aliases:** Filter by document type aliases.
- **SearchOrderings:** Define how to sort results.
- **Filters:** Apply filters to the search.
- **FacetOn:** Request facet counts for specific fields.
- **ExtraBoostingOptions:** Boost the score of certain documents.
- **IndexName:** Specify which index to search.
- **Root:** Restrict the search to a subtree by node ID or node key.
- **LanguageIsoCode:** The culture to search in.
- **EngineSpecific:** Optional escape hatch for engine-specific request customization right before execution.

Example:

```csharp
var searchParameters = new SearchParameters
{
    FullTextSearch = new FullTextSearch("umbraco"),
    Root = Guid.Parse("11111111-1111-1111-1111-111111111111"),
    EngineSpecific =
    {
        AzureRequestOverride = options =>
        {
            // Modify Azure SearchOptions right before SearchAsync executes
        },
        LuceneRequestOverride = query =>
        {
            // Modify the underlying LuceneSearchQuery right before Execute runs
        }
    }
};
```

Both overrides are nullable and are invoked as `action?.Invoke(...)`.

## Minimal API Endpoints

The package includes opt-in minimal API registrations for teams that want a ready-made HTTP surface on top of `ISearchService`.

```csharp
app.UseSearch();
app.UseSuggestionsSearch();
```

Default paths:

- search: `/api/search`
- suggestions: `/api/search/suggest`

Both endpoints can be customized at registration time:

```csharp
app.UseSearch(options =>
{
    options.Path = "/custom/search";
    options.SearchParametersOverride = static (httpContext, request, parameters, cancellationToken) =>
    {
        parameters.Aliases = ["contentPage"];
        return ValueTask.FromResult(parameters);
    };
});

app.UseSuggestionsSearch(options =>
{
    options.Path = "/custom/search/suggest";
    options.SuggestionSettingsOverride = static (httpContext, request, parameters, cancellationToken) =>
    {
        parameters.Size = Math.Min(parameters.Size, 5);
        return ValueTask.FromResult(parameters);
    };
});
```

If you want the same endpoint behavior with a different search result model, use the generic overloads:

```csharp
app.UseSearch<MySearchItemModel>();
app.UseSuggestionsSearch<MySearchItemModel>();
```

The default request contracts are `SearchRequest` and `SuggestionsSearchRequest`. They are mapped onto `SearchParameters` and `SuggestionSettings`, and any unknown field names are ignored by default.

## MCP Tools

The package includes opt-in Model Context Protocol tool registration for exposing search to MCP clients.

Built-in tools:

- `fetchAvailableIndexes`: returns `DefaultIndexName`, `Indexes`, and `PreviewIndexes` from `SearchSettings`.
- `search`: runs a default content search with `(string? indexName, string? q, int top_k = 5)`.

Register the tools directly from `builder.Services`:

```csharp
using IL.UmbracoSearch.Search.Mcp;

builder.Services
    .AddUmbracoSearchMcpTools()
    .WithHttpTransport();

app.MapMcp("/searchMcp");
```

Or add the tools to an existing MCP server builder:

```csharp
builder.Services
    .AddMcpServer()
    .WithHttpTransport()
    .WithUmbracoSearchTools();

app.MapMcp("/searchMcp");
```

`AddUmbracoSearchMcpTools()` and `WithUmbracoSearchTools()` register the IL.UmbracoSearch MCP service and add the package tools. The package does not force a transport; configure the MCP transport in the consuming app. For an HTTP MCP endpoint, add the MCP ASP.NET Core package in the web app and configure transport/routing there.

The default `search` tool maps `q` to `FullTextSearch`, `indexName` to `SearchParameters.IndexName`, and clamps `top_k` to `1..50`. When `SearchSettings.Azure.UseHybridSearch` is enabled, MCP search also enables hybrid full-text search by default. It returns common result fields: `id`, `title`, `description`, `url`, `languageIsoCode`, plus `additionalData`.

Customize the default query or map each returned item:

```csharp
builder.Services.AddUmbracoSearchMcpTools(options =>
{
    options.SearchParametersOverride = static (request, parameters, cancellationToken) =>
    {
        parameters.Aliases = ["contentPage"];
        return ValueTask.FromResult(parameters);
    };

    options.ResultMapper = static (context, cancellationToken) =>
        ValueTask.FromResult(new McpSearchItem
        {
            Id = context.Item.Id,
            Title = context.Item.SearchTitle ?? context.Item.NodeName,
            Description = context.Item.SearchDescription,
            Url = context.Item.Url,
            AdditionalData = new Dictionary<string, object?>
            {
                ["nodeName"] = context.Item.NodeName,
                ["requestedIndex"] = context.Request.IndexName
            }
        });
}).WithHttpTransport();
```

Use a custom strongly typed search result model when you want the mapper to work with your own result properties or `ValueFor(...)` calls:

```csharp
builder.Services
    .AddUmbracoSearchMcpTools<ProductSearchItem>(options =>
    {
        options.ResultMapper = static (context, cancellationToken) =>
            ValueTask.FromResult(new McpSearchItem
            {
                Id = context.Item.ProductId,
                Title = context.Item.ValueFor(CustomIndexingConstants.ProductName),
                AdditionalData = new Dictionary<string, object?>
                {
                    ["sku"] = context.Item.ValueFor(CustomIndexingConstants.ProductSku)
                }
            });
    })
    .WithHttpTransport();
```

For a fully custom MCP search implementation, replace/decorate `IUmbracoSearchMcpService` in DI. The MCP tool layer calls that service for both `fetchAvailableIndexes` and `search`.

Rate limiting should be configured in the consuming app, CDN, or hosting layer because that layer owns the MCP transport, endpoint routing, authentication, and client identity. For an ASP.NET Core fixed-window limit on the MCP endpoint:

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("umbraco-search-mcp", limiter =>
    {
        limiter.PermitLimit = 60;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueLimit = 0;
    });
});

builder.Services
    .AddUmbracoSearchMcpTools()
    .WithHttpTransport();

app.UseRateLimiter();

app.MapMcp("/searchMcp")
    .RequireRateLimiting("umbraco-search-mcp");
```

### Filter API

Use the filter helper that matches the field shape:

- `ISearchFilter.FilterFor(...)` for scalar `IndexFieldDefinition<T>` fields.
- `ISearchFilter.FilterForCollectionField(...)` for collection fields such as `IndexFieldDefinition<string[]>`, `IndexFieldDefinition<int[]>`, `IndexFieldDefinition<List<string>>`, etc.
- `ISearchFilter.FilterFor(IIndexFieldDefinition, ...)` as a dynamic fallback when the field is only known at runtime, for example after looking it up by name.
- `ISearchFilter.And(...)`, `ISearchFilter.Or(...)`, and `ISearchFilter.Not(...)` when you need to combine single-field filters into a more advanced expression.

Examples:

```csharp
Filters =
[
    ISearchFilter.FilterFor(CustomIndexingConstants.ProductName, "My Product"),
    ISearchFilter.FilterFor(IndexingConstants.ComputedIndexFields.UmbracoNodeIdInt, 123),
    ISearchFilter.FilterForCollectionField(CustomIndexingConstants.ProductCategories, "Category1", "Category2")
];
```

Each `FilterFor(...)` or `RangeFilterFor(...)` leaf targets a single field. When you need cross-field logic, compose leaves explicitly instead of putting multiple fields into one filter body:

```csharp
var filter = ISearchFilter.And(
    ISearchFilter.FilterFor(CustomIndexingConstants.ProductName, "My Product"),
    ISearchFilter.Not(ISearchFilter.FilterFor(IndexingConstants.ComputedIndexFields.ItemType, "archived")));
```

When you have only `IIndexFieldDefinition` available, use the non-generic fallback:

```csharp
IIndexFieldDefinition field = GetFieldByName("custom_productCategories");
var filter = ISearchFilter.FilterFor(field, "Category1", "Category2");
```

The package includes analyzers that fail the build when the wrong strongly-typed helper is used:

- `ILUS0002`: `FilterFor(...)` used with a collection field
- `ILUS0003`: `FilterForCollectionField(...)` used with a scalar field

`Root` accepts all of these forms:

```csharp
Root = 123;
Root = "123";
Root = Guid.Parse("11111111-1111-1111-1111-111111111111");
```

## Advanced Usage

### Advanced `SearchApiController` Example

For a complete example of how to build a search query from URL parameters, see the controller below. It handles parsing filters, ordering, facets, and more from the query string.

An example URL for this controller would be:
`/Search/Search?q=umbraco&filters=__NodeTypeAlias:contentPage&orderBy=score:desc`

<details>
<summary><strong>`SearchApiController` Code</strong></summary>

```csharp
public class SearchApiController(ISearchService searchService, IIndexService indexService) : Controller
{
    [HttpGet]
    public async Task<IActionResult> Search([FromQuery] string q = "",
        [FromQuery] bool useHybridSearch = false,
        [FromQuery] string? filters = null,
        [FromQuery] string? orderBy = null,
        [FromQuery] string? facetOn = null,
        [FromQuery] string? boostById = null,
        [FromQuery] int skip = 0,
        [FromQuery] int take = int.MaxValue,
        [FromQuery] int? root = null,
        [FromQuery] string? indexName = null,
        [FromQuery] string? lang = null
    )
        =>
                        new JsonResult(await searchService.SearchAsync<CommonSearchItemModel>(
                    new SearchParameters
                    {
                        FullTextSearch = new FullTextSearch(q, useHybridSearch: useHybridSearch)
                        {
                            VectorSimilarityThreshold = 0.3f,
                        },
                        Aliases = [ContentPage.ModelTypeAlias],
                        Filters = TryBuildFilters(filters, indexName),
                        FacetOn = TryBuildFacetOn(facetOn, indexName),
                        SearchOrderings = TryBuildSearchOrderings(orderBy, indexName),
                        ExtraBoostingOptions = TryBuildExtraBoostingOptions(boostById),
                        Skip = skip,
                        Take = take,
                        Root = root,
                        IndexName = indexName,
                        LanguageIsoCode = lang
                    }));

    // Helper methods for parsing query string parameters...
}
```

</details>

### Multi-Language Support

The package supports multi-language indexing and search. The system automatically creates language-specific fields (e.g., `productName_en`, `productName_de`).

Multi-language fields can be defined via factory methods using `isMultiLanguage: true` (for example `IndexFieldDefinitionFactory.ForString(..., isMultiLanguage: true)`).

Set the `LanguageIsoCode` property in `SearchParameters` to search in a specific language.

The library now normalizes and resolves language values consistently across indexing and search:
- `en`, `en-GB`, and `en_GB` are all accepted
- field suffixes are generated in a consistent `_en_gb` format
- if the exact language is not configured but a partial match exists, the closest configured language is used
- if the input cannot be matched at all, the system falls back to the default configured language during search

### Hybrid And Granular Vectorization Notes

- The existing averaged vector behavior is preserved for large sources (source is chunked and embeddings are averaged).
- Chunk-level vectors are generated only when `UseChunkedVectorSearch` is enabled.
- Small documents (below token threshold used for single-pass embedding) are short-circuited to average-only vector storage; chunk vectors are skipped to reduce indexing/storage volume.
- Reindexing unchanged content does not regenerate chunk vectors: cache invalidation uses a hash of the full source content used for vectorization.
- Chunk planning metadata (tokenization/chunk split result) is cached by source hash and chunk settings to avoid repeated re-tokenization work across reindex operations.
- Azure multi-vector fields currently allow up to 100 vectors per document (across complex collection vector fields). If configured chunk granularity would exceed this quota, the system automatically increases effective chunk size and logs a warning.
- When `BackgroundEmbeddingsProcessing` is enabled, indexing events enqueue bounded Azure document writes instead of waiting for Azure. Rebuild completion drains those document batches, but average and chunk vector enrichment remains eventually consistent.

### Granular Chunk Size Recommendations

Choose chunking based on retrieval quality needs and expected content size:

- **Balanced default (recommended for most sites):** `Min=2000`, `Max=3000`, `Overlap=200`
- **Higher precision semantic matching (short sections, FAQs, product specs):** `Min=600`, `Max=900`, `Overlap=100`
- **Very granular matching (only when you need narrow passage recall):** `Min=400`, `Max=500`, `Overlap=50`
- **Long-form content efficiency (large articles/docs, lower indexing cost):** `Min=2500`, `Max=4000`, `Overlap=200`

Notes:
- More granular chunks increase vector count and indexing/storage cost.
- If your chosen values would create more than 100 chunk vectors for an item, the system increases chunk size automatically to stay within quota and logs:
  `system had to increase chunk size because otherwise vectors count exceeds quota (100)`.

## Customizing the Index

You can add your own custom fields to the search index by implementing the `IIndexingConverter` interface.

> [!IMPORTANT]
> `IndexFieldDefinition` constructors are no longer part of the public API.
> Always create field definitions through `IndexFieldDefinitionFactory` (`ForString`, `ForStringArray`, `ForInt`, `ForBool`, `Custom` or `AutoDetection`).

### The `IIndexingConverter` Interface

A class implementing this interface allows you to:

- **`GetIndexFieldDefinitions()`**: Return a collection of `IIndexFieldDefinition` objects that define your custom fields.
- **`AddContentComputedFields(..., IndexingItemContext context)`**: Add computed values to your custom fields for each document being indexed.
- **`AddMediaComputedFields(..., IndexingItemContext context)`**: Do the same for media items.
- **`RunForIndexes()`**: Specify which indexes the converter should run for.

### Defining Index Fields

Create field definitions with `IndexFieldDefinitionFactory` and group them in a static constants class.

```csharp
public static class CustomIndexingConstants
{
    public const string CustomFieldPrefix = "custom_";

    // Full-text searchable + sortable string field
    public static readonly IndexFieldDefinition<string> ProductName =
        IndexFieldDefinitionFactory.ForString(
            fieldName: $"{CustomFieldPrefix}productName",
            isSearchable: true,
            isFacetable: false,
            isSortable: true,
            isFilterable: true);

    // Multi-value string field for filters/facets
    public static readonly IndexFieldDefinition<string[]> ProductCategories =
        IndexFieldDefinitionFactory.ForStringArray(
            fieldName: $"{CustomFieldPrefix}productCategories",
            isFacetable: true,
            isFilterable: true);

    // Vector field stored only in Azure (Lucene definition intentionally null)
    public static readonly IndexFieldDefinition<float[]> ProductVector =
        IndexFieldDefinitionFactory.Custom<float[]>(
            luceneFieldDefinition: null,
            azureFieldDefinition: new VectorSearchField(
                name: $"{CustomFieldPrefix}productVector",
                vectorSearchDimensions: 1536,
                vectorSearchProfileName: "vector-profile"));
}
```

### Auto-Discovery with DI Registration Attributes

Decorate your `IIndexingConverter` implementation with `[Service]`, `[ServiceWithOptions]`, or `[Decorator]` so it can be registered by the dependency injection container.

The package now includes analyzer `ILUS0001`, which fails the build for concrete `IIndexingConverter` implementations that do not declare one of these IL.AttributeBasedDI registration attributes.

### Example: Custom Indexing Converter

```csharp

[Service<SearchOptions>(Lifetime = ServiceLifetime.Singleton, Feature = SearchOptions.Lucene | SearchOptions.Azure)]
public class CustomIndexingConverter : IIndexingConverter
{
    public int Order => 100; // Higher order runs after core converters

    public IEnumerable<IIndexFieldDefinition> GetIndexFieldDefinitions()
    {
        return new IIndexFieldDefinition[]
        {
            CustomIndexingConstants.ProductName,
            CustomIndexingConstants.ProductCategories
        };
    }

    public void AddContentComputedFields(IPublishedContent? publishedContent, IndexingModel indexingObject, IndexingItemContext context)
    {
        if (publishedContent != null)
        {
            indexingObject.SetComputedValue(CustomIndexingConstants.ProductName, "My Product");
            indexingObject.SetComputedValue(CustomIndexingConstants.ProductCategories, new[] { "Category1", "Category2" });
            indexingObject.SetVectorContent("Long text used for vectorization...");
            indexingObject.SetVectorContentPrecise("Long text used for vectorization (settings-driven chunking)...");
            indexingObject.SetVectorContentPreciseManual(
                "Long text used for vectorization (manual chunking)...",
                minTokens: 600,
                maxTokens: 900,
                overlapTokens: 100);
        }
    }
}
```

Use dedicated vector helpers:
- `SetVectorContent(...)`: average-vector content.
- `SetVectorContentPrecise(...)`: chunk-vector content using `SearchSettings.Azure` chunk settings.
- `SetVectorContentPreciseManual(...)`: chunk-vector content with per-item manual chunk sizing.

Behavior notes:
- If granular hybrid search is enabled and `SetVectorContentPrecise(...)` is not called, granular chunk vectors automatically reuse the `SetVectorContent(...)` content with settings-based chunk sizes.
- `SetVectorContent(...)` and `SetVectorContentPrecise(...)` can use different content values for average-vector and chunk-vector flows.

## Customizing Search Results

### `CommonSearchItemModel`

The default `CommonSearchItemModel` provides access to common fields like `Id`, `NodeName`, `SearchTitle`, `SearchDescription`, `Url`, etc.

### Custom Models with `ValueFor<T>`

For custom data, create your own search result model by inheriting from `SearchResultModelBase`. Use the `ValueFor<T>(IndexFieldDefinition<T> fieldDefinition)` method to retrieve values from the index.

```csharp
public class ProductSearchResultModel : SearchResultModelBase
{
    public string ProductName => ValueFor(CustomIndexingConstants.ProductName) ?? string.Empty;
    public string[] Categories => ValueFor(CustomIndexingConstants.ProductCategories) ?? Array.Empty<string>();
}
```

This approach provides strongly-typed properties for your custom fields while allowing dynamic access to any indexed field.

## Built-in Indexing Constants

The library provides a set of pre-defined constants for common Umbraco fields in `IndexingConstants.ComputedIndexFields`. You can reuse these in your queries and converters.

<details>
<summary><strong>List of `IndexingConstants.ComputedIndexFields`</strong></summary>

```csharp
public static class IndexingConstants
{
    public static class ComputedIndexFields
    {
        public const string ComputedFieldNameCommonPrefix = "computed";

        public static readonly IndexFieldDefinition<int> UmbracoNodeIdInt =
            IndexFieldDefinitionFactory.ForInt("intNodeId", isFacetable: false);

        public static readonly IndexFieldDefinition Url =
            IndexFieldDefinitionFactory.ForString(
                $"{ComputedFieldNameCommonPrefix}{nameof(Url)}",
                isFacetable: false,
                isSortable: true,
                isRaw: true,
                isMultiLanguage: true);

        public static readonly IndexFieldDefinition<float[]> VectorSearchContent =
            IndexFieldDefinitionFactory.Custom<float[]>(
                luceneFieldDefinition: null,
                azureFieldDefinition: new VectorSearchField(
                    $"{ComputedFieldNameCommonPrefix}{nameof(VectorSearchContent)}",
                    1536,
                    "vector-profile"));
    }
}
```

</details>
