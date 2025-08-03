[![NuGet version (IL.AttributeBasedDI)](https://img.shields.io/nuget/v/IL.UmbracoSearch.svg?style=flat-square)](https://www.nuget.org/packages/IL.UmbracoSearch/)
# IL.UmbracoSearch

A comprehensive search solution for Umbraco, supporting both Lucene and Azure Search, with extensible indexing and flexible search parameters.

## Configuration

> [!NOTE]
> Library is unable to create new indexes that are not defined in Umbraco by default (for now). It can only attach to existing indexes (Internal,External etc).

### Service Registration

In your `Program.cs`, you need to register the services from this package. You can do this by calling the `AddServiceAttributeBasedDependencyInjection` extension method.

```csharp
// Program.cs
    // Add the following line
builder.AddServiceAttributeBasedDependencyInjection(options =>
    {
        options.AddFeature(SearchOptions.Lucene);
        // or options.AddFeature(SearchOptions.Azure);
    });
```

This will register all the services decorated with the `[Service]` attribute that match the configured feature.

### AppSettings

To configure the IL.UmbracoSearch package, you need to add a `SearchSettings` section to your `appsettings.json` file. This section contains the settings for the search providers (Azure, OpenAI) and the license token.

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

### Configuration Details

- **LicenseToken:** Your license token for the IL.UmbracoSearch package.
- **DefaultIndexName:** Index name to be used by search service if no index name parameter was specified manually.
- **Indexes**: A list of search index names to be used. Defaults to `["ExternalIndex"]`.
- **PreviewIndexes**: A list of search index names where soft deletion is enabled (soft deletion means item will be marked as ExcludedFromSearch instead of deleted).
- **Azure:**
    - **ServiceUrl:** The URL of your Azure Search service.
    - **ApiKey:** The API key for your Azure Search service.
    - **UseHybridSearch:** A boolean to enable hybrid search. Defaults to `false`.
- **OpenAi:**
    - **ApiKey:** Your OpenAI API key.
    - **ServiceUrl:** The URL of your OpenAI service.
    - **EmbeddingsDeploymentName:** The name of the embeddings deployment.

## Search Overview

This package provides a robust search implementation for Umbraco, supporting both Lucene and Azure Search.

### Features

- **Full-text search:** Search for keywords in the content of the documents.
- **Filtering:** Filter search results by various criteria, such as document type, date range, and tags.
- **Sorting:** Sort search results by relevance, date, or any other field.
- **Faceting:** Get a count of the number of documents that match each value of a field.
- **Hybrid search (Azure only):** Combine keyword search with vector search for more relevant results.

### How to Use

To use the search functionality, you need to inject the `ISearchService` into your code. Then, you can call the `SearchAsync` method to perform a search. The `SearchAsync` method takes a `SearchParameters` object as a parameter. This object allows you to specify the search query, filters, sorting, and other options.

A good example of how to use the `ISearchService` can be found in the `SearchApiController` in the `SearchApiController Example` project. This controller shows how to build a search query from URL parameters.

#### `SearchApiController` Example

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
        [FromQuery] string? indexName = null
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
                        IndexName = indexName
                    },
                    HttpContext.RequestAborted
                )
            );

    /// <summary>
    /// Handles boostById string of kind ?boostById=1072
    /// </summary>
    /// <param name="boostById"></param>
    /// <returns></returns>
    private static List<ExtraBoostingOption>? TryBuildExtraBoostingOptions(string? boostById)
    {
        if (string.IsNullOrEmpty(boostById))
        {
            return null;
        }

        return boostById
            .Split(',')
            .Select(x => new ExtraBoostingOption
            {
                Boost = 10f,
                FieldDefinition = IndexingConstants.ComputedIndexFields.UmbracoNodeId,
                Value = x
            })
            .ToList();
    }

    /// <summary>
    /// Handles orderBy string of kind ?orderBy=score:desc;anotherSortableIndexField:desc
    /// </summary>
    /// <param name="orderBy"></param>
    /// <returns></returns>
    private List<ISearchOrdering>? TryBuildSearchOrderings(string? orderBy, string? indexName)
    {
        if (string.IsNullOrEmpty(orderBy))
        {
            return
            [
                ISearchOrdering.ByScore(),
                ISearchOrdering.ByField(IndexingConstants.ComputedIndexFields.SearchDate, OrderingType.Descending)
            ];
        }

        return orderBy
            .Split(';')
            .Select(ISearchOrdering (orderByEntry) =>
            {
                var parts = orderByEntry.Split(':');
                var orderingType = parts.Last() switch
                {
                    "desc" => OrderingType.Descending,
                    "asc" => OrderingType.Ascending,
                    _ => OrderingType.Descending
                };
                return parts[0] switch
                {
                    "score" => ISearchOrdering.ByScore(orderingType),
                    _ => ISearchOrdering.ByField(indexService.GetFieldDefinitions(indexName)[parts[0]], orderingType)
                };
            })
            .ToList();
    }

    /// <summary>
    /// Handles facetOn string of kind ?facetOn=indexFieldName;anotherFieldName
    /// </summary>
    /// <param name="facetOn"></param>
    /// <returns></returns>
    private List<FacetOn>? TryBuildFacetOn(string? facetOn, string? indexName)
    {
        return !string.IsNullOrEmpty(facetOn)
            ? facetOn
                .Split(',')
                .Select(x => new FacetOn(indexService.GetFieldDefinitions(indexName)[x]))
                .ToList()
            : null;
    }

    /// <summary>
    /// Handles filter string of kind ?filters=indexFieldName:value1,value2;anotherFieldName:value3,value4
    /// </summary>
    /// <param name="filters"></param>
    /// <returns></returns>
    private List<SearchFilterBase>? TryBuildFilters(string? filters, string? indexName) =>
        !string.IsNullOrEmpty(filters)
            ? filters
                .Split(';')
                .Select(SearchFilterBase (x) =>
                {
                    var keyValue = x.Split(':');
                    var indexField = indexService.GetFieldDefinitions(indexName)[keyValue[0]];
                    var value = keyValue[1];
                    return new SearchFilter
                    {
                        Fields = [indexField],
                        Values = value.Split(','),
                        FilteringBehavior = FilteringBehavior.Or
                    };
                }).ToList()
            : null;
    // or use direct filter building logic, for example
    // (range filters is quite specific logic, so I'd imagine you would need have to introduce separate QS parameter for it)
    //[
    // new RangeFilter<int>
    // {
    //     Fields = [IndexingConstants.ComputedIndexFields.UmbracoNodeIdInt],
    //     IncludeMin = true,
    //     IncludeMax = true,
    //     Min = 1070,
    //     Max = 1073
    // }
    //]
}
```

The `SearchApiController` exposes a `Search` endpoint that can be called with the following parameters:

- `q`: The search query.
- `useHybridSearch`: A boolean to enable hybrid search.
- `filters`: A string to filter the results, e.g., `indexedField:value1,value2;anotherField:value3`.
- `orderBy`: A string to order the results, e.g., `score:desc;dateField:asc`.
- `facetOn`: A string to specify the fields to facet on, e.g., `field1,field2`.
- `boostById`: A string to boost results by ID, e.g., `123,456`.
- `skip`: The number of results to skip.
- `take`: The number of results to take.
- `root`: The root node ID to search under.

An example of a URL to call the API:
`/Search/Search?q=umbraco&filters=__NodeTypeAlias:contentPage&orderBy=score:desc`

#### `SearchParameters`

The `SearchParameters` object has the following properties:

- **FullTextSearch:** The full-text search query.
- **Skip:** The number of results to skip.
- **Take:** The number of results to take.
- **Aliases:** The document type aliases to search for.
- **SearchOrderings:** The order in which to sort the search results.
- **Root:** The root node to search in.
- **Filters:** The filters to apply to the search results.
- **FacetOn:** The fields to get facets for.
- **ExtraBoostingOptions:** The options to boost the score of documents.
- **IndexName:** The options to specify which index to search against.

#### C# Example

Here is an example of how to use the `ISearchService` directly:

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
    ExtraBoostingOptions = new List<ExtraBoostingOption>
    {
        new ExtraBoostingOption
        {
            Boost = 10f,
            FieldDefinition = IndexingConstants.ComputedIndexFields.UmbracoNodeId,
            Value = "1234"
        }
    }
};

var searchResults = await searchService.SearchAsync<CommonSearchItemModel>(searchParameters);
```
This example demonstrates a search for "umbraco" on `contentPage` document types, with hybrid search enabled, ordering by score and date, filtering by node type alias, faceting on shared tags, and boosting a specific document from External Index by default.

