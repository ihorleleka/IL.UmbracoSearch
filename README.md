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
  "PreviewIndexes": ["InternalIndex"],
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
- **Indexes**: A list of search index names to be used. Defaults to `["ExternalIndex"]`.
- **PreviewIndexes**: A list of search index names where soft deletion is enabled.
- **Azure:** Azure Search service credentials.
- **OpenAi:** OpenAI credentials for vector embeddings.

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
        new FacetOn(IndexingConstants.ComputedIndexFields.SharedTags)
    },
    LanguageIsoCode = "en-US"
};

var searchResults = await searchService.SearchAsync<CommonSearchItemModel>(searchParameters);
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

Multi-language field can be defined by activating this option in constructors of IndexFieldDefinition<> class  `(..., multiLanguage: true)`.

Set the `LanguageIsoCode` property in `SearchParameters` to search in a specific language.

## Customizing the Index

You can add your own custom fields to the search index by implementing the `IIndexingConverter` interface.

### The `IIndexingConverter` Interface

A class implementing this interface allows you to:

- **`GetIndexFieldDefinitions()`**: Return a collection of `IIndexFieldDefinition` objects that define your custom fields.
- **`AddContentComputedFields()`**: Add computed values to your custom fields for each document being indexed.
- **`AddMediaComputedFields()`**: Do the same for media items.
- **`RunForIndexes()`**: Specify which indexes the converter should run for.

### Defining Index Fields

Create instances of `IndexFieldDefinition<T>` for your custom fields. It's best practice to group these in a static constants class.

```csharp
public static class CustomIndexingConstants
{
    public const string CustomFieldPrefix = "custom_";

    // For string fields that need to be full-text searchable and sortable
    public static readonly IndexFieldDefinition ProductName = new(
        luceneFieldDefinition: new FieldDefinition(name: $"{CustomFieldPrefix}productName", type: FieldDefinitionTypes.FullText),
        azureFieldDefinition: new SearchField(name: "custom_productName", type: SearchFieldDataType.String) { IsSearchable = true, IsSortable = true }
    );

    // For multi-value string fields that need to be filterable and facetable
    public static readonly IndexFieldDefinition<string[]> ProductCategories = new(
        luceneFieldDefinition: new FieldDefinition(name: $"{CustomFieldPrefix}productCategories", type: IndexingConstants.LuceneFaceting.FieldDefinitionTypes.FacetFullText),
        azureFieldDefinition: new SearchField(name: "custom_productCategories", type: SearchFieldDataType.Collection(SearchFieldDataType.String)) { IsFilterable = true, IsFacetable = true }
    );
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

        public static readonly IndexFieldDefinition UmbracoNodeId = new(new SearchableField("nodeId") { IsKey = true });
        public static readonly IndexFieldDefinition<int> UmbracoNodeIdInt = new FieldDefinition("intNodeId", FieldDefinitionTypes.Integer);
        // ... and many more
    }
}
```

</details>
```
