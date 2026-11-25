---
name: umbraco-search
description: Use when building, extending or debugging search in an Umbraco 17/18 site that uses the IL.UmbracoSearch NuGet package - ISearchService, SearchParameters, filters, facets, suggestions, IIndexingConverter custom fields, result projections, Lucene or Azure AI Search engine choice, hybrid/vector search, minimal API or MCP search endpoints, or SearchSettings configuration.
---

# IL.UmbracoSearch

Search abstraction and extensible indexing for **existing** Umbraco/Examine indexes, with two engines: Lucene (Examine) or Azure AI Search. Applies to `IL.UmbracoSearch` 17.8.0.1 and later (including 18.0.x). The application source of the package is not available; this skill and its references are the contract.

Main runtime service: `ISearchService`. Main customization seam: `IIndexingConverter`.

## Non-negotiable contracts

- Register exactly one engine, `SearchOptions.Lucene` or `SearchOptions.Azure` (see `references/configuration.md`). Hybrid/vector behavior is Azure-only.
- The package never creates Lucene indexes. `SearchSettings.Indexes` lists existing Umbraco indexes; `DefaultIndexName` is used when a request omits `IndexName`. A wrong index name can mean a logged error and empty results, not a startup failure.
- Azure requires a valid `LicenseToken` and valid Azure settings at startup; Lucene needs no license.
- Declare every field you query, sort, facet or return with a typed field definition (`IndexFieldDefinitionFactory`); the field's shape (scalar vs collection, multi-language, facetable, sortable) is part of the query contract.
- Use the filter helper that matches the field shape: `FilterFor` for scalar fields, `FilterForCollectionField` for collection fields (analyzers `ILUS0002`/`ILUS0003` fail the build on a mismatch).
- Query languages with `SearchParameters.LanguageIsoCode`; never append culture suffixes to field names yourself.
- Lucene full-text without `SearchFields` searches only fields declared `isSearchable: true`; declare at least one.
- Everything exposed over HTTP or MCP is the application's to secure: authentication, rate limits, tenancy and input caps (including `Take`).
- Await every async API and forward cancellation tokens; converters are asynchronous (`ValueTask`).

## Decision guide

| Need | Use |
| --- | --- |
| Keyword search | `ISearchService.SearchAsync<T>(SearchParameters)` |
| Search one content type | `SearchParameters.Aliases` |
| Structured value filter | typed `ISearchFilter` (`FilterFor`, `FilterForCollectionField`, `RangeFilterFor`, `And`/`Or`/`Not`) |
| Counts for a filter UI | declare the field facetable, then `FacetOn` |
| Search within a subtree | `SearchParameters.Root` |
| Typed-ahead suggestions | `ISearchService.SuggestAsync<T>(SuggestionSettings)` |
| Hierarchical tags | `AddTaxonomyValue` + taxonomy settings |
| Own fields from Umbraco content/media | `IIndexingConverter` |
| Data from outside Umbraco | `IExternalIndexingConverter` |
| Small result model | `[SearchResultFieldProjection]` on a partial `SearchResultModelBase` model |
| Semantic/vector relevance | Azure + `UseHybridSearch` (and optionally chunk vectors) |
| Azure relevance tuning | `IScoringProfileProvider` |
| Ready-made browser endpoint | `UseSearch` / `UseSuggestionsSearch` |
| Search for an AI client | MCP tools (`AddUmbracoSearchMcpTools`) |
| Usage statistics | the `umbraco-search-analytics` skill |

## First search

```csharp
var results = await searchService.SearchAsync<CommonSearchItemModel>(new SearchParameters
{
    FullTextSearch = new FullTextSearch("umbraco"),
    Aliases = ["contentPage"],
    LanguageIsoCode = "en-US",
    Skip = 0,
    Take = 20,
    SearchOrderings =
    [
        ISearchOrdering.ByScore(OrderingType.Descending),
        ISearchOrdering.ByField(IndexingConstants.ComputedIndexFields.SearchDate, OrderingType.Descending)
    ]
});
```

`CommonSearchItemModel` is the all-fields result model (`Id`, `NodeName`, `SearchTitle`, `SearchDescription`, `SearchContent`, `SearchDate`, `Url`). `Take` defaults to `int.MaxValue`: always set it.

## Where to read next

- `references/configuration.md` - registration, `SearchSettings`, startup rules.
- `references/querying.md` - `SearchParameters`, filters, ordering, facets, suggestions, taxonomy, language, result models and projections.
- `references/indexing.md` - converters, field definitions, DI attributes, analyzers, Azure scoring profiles.
- `references/azure-and-vectors.md` - Azure lifecycle, blue/green rebuilds, hybrid and chunk vectors, background embeddings.
- `references/endpoints-and-mcp.md` - minimal API endpoints and MCP tools.
- `references/troubleshooting.md` - diagnosis order and definition of done.
- `references/compatibility-and-upgrade.md` - framework/Umbraco pairing, dependency ranges, breaking changes.