## Indexing

The indexing process is extensible, allowing you to add custom fields to the search index. This is achieved by creating a class that implements the `IIndexingConverter` interface and registering it in the dependency injection container. The system uses attribute-based dependency injection to automatically discover and register these converters.

### `IIndexingConverter` Interface

The `IIndexingConverter` interface has two methods:

- `RunForIndexes()`: This method should return a collection of `string` names of indexes. Leave empty if converter should run for all indexes, or specify names.
- `GetIndexFieldDefinitions()`: This method should return a collection of `IIndexFieldDefinition` objects. These objects define the fields that will be added to the index.
- `AddContentComputedFields(IPublishedContent? publishedContent, IndexingModel indexingObject, string? indexFieldNameSuffix = null)`: This method is called for each document being indexed. It allows you to add computed values to the fields defined in `GetIndexFieldDefinitions()`.
- `AddMediaComputedFields(IPublishedContent? publishedContent, IndexingModel indexingObject, string? indexFieldNameSuffix = null)`: This method is called for each media item being indexed. It allows you to add computed values to the fields defined in `GetIndexFieldDefinitions()`.

### Defining Index Fields

To define a new index field, you need to create an instance of the `IndexFieldDefinition<T>` class, where `T` is the data type of the field. The following data types are supported:

- `string`
- `int`
- `long`
- `bool`
- `string[]` (for multi-value string fields)
- `float[]` (for vector fields)

