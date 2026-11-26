# Querying

Inject `ISearchService`; `SearchAsync<T>(SearchParameters)` returns `SearchResults<T>` (`Items`, `TotalItemCount`, `Facets`, `Analytics`). `T` is `CommonSearchItemModel` or your own `SearchResultModelBase` model.

## `SearchParameters`

| Property | Notes |
| --- | --- |
| `IndexName` | Optional; falls back to `SearchSettings.DefaultIndexName`. |
| `FullTextSearch` | Keywords, optional field targeting and boosts. A plain `string` converts implicitly. |
| `Skip` / `Take` | Paging. `Take` defaults to `int.MaxValue`; always set it and cap public input. |
| `Aliases` | Umbraco content type aliases. |
| `Filters` | `List<ISearchFilter>`; all must match. |
| `SearchOrderings` | Relevance and/or typed field orderings. |
| `FacetOn` | Facets to compute (fields must be declared facetable). |
| `ExtraBoostingOptions` | `{ FieldDefinition, Value, Boost = 10 }` score boosts. |
| `Root` | Subtree by node id, key or `Guid`; accepts `int`, `string` or `Guid` implicitly. |
| `LanguageIsoCode` | Culture; normalized against configured Umbraco languages. |
| `IncludeExcludedFromSearch` | Opt in to normally excluded/soft-deleted content, only when intended. |
| `Analytics` | Optional analytics request (see the analytics skill). |
| `EngineSpecific` | Last-mile `AzureRequestOverride` / `LuceneRequestOverride`; you then own engine parity. |

## Full-text

`new FullTextSearch(query, searchFields, useWildcardSearch: true, useHybridSearch: false, boostExactPhrase: false)`. Every option is also a settable property (`new FullTextSearch(query) { UseHybridSearch = true, SearchMode = SearchMode.Any }`). `SearchMode` is the Azure SDK enum `Azure.Search.Documents.Models.SearchMode` on both engines and defaults to `All` (every term required); use `Any` for natural-language input.

- A complete `"quoted"` section is one required word group, not an exact phrase. With explicit `SearchFields`, every word of the group is bound to its field.
- Standalone `AND`, `OR`, `NOT` are treated as search terms; special characters are escaped.
- Wildcards apply only to terms longer than 2 characters (Azure matches exact and prefix; Lucene prefix).
- `BoostExactPhrase = true` also matches the whole unquoted input as an exact phrase with boost 10; input containing a double quote skips it.
- `SearchFields = [new SearchField(ProductFields.Name, 2f)]` is authoritative. Without it, Lucene searches only `isSearchable: true` fields.
- Azure-only: `UseHybridSearch`, `VectorBasedResultsWeight` (default 1), `VectorSimilarityThreshold`, `CustomVector` (see `azure-and-vectors.md`).

## Filters

```csharp
Filters =
[
    ISearchFilter.FilterFor(ProductFields.Sku, "A-100"),                        // scalar string
    ISearchFilter.FilterFor(IndexingConstants.ComputedIndexFields.NodeIdInt, 1234), // scalar int
    ISearchFilter.FilterForCollectionField(ProductFields.Tags, "sale", "new"),  // collection (default OR)
    ISearchFilter.RangeFilterFor(ProductFields.Stock, 1, 100),                  // numeric range
    ISearchFilter.And(
        ISearchFilter.FilterFor(ProductFields.Sku, "A-100"),
        ISearchFilter.Not(ISearchFilter.FilterFor(IndexingConstants.ComputedIndexFields.ItemType, "archived")))
]
```

- One leaf targets one field; combine fields with `And`, `Or`, `Not`. Pass a `FilteringBehavior` (`And`, `Or`, `Not`) before the values to control how multiple values combine (collection leaves default to `Or`).
- `FilterFor(IIndexFieldDefinition, params string[])` is the fallback when a field is only known at runtime.
- `FilterFor` on a collection field is analyzer `ILUS0002` (a build error). `FilterForCollectionField` on a scalar field needs no analyzer: its `TCollection : IEnumerable<TElement>` constraint makes the compiler reject it. The former `ILUS0003` was removed after 18.0.0.5 (18.0.0.5 and older still report it); delete any `NoWarn`, `#pragma` or `.editorconfig` entry that names it. A collection field is one whose type is an array or an `IEnumerable<T>` (`List<T>`, `HashSet<T>`, `IReadOnlyList<T>`, `IEnumerable<T>` itself), but not `string` and not a dictionary type (`Dictionary<,>`, `IReadOnlyDictionary<,>`, ...), which is a single-value field. `ILUS0002` only fires when the typed overload is chosen: `FilterFor(field, "text")` with an `IndexFieldDefinition<string[]>` (or `List<string>`, `HashSet<string>`, the built-in `TaxonomyTags`) binds to the untyped `FilterFor(IIndexFieldDefinition, params string[])` overload and compiles without an error. Releases after 18.0.0.5 warn with `ILUS0004`; always use `FilterForCollectionField` for collection fields.
- Keep the field's real type; filtering numbers or dates as strings is engine-specific and fragile. Lucene string filters are case-sensitive on stored values.

## Ordering, facets, boosts

- `ISearchOrdering.ByScore(OrderingType.Descending)` and `ISearchOrdering.ByField(field, OrderingType...)`; a sort field must be declared sortable. Azure drops score ordering for hybrid requests.
- `FacetOn = [new FacetOn(ProductFields.Tags, amountOfTopFacetsToTake: 20)]` returns `Facet`s keyed by the requested logical field name on both engines, including multi-language fields. A request with `Skip 0`, `Take 0` and facets returns totals and facets without items (covered by the package tests; very old builds may need `Take 1`, so confirm on the installed version before copying a `Take = 1` workaround). A string, `string[]`, `List<string>` or `HashSet<string>` field converts to `FacetOn` implicitly.

