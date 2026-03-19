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
  - [Advanced Usage](#advanced-usage)
    - [Advanced SearchApiController Example](#advanced-searchapicontroller-example)
    - [Multi-Language Support](#multi-language-support)
  - [Customizing the Index](#customizing-the-index)
    - [The `IIndexingConverter` Interface](#the-iindexingconverter-interface)
    - [Defining Index Fields](#defining-index-fields)
    - [Auto-Discovery with `[Service]` Attribute](#auto-discovery-with-service-attribute)
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
- **Extensible:** Add your own custom fields to the search index.

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
  "Azure": {
    "ServiceUrl": "YOUR_AZURE_SEARCH_SERVICE_URL",
    "ApiKey": "YOUR_AZURE_SEARCH_API_KEY",
    "UseHybridSearch": false
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

- **LicenseToken:** Your license token for the package.
- **DefaultIndexName:** The default index to use if not specified in a search.
- **CustomIndexSuffix:** Allows to specify custom suffix applied to all of your configured indexes. Optional.
- **Indexes**: A list of search index names to be used. Defaults to `["ExternalIndex"]`.
- **PreviewIndexes**: A list of search index names where soft deletion is enabled.
- **Azure:** Azure Search service credentials.
- **OpenAi:** OpenAI credentials for vector embeddings.
- **ReadOnly:** Disallows runtime to create or modify existing index in any way.
- **EnableBlueGreenIndexing:** Allows to enable b/g indexing behavior when rebuilding index, so that old functional index temporary remains available for search operations.
- **DisableSwapDelay:** System will delay index swap to allow for indexing to be finalized (1 min per 1000 items of delay); you can disable this behavior with this flag (probably for dev/test purpose).
- **EnableTaxonomyFacetExpansion:** Enables hierarchical taxonomy facet expansion for `TaxonomyTags` values. Keep disabled if you want to work with raw facet values only.

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
    Filters = new List<SearchFilterBase>
    {
        new SearchFilter
        {
            Fields = [IndexingConstants.ComputedIndexFields.NodeTypeAlias],
            Values = new[] { "contentPage" },
            FilteringBehavior = FilteringBehavior.And
        }
    },
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

#### Enabling hierarchical facet expansion

Set the following appsetting to `true`:

```json
"SearchSettings": {
  "EnableTaxonomyFacetExpansion": true
}
```

When enabled, the decorator expands taxonomy facets into a hierarchy:
- the root `Facet` is always bound to `TaxonomyCategories`
- `Facet.ChildFacetsIndexFieldName` tells you which field to use for the next level
- intermediate category nodes use `TaxonomyCategories`
- terminal tag nodes use `TaxonomyTags`

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
        ISearchFilter.FilterFor(IndexingConstants.ComputedIndexFields.TaxonomyCategories,
            "Category__SubCategory")
    ]
};
```

> `SharedTags` is still available as an obsolete alias for backwards compatibility; prefer `TaxonomyTags` for new code.

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
- **LanguageIsoCode:** The culture to search in.

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

## Customizing the Index

You can add your own custom fields to the search index by implementing the `IIndexingConverter` interface.

> [!IMPORTANT]
> `IndexFieldDefinition` constructors are no longer part of the public API.
> Always create field definitions through `IndexFieldDefinitionFactory` (`ForString`, `ForStringArray`, `ForInt`, `ForBool`, `Custom` or `AutoDetection`).

### The `IIndexingConverter` Interface

A class implementing this interface allows you to:

- **`GetIndexFieldDefinitions()`**: Return a collection of `IIndexFieldDefinition` objects that define your custom fields.
- **`AddContentComputedFields()`**: Add computed values to your custom fields for each document being indexed.
- **`AddMediaComputedFields()`**: Do the same for media items.
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

### Auto-Discovery with `[Service]` Attribute

Decorate your `IIndexingConverter` implementation with the `[Service]` attribute to have it automatically registered by the dependency injection container.

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

    public void AddContentComputedFields(IPublishedContent? publishedContent, IndexingModel indexingObject)
    {
        if (publishedContent != null)
        {
            indexingObject.SetComputedValue(CustomIndexingConstants.ProductName, "My Product");
            indexingObject.SetComputedValue(CustomIndexingConstants.ProductCategories, new[] { "Category1", "Category2" });
        }
    }
}
```

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
