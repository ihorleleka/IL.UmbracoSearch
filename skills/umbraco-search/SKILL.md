---
name: umbraco-search
description: Use when building, extending or debugging search in an Umbraco 17/18 site that uses the IL.UmbracoSearch NuGet package - ISearchService, SearchParameters, filters, facets, suggestions, IIndexingConverter custom fields, result projections, Lucene or Azure AI Search engine choice, hybrid/vector search, minimal API or MCP search endpoints, or SearchSettings configuration.
---

# IL.UmbracoSearch

Search abstraction and extensible indexing for **existing** Umbraco/Examine indexes, with two engines: Lucene (Examine) or Azure AI Search. Written against `IL.UmbracoSearch` 18.0.0.5 and the current 17.8.x line; older 17.8.x builds differ in dependency ranges and analyzers (see `references/compatibility-and-upgrade.md`). The application source of the package is not available; this skill and its references are the contract.

Main runtime service: `ISearchService`. Main customization seam: `IIndexingConverter`.

## Non-negotiable contracts

- Register exactly one engine, `SearchOptions.Lucene` or `SearchOptions.Azure` (see `references/configuration.md`). Hybrid/vector behavior is Azure-only.
- The package never creates Lucene indexes. `SearchSettings.Indexes` lists existing Umbraco indexes; `DefaultIndexName` is used when a request omits `IndexName`. A wrong index name can mean a logged error and empty results, not a startup failure.
- Azure requires a valid `LicenseToken` (an invalid one throws) and valid Azure settings (invalid ones are logged and search returns nothing; the host keeps running); Lucene needs no license.
- Declare every field you query, sort, facet or return with a typed field definition (`IndexFieldDefinitionFactory`); the field's shape (scalar vs collection, multi-language, facetable, sortable) is part of the query contract.
- Use the filter helper that matches the field shape: `FilterFor` for scalar fields, `FilterForCollectionField` for collection fields. The analyzers enforce this only partly (see "Compiles but does nothing" below), so check it yourself.
- Query languages with `SearchParameters.LanguageIsoCode`; never append culture suffixes to field names yourself.
- Lucene full-text without `SearchFields` searches only fields declared `isSearchable: true`; declare at least one.
- Everything exposed over HTTP or MCP is the application's to secure: authentication, rate limits, tenancy and input caps (including `Take`).
- Await every async API and forward cancellation tokens; converters are asynchronous (`ValueTask`).

## Compiles but does nothing

These mistakes build and run. Package versions after 18.0.0.5 warn about them (analyzers `ILUS0004`-`ILUS0006`, warnings only); older versions do not warn at all, so verify them by reading the code.

- `AddMediaComputedFieldsAsync`, `Order` and `RunForIndexes` have default implementations, so a member with the right name and a wrong signature (`async Task AddMediaComputedFieldsAsync`, `IEnumerable<string> RunForIndexes()`, `long Order`) compiles and is silently never called. Copy the interface signature exactly: `ValueTask AddMediaComputedFieldsAsync(IPublishedContent? publishedContent, IndexingModel indexingObject, IndexingItemContext context, CancellationToken cancellationToken = default)`.
- `[SearchResultFieldProjection]` on a class that is not `partial` is ignored without any message and the model keeps fetching every field.
- `ISearchFilter.FilterFor(collectionField, "value")` binds to the untyped `FilterFor(IIndexFieldDefinition, params string[])` overload, so analyzer `ILUS0002` cannot see it (`ILUS0004` warns in newer packages). It currently works, but use `FilterForCollectionField`.
- Same-`Order` converters run concurrently: two of them writing one field race, and the winner is not deterministic.

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
| Read one facet from a result | `results.Facets.FacetFor(field)` / `FacetOptionsFor(field)` (releases after 18.0.0.5; before that `FirstOrDefault(f => f.IndexFieldName == field.FieldName)`) |
| Re-index external records on demand | `IExternalRebuildIndexingRunner` (see `references/indexing.md`) |
| Change or relabel results of every search | decorate `ISearchService` (see `references/querying.md`) |
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