### Reading facets

`SearchResults<T>.Facets` is a `List<Facet>` (null when nothing was requested). Without taxonomy expansion each facet is one requested field:

| Member | Meaning |
| --- | --- |
| `Facet.IndexFieldName` | the logical field the facet was computed on (the name of the field you passed to `FacetOn`) |
| `Facet.Values` | `FacetOption`s: `Value` (the stored value to filter by), `HitsCount`, optional `Label` |
| `Facet.Label`, `Facet.Value`, `Facet.HitsCount` | set for taxonomy hierarchy nodes; otherwise usually empty |
| `Facet.ChildFacetsIndexFieldName`, `FacetOption.ChildFacets` / `ChildFacetsIndexFieldName` | next level of a taxonomy hierarchy and the field to filter that level by |

Find a facet by field instead of repeating string comparisons: `results.Facets.FacetFor(ProductFields.Tags)` and `results.Facets.FacetOptionsFor(ProductFields.Tags)` (releases after 18.0.0.5; earlier, `Facets?.FirstOrDefault(f => f.IndexFieldName == field.FieldName)`). Facet counts are computed over the whole filtered result set, including the filters the user just selected, so a "show all options" UI must run a second facet-only query without that filter. Reuse the same field definition objects for `FacetOn`, filters and lookups; do not hand-build field names.
- Use `ExtraBoostingOptions` only for explicit relevance policy and check it on the chosen engine. On Azure, a request with `ExtraBoostingOptions` forces `SearchMode.Any` whatever `FullTextSearch.SearchMode` says; if you need `All` with a relevance bias, use a scoring profile instead (`indexing.md`).

## Suggestions

`SuggestAsync<T>(new SuggestionSettings { Term, Size = 8, Fuzzy, Highlights, HighlightsTag = "b", Aliases, IndexName, LanguageIsoCode })` returns `SuggestionResult<SuggestionResultItem<T>>` (`Items` with `Item` and `Text`, `TotalItemCount`). The term is trimmed (blank returns nothing), `Size` is at least 1, and `HighlightsTag` must be a plain tag name such as `em`, otherwise `b` is used. Azure suggestions exclude content flagged excluded for the request language.

## Language

- Define fields `isMultiLanguage: true` only when values differ per culture; indexing expands them to culture-specific fields.
- Set `LanguageIsoCode`. `en`, `en-GB` and `en_GB` are equivalent; the closest configured language is used, otherwise the default language. The default language reads the base (unsuffixed) field.
- Never build suffixed names by hand.
- Test the default and one non-default culture for every multi-language feature.

## Taxonomy

`item.AddTaxonomyValue("Category", "SubCategory", "Tag")` (in a converter) stores the full path `Category__SubCategory__Tag` in `TaxonomyTags` and `Category__SubCategory` in `TaxonomyCategories`; a single segment adds only a tag. Another overload writes custom taxonomy fields.

- Facet on `TaxonomyTags`; filter `TaxonomyCategories` to drill down.
- With `EnableTaxonomyFacetExpansion`, the built-in facets become a hierarchy: a root `Facet` (bound to `TaxonomyCategories`) with `Values`, nested through `FacetOption.ChildFacets`; `ChildFacetsIndexFieldName` names the field for the next level. Expansion does not apply to custom taxonomy fields.
- With both expansion settings on, an `Or` filter spanning categories is regrouped by parent path into an `And` of per-category `Or` groups. Do not repeat that logic in UI code.

## Result models and projections

Derive from `SearchResultModelBase` and read fields lazily with `ValueFor(field)` (typed) or `ValueForAs<T>(field)` (serialized/custom values):

```csharp
[SearchResultFieldProjection]
public partial class ProductSearchResult : SearchResultModelBase
{
    public string? Sku => ValueFor(ProductFields.Sku);
    public string[] Tags => ValueFor(ProductFields.Tags) ?? [];
}
```

- `[SearchResultFieldProjection]` on a model that is not `partial` is silently ignored (the model keeps all-fields retrieval; releases after 18.0.0.5 warn with `ILUS0005`). On a `partial` model the bundled generator emits `RequiredIndexFields` from the fields used in `ValueFor`/`ValueForAs` and in methods marked `[SearchResultFieldAccessor]`; the engine then fetches only those fields. Do not declare `ISearchResultFieldProjection` or `RequiredIndexFields` yourself. Unmarked models keep all-fields retrieval.
- Projection is direct-only: an unmarked derived model does not inherit its base's projection. Each reusable base must itself be `partial` and marked.
- `CommonSearchResultProjection` is the projected common base; `CommonSearchItemModel` is the ordinary unprojected one. Across assemblies a model composes `RequiredIndexFields` from its nearest marked base; mark intermediate bases whose fields must flow further.
- Every projected field must be registered by an indexing converter and stored by the engine's schema.
- Both engines always add the node key to what they fetch, so analytics works for projected models.

## Decorating `ISearchService`

Wrapping the search service is supported and is the place for behavior that must apply to every search (relabelling or dropping facets, default parameters, logging). Register the decorator with `[Decorator<SearchOptions>(Feature = SearchOptions.Lucene | SearchOptions.Azure)]` and implement both `SearchAsync<T>` and `SuggestAsync<T>` (forward the one you do not change). Several decorators are ordered with `DecorationOrder`; give them distinct values and cover the wrapping order with a test. Prefer a decorator or a small application service over copying the same post-processing into every page converter, and keep it cheap: it runs on every request.