When creating an `IndexFieldDefinition`, it is important to follow the patterns established in the `IndexingConstants.cs` file to ensure compatibility with the underlying search providers (Lucene and Azure Search).

#### Grouping Index Fields in Constants

It is a good practice to group your custom index field definitions in a static class, similar to `IndexingConstants.ComputedIndexFields`. This makes it easier to reuse the definitions and avoids magic strings in your code.

Here is an example of a constants class with different types of index fields, following the correct patterns:

```csharp
using IL.UmbracoSearch.Search.Indexing.Common.Factories;
using IL.UmbracoSearch.Search.Indexing.Common.Models;

public static class CustomIndexingConstants
{
    public const string CustomFieldPrefix = "custom_";

    // For string fields that need to be full-text searchable and sortable
    public static readonly IndexFieldDefinition ProductName = new($"{CustomFieldPrefix}productName", FieldDefinitionTypes.FullText, new SearchField("custom_productName", SearchFieldDataType.String) { IsSearchable = true, IsSortable = true });

    // For integer fields that need to be filterable and sortable
    public static readonly IndexFieldDefinition<int> ProductPrice = new($"{CustomFieldPrefix}productPrice", FieldDefinitionTypes.Integer, new SearchField("custom_productPrice", SearchFieldDataType.Int32) { IsFilterable = true, IsSortable = true });

    // For boolean fields, always use the IndexFieldDefinitionFactory.ForBool method
    public static readonly IndexFieldDefinition<bool> IsInStock = IndexFieldDefinitionFactory.ForBool($"{CustomFieldPrefix}isInStock");

    // For multi-value string fields that need to be filterable and facetable
    public static readonly IndexFieldDefinition<string[]> ProductCategories = new($"{CustomFieldPrefix}productCategories", IndexingConstants.LuceneFaceting.FieldDefinitionTypes.FacetFullText, new SearchField("custom_productCategories", SearchFieldDataType.Collection(SearchFieldDataType.String)) { IsFilterable = true, IsFacetable = true });
}
```

### Auto-Discovery with `[Service]` Attribute

To have your custom `IIndexingConverter` automatically registered, you need to decorate it with the `[Service]` attribute from the `IL.AttributeBasedDI` library. This attribute tells the dependency injection container to register your class. You can also specify the lifetime of the service (e.g., `ServiceLifetime.Singleton`).

### Example of a Custom Indexing Converter

Here is an example of a custom indexing converter that adds a `customField` to the index:

```csharp
using IL.AttributeBasedDI.Attributes;
using Microsoft.Extensions.DependencyInjection;
using IL.UmbracoSearch;
using IL.UmbracoSearch.Search.Indexing;
using IL.UmbracoSearch.Search.Indexing.Common.Converters;
using IL.UmbracoSearch.Search.Indexing.Common.Models;
using Umbraco.Cms.Core.Models.PublishedContent;

namespace MyProject.Indexing;

[Service<SearchOptions>(Lifetime = ServiceLifetime.Singleton, Feature = SearchOptions.Lucene | SearchOptions.Azure)]
public class CustomIndexingConverter : IIndexingConverter
{
    public int Order => 100; // Higher order means it runs after the core converters

    public IEnumerable<IIndexFieldDefinition> GetIndexFieldDefinitions()
    {
        return new IIndexFieldDefinition[]
        {
            CustomIndexingConstants.ProductName,
            CustomIndexingConstants.ProductPrice,
            CustomIndexingConstants.IsInStock,
            CustomIndexingConstants.ProductCategories
        };
    }

    public void AddContentComputedFields(IPublishedContent? publishedContent, IndexingModel indexingObject, string? indexFieldNameSuffix = null)
    {
        if (publishedContent != null)
        {
            // Add your custom logic here to get the values for the custom fields
            indexingObject.SetComputedValue(CustomIndexingConstants.ProductName, "My Product");
            indexingObject.SetComputedValue(CustomIndexingConstants.ProductPrice, 100);
            indexingObject.SetComputedValue(CustomIndexingConstants.IsInStock, true);
            indexingObject.SetComputedValue(CustomIndexingConstants.ProductCategories, new[] { "Category1", "Category2" });
        }
    }
}
```

