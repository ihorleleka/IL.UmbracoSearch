# Agent-Readme

Purpose: implementation playbook for coding agents integrating `IL.UmbracoSearch` in real projects.

## Core Model

- Primary responsibilities:
  - indexing Umbraco data into existing indexes
  - querying with full-text, filters, sorting, facets, boosts, suggestions
  - engine abstraction over Lucene and Azure
- Main extension seam: `IIndexingConverter`
- Main runtime entrypoint: `ISearchService`

## Engine Choice

- `SearchOptions.Lucene`:
  - no vector search
  - strong local/full-text/facets path
- `SearchOptions.Azure`:
  - full-text + filters + facets + boosts + suggestions
  - optional hybrid vector search
  - optional granular chunk vectors

Agents must align implementation with enabled feature flag(s) in DI registration.

## Bootstrapping Checklist

1. Register feature:
```csharp
builder.AddServiceAttributeBasedDependencyInjection(options =>
{
    options.AddFeature(SearchOptions.Lucene); // or SearchOptions.Azure
});
```
2. Add `SearchSettings` config section.
3. Ensure `LicenseToken` is valid (initialization is fail-fast on invalid token).
4. Ensure `Indexes` names match real existing Umbraco indexes.

## SearchSettings Keys You Usually Need

- Always:
  - `LicenseToken`
  - `DefaultIndexName`
  - `Indexes`
  - optional `PreviewIndexes`
  - optional `CustomIndexSuffix`
  - optional `EnableBlueGreenIndexing`
  - optional `ReadOnly`
- Taxonomy behavior:
  - `EnableTaxonomyFacetExpansion`
  - `EnableTaxonomyFiltersExpansion`
- Azure mode:
  - `Azure.ServiceUrl`, `Azure.ApiKey`
  - `Azure.UseHybridSearch`
  - `Azure.UseGranularHybridSearch`
  - chunk tuning + background settings
- Hybrid search mode:
  - `OpenAi.ServiceUrl`, `OpenAi.ApiKey`, `OpenAi.EmbeddingsDeploymentName`

## Query Construction Contract

Use `SearchParameters` and call:
```csharp
await searchService.SearchAsync<CommonSearchItemModel>(parameters);
```

Important fields:
- `FullTextSearch`
- `Skip`, `Take`
- `Aliases`
- `SearchOrderings`
- `Filters`
- `FacetOn`
- `ExtraBoostingOptions`
- `Root`
- `LanguageIsoCode`
- `EngineSpecific` (last-mile escape hatch)

## Filtering Rules (High Priority)

Use typed helpers correctly:
- scalar fields: `ISearchFilter.FilterFor(...)`
- collection fields: `ISearchFilter.FilterForCollectionField(...)`
- dynamic field fallback: `ISearchFilter.FilterFor(IIndexFieldDefinition, ...)`

Analyzer-enforced mistakes:
- `ILUS0002`: scalar method used with collection field
- `ILUS0003`: collection method used with scalar field

Cross-field logic must be composed via filter trees (`And`/`Or`/`Not`), not packed into one leaf.

## Sorting, Boosting, Faceting

- Sorting:
  - use `ISearchOrdering.ByScore(...)` / `ByField(...)`
- Boosting:
  - use `ExtraBoostingOptions` with field + value + boost factor
- Facets:
  - use `FacetOn(field)`
  - supports `AmountOfTopFacetsToTake`
  - supports `AllowMultiLanguageFaceting`

## Taxonomy Behavior

Indexing taxonomy:
- use `indexingObject.AddTaxonomyValue("Category", "SubCategory", "Tag")`
- library writes:
  - full path to `TaxonomyTags`
  - category path to `TaxonomyCategories`

Querying taxonomy:
- facet on `TaxonomyTags`
- filter by `TaxonomyCategories` for drilldown

Optional normalization:
- with both `EnableTaxonomyFacetExpansion=true` and `EnableTaxonomyFiltersExpansion=true`,
  - taxonomy `OR` filters are regrouped by category path
  - facet tree is expanded into hierarchy for UI rendering

## Indexing Converter Contract

Implement `IIndexingConverter` and include DI attribute (`[Service]`, etc.).
Analyzer `ILUS0001` enforces registration attributes for concrete converters.

Guidelines:
- expose fields via `IndexFieldDefinitionFactory`
- keep field names stable and explicit
- use multi-language field semantics when needed
- add computed fields in `AddContentComputedFields` / `AddMediaComputedFields`

## Language / Multi-language Rules

- `LanguageIsoCode` in queries is normalized/resolved by internal language services.
- Many built-in fields have language-variant expansions.
- Use `LanguageInvariantFieldName(...)` when constructing low-level field-aware custom logic.

## Built-in HTTP Endpoint Surface

Minimal API helpers exist:
- `UseSearch(...)`
- `UseSuggestionsSearch(...)`

Default request DTOs:
- `SearchRequest`
- `SuggestionsSearchRequest`

Unknown request fields are ignored by default mapper.

## Search Result Models

- `CommonSearchItemModel` is the default pragmatic model.
- For custom models, inherit from `SearchResultModelBase` and use `ValueFor<T>(fieldDefinition)`.

## Vector/Hybrid Subsystem (Azure Only)

Use only when `Azure.UseHybridSearch=true`.

Public vector content methods:
- `SetVectorContent(...)`
- `SetVectorContentPrecise(...)`
- `SetVectorContentPreciseManual(min,max,overlap)`

Behavior:
- average vector is the baseline
- chunk vectors require `UseGranularHybridSearch=true`
- if precise content is absent, chunk flow reuses `SetVectorContent(...)` content
- background mode (`GranularHybridBackgroundEmbedding=true`) makes chunk indexing eventually consistent

Quota and chunking:
- chunk vectors are capped effectively to Azure multi-vector quota (100)
- system auto-adjusts chunk size upward if config would exceed quota and logs warning

## Common Failure Modes Agents Should Check First

1. Wrong index name / missing configured index.
2. Feature mismatch (code assumes Azure but Lucene feature enabled).
3. Missing DI attribute on converter (analyzer failure).
4. Wrong filter helper (`FilterFor` vs `FilterForCollectionField`).
5. Hybrid enabled but missing OpenAI settings.
6. Live test in-memory DB misconfigured with non-shared DB names across scopes.

## High-Value Files To Inspect During Debugging

- `src/UmbracoSearch/Search/Common/SearchInitialization.cs`
- `src/UmbracoSearch/Search/Orchestration/OrchestrationService.cs`
- `src/UmbracoSearch/Search/Common/SearchRequestNormalizer.cs`
- `src/UmbracoSearch/Search/Services/Common/SearchServiceBase.cs`
- `src/UmbracoSearch/Search/Services/Lucene/LuceneSearchService.cs`
- `src/UmbracoSearch/Search/Services/Azure/AzureSearchService.cs`
- `src/UmbracoSearch/Search/Indexing/Common/Factories/IndexFieldDefinitionFactory.cs`
- `src/UmbracoSearch/Search/Indexing/IndexingConstants.cs`

## Agent Workflow Recommendation

1. Start with non-vector full-text + filters + sorting + facets.
2. Add taxonomy indexing/facets when tag-based classification is needed:
   - flat tags are supported
   - hierarchical taxonomy paths are also supported
3. Add boosts and language behavior.
4. Add hybrid vectors only when relevance needs semantic lift.
5. Add granular/background vector mode only after baseline search behavior is stable.
