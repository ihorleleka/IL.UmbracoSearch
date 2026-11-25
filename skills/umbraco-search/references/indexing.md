# Custom indexing

Add your own fields by implementing `IIndexingConverter` in the application. The package discovers it through attribute-based DI; no further registration is needed.

## Field definitions

Create every definition with `IndexFieldDefinitionFactory`; the `IndexFieldDefinition` constructors are not public. Keep them in one static class so queries, converters and result models share them.

```csharp
public static class ProductFields
{
    public static readonly IndexFieldDefinition<string> Sku =
        IndexFieldDefinitionFactory.ForString("productSku", isRaw: true, isSortable: true);

    public static readonly IndexFieldDefinition<string> Name =
        IndexFieldDefinitionFactory.ForString("productName", isSearchable: true, isMultiLanguage: true);

    public static readonly IndexFieldDefinition<string[]> Tags =
        IndexFieldDefinitionFactory.ForStringArray("productTags", isFacetable: true, isRaw: true);

    public static readonly IndexFieldDefinition<int> Stock = IndexFieldDefinitionFactory.ForInt("stockLevel");

    // Azure-only vector field; the Lucene part is intentionally null.
    public static readonly IndexFieldDefinition<float[]> Vector =
        IndexFieldDefinitionFactory.Custom<float[]>(
            luceneFieldDefinition: null,
            azureFieldDefinition: new VectorSearchField("custom_productVector", 1536, "vector-profile"));
}
```

- Factory methods: `ForString`, `ForStringArray`, `ForInt`, `ForLong`, `ForBool`, `AutoDetection<T>`, `Custom<T>(luceneField, azureField)`.
- Flags (string/array/number as applicable): `isSearchable`, `isFacetable`, `isSortable`, `isFilterable`, `isRaw` (untokenized exact value), `isMultiLanguage` (one field per language).
- A definition has an optional Lucene part and an optional Azure part. A Lucene-only definition never reaches the Azure schema, and an Azure-only one never reaches Lucene.
- Field name, generic type, scalar/collection shape, raw/tokenized, sortable, facetable and language variance are part of the data contract. Changing any of them needs a reindex and a review of every query and result model that uses the field.
- Built-in fields live in `IndexingConstants.ComputedIndexFields` (for example `NodeIdInt`, `NodeName`, `SearchTitle`, `SearchContent`, `SearchDate`, `Url`, `ItemType`, `TaxonomyTags`); reuse them rather than redeclaring.

## `IIndexingConverter`

```csharp
[Service<SearchOptions>(Lifetime = ServiceLifetime.Singleton, Feature = SearchOptions.Lucene | SearchOptions.Azure)]
public sealed class ProductIndexingConverter : IIndexingConverter
{
    public int Order => 100;                               // higher runs after lower
    public List<string> RunForIndexes() => ["ExternalIndex"]; // empty = every index

    public IEnumerable<IIndexFieldDefinition> GetIndexFieldDefinitions() =>
        [ProductFields.Sku, ProductFields.Name, ProductFields.Tags, ProductFields.Stock];

    public ValueTask AddContentComputedFieldsAsync(
        IPublishedContent? publishedContent, IndexingModel indexingObject,
        IndexingItemContext context, CancellationToken cancellationToken = default)
    {
        if (publishedContent is null) return ValueTask.CompletedTask;
        indexingObject.SetOrOverrideComputedValue(ProductFields.Sku, publishedContent.Value<string>("sku") ?? string.Empty);
        indexingObject.SetComputedValue(ProductFields.Tags, publishedContent.Value<IEnumerable<string>>("tags")?.ToArray() ?? []);
        return ValueTask.CompletedTask;
    }
}
```

`AddMediaComputedFieldsAsync` has a no-op default; override it only for media fields.