In this example:

- The `CustomIndexingConverter` is registered as a singleton service for the `Search` feature.
- The `GetIndexFieldDefinitions` method returns the custom index field definitions from the `CustomIndexingConstants` class.
- The `AddContentComputedFields` method sets the values for the custom fields.

By following this pattern, you can easily extend the search index with your own custom data.

## Models

### Facet Models

This folder contains the models for facets.

- **Facet.cs:** This class represents a facet, which is a way of refining search results.
- **FacetOn.cs:** This class represents a request to get facets for a specific field.
- **FacetOption.cs:** This class represents a single option within a facet.
- **FacetWithDisplayName.cs:** This class represents a facet option with a display name.

### Search Result Models

This folder contains the models for search results, which are used to represent individual search results returned by the search service.

#### `CommonSearchItemModel`

The `CommonSearchItemModel.cs` class is a concrete implementation of `SearchResultModelBase` that provides convenient access to commonly used search result fields such as `Id`, `NodeName`, `SearchTitle`, `SearchDescription`, `SearchContent`, `SearchDate`, `Tags`, and `Url`. This model is suitable for most general search result displays.

#### Customizing Search Result Models with `ValueFor<T>`

While `CommonSearchItemModel` covers many typical scenarios, you can create your own custom search result models by inheriting from `SearchResultModelBase`. This allows you to define properties that map directly to your custom index fields.

The `SearchResultModelBase` provides a generic `ValueFor<T>(IndexFieldDefinition<T> fieldDefinition)` method. This powerful method allows you to extract the value of any field from the search result using its `IndexFieldDefinition`. This means you can retrieve values for fields that are not explicitly defined as properties in your custom search result model, or even for fields that are not part of `CommonSearchItemModel`.

##### Example: Creating a Custom Search Result Model

Here's how you can create a custom search result model and use `ValueFor<T>` to access specific fields:

```csharp
using IL.UmbracoSearch.Search.Indexing;
using IL.UmbracoSearch.Search.Models.SearchResults;
using IL.UmbracoSearch.Search.Indexing.Common.Models; // Needed for IndexFieldDefinition

namespace MyProject.Search.Models;

public class ProductSearchResultModel : SearchResultModelBase
{
    public string ProductId => ValueFor(CustomIndexingConstants.ProductIdField) ?? string.Empty;
    public string ProductName => ValueFor(CustomIndexingConstants.ProductNameField) ?? string.Empty;
    public decimal Price => ValueFor(CustomIndexingConstants.ProductPriceField);

    // You can also access other fields dynamically using ValueFor
    public string? Category => ValueFor(CustomIndexingConstants.ProductCategoryField);

    // Example of accessing a field not explicitly mapped as a property
    public string? GetSupplierName()
    {
        // Assuming you have an IndexFieldDefinition for SupplierName
        return ValueFor(CustomIndexingConstants.ProductSupplierNameField);
    }
}

// In your IndexingConstants (or similar) file:
public static class CustomIndexingConstants
{
    public static readonly IndexFieldDefinition<string> ProductIdField = new("productId", FieldDefinitionTypes.FullText);
    public static readonly IndexFieldDefinition<string> ProductNameField = new("productName", FieldDefinitionTypes.FullText);
    public static readonly IndexFieldDefinition<decimal> ProductPriceField = new("productPrice", FieldDefinitionTypes.Double);
    public static readonly IndexFieldDefinition<string> ProductCategoryField = new("productCategory", FieldDefinitionTypes.FullText);
    public static readonly IndexFieldDefinition<string> ProductSupplierNameField = new("productSupplierName", FieldDefinitionTypes.FullText);
}
```

In this example:

- The `ProductSearchResultModel` inherits from `SearchResultModelBase`.
- Properties like `ProductId`, `ProductName`, and `Price` directly use `ValueFor` with their respective `IndexFieldDefinition` to retrieve values.
- The `GetSupplierName()` method demonstrates how you can use `ValueFor` to fetch a field's value even if it's not a direct property on the model.

This approach provides flexibility, allowing you to define strongly-typed properties for frequently accessed fields while retaining the ability to dynamically retrieve any indexed field using its `IndexFieldDefinition`.