- `GetIndexFieldDefinitions()` is the schema contract: declare every field that queries, facets or results use.
- `Order`: converters with the same `Order` run concurrently and must not write the same field; the next `Order` starts after the previous group finishes. A converter that throws is logged and skipped; the item is still indexed.
- `RunForIndexes()` must return a fixed answer: the plan is cached per index name.
- When two converters declare the same field name (case-insensitive), the definition from the lowest `Order` wins; internal converters rank before external ones.
- `SetComputedValue` keeps an existing value; `SetOrOverrideComputedValue` replaces it. Choose by which feature owns the field.
- Vector text is set with `SetVectorContent`, `SetVectorContentPrecise` and `SetVectorContentPreciseManual` (see `azure-and-vectors.md`).
- Taxonomy values use `AddTaxonomyValue` (see `querying.md`).

## Registration and the analyzer

Concrete converters (and `IScoringProfileProvider` implementations) must carry an IL.AttributeBasedDI registration attribute - `[Service]`, `[Service<SearchOptions>(Lifetime, Feature)]`, `[ServiceWithOptions]` or `[Decorator]` - otherwise analyzer `ILUS0001` fails the build. Use `Feature = SearchOptions.Azure` (or `Lucene`) to restrict a converter to one engine.

## External data: `IExternalIndexingConverter`

For records that do not come from `IPublishedContent`. It runs during configured index rebuilds on both engines.

```csharp
[Service]
public sealed class ExternalProducts : IExternalIndexingConverter
{
    public IEnumerable<IIndexFieldDefinition> GetIndexFieldDefinitions() => [ProductFields.Sku];

    public Task<IEnumerable<ExternalIndexingModel>> GetIndexingModelsAsync(
        IndexingItemContext context, CancellationToken cancellationToken = default)
    {
        var model = new ExternalIndexingModel("erp-100", "Product 100");   // nodeKey, nodeName, excludedFromSearch = false
        model.SetComputedValue(ProductFields.Sku, "A-100");
        return Task.FromResult<IEnumerable<ExternalIndexingModel>>([model]);
    }
}
```

- Only fields registered for the target index are indexed (plus `Path`, `Published`, `UmbracoPath`, `UmbracoId`), so declare every value you set.
- Same-`Order` external converters run concurrently; later groups wait. `RunForIndexes()` and `Order` behave as above.
- With vector content set and embeddings enabled, vectors are generated during the rebuild; with background embeddings enabled they are queued instead (see `azure-and-vectors.md`).

## Azure scoring profiles (`IScoringProfileProvider`)

Azure only. Implement it to contribute a scoring profile to the index definitions the package creates (startup, rebuild, synonym publication). Do not decorate `IAzureIndexDefinitionProvider` for this.

```csharp
[Service<SearchOptions>(Lifetime = ServiceLifetime.Singleton, Feature = SearchOptions.Azure)]
public sealed class RelevanceScoringProfileProvider : IScoringProfileProvider
{
    public ScoringProfile? GetScoringProfile(ScoringProfileContext context)
    {
        var title = IndexingConstants.ComputedIndexFields.SearchTitle.FieldName;
        if (!context.HasField(title)) return null;
        return new ScoringProfile("relevance")
        {
            TextWeights = new TextWeights(new Dictionary<string, double> { [title] = 5 })
        };
    }
}
```

- Register as a singleton without scoped dependencies; the method can run concurrently.
- Return `null` to contribute nothing. Reference only fields that exist (`context.HasField`): an empty `RunForIndexes()` also covers other indexes the package provisions, such as the backoffice internal index, and Azure rejects unknown fields.
- `Order` (default 1) sets merge precedence: same-named profiles from several providers merge into one per index, higher `Order` wins per weighted field, functions concatenate by `Order`, and equal orders fall back to registration order (use distinct values). Different names stay separate. A provider returns at most one profile per index.
- Treat returned `ScoringFunction` instances as immutable. Exceptions propagate out of index creation.
- The package never sets a default profile. Select it per query: `parameters.EngineSpecific.AzureRequestOverride = o => o.ScoringProfile = "relevance";` (`Azure.Search.Documents.SearchOptions`, not the `SearchOptions` feature enum).
- A custom `IAzureIndexDefinitionProvider` receives the logical index name as its third argument.
