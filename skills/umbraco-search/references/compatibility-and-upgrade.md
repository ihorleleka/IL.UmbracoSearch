# Compatibility and upgrade

Applies to `IL.UmbracoSearch` 17.8.0.1 and later, including the 18.0.x releases. One package line serves both Umbraco 17 and 18 through two target frameworks.

## Pairing (choose by runtime)

| Application runtime | Umbraco CMS |
| --- | --- |
| .NET 10 | 17.8.1 or later, below 18 |
| .NET 11 | 18.3.1 or later, below 19 |

- Umbraco 13-16 and .NET 8-9 are not supported.
- Do not pair Umbraco 17 with the .NET 11 asset or Umbraco 18 with the .NET 10 asset.
- Both targets use EF Core 10 (`[10.0.15, 11.0.0)`) and DI abstractions 10 (`[10.0.15, 11.0.0)`); keep the application on those lines.

## Dependency versions

- A bare NuGet version means a minimum. The package declares `[minimum, next-major)` ranges, so the application's own `PackageReference` or central package version can pick any version inside the range, subject to the rest of its graph.
- `IL.AttributeBasedDI` is the exception: it declares only a minimum (2.11.1.1, no upper bound).
- The Azure preview dependencies are required and must not be downgraded to older stable releases: `Azure.AI.OpenAI` `[2.9.0-beta.1, 3.0.0)` and `Azure.Search.Documents` `[12.1.0-beta.2, 13.0.0)`.
- Other runtime ranges include `ModelContextProtocol` `[2.4.0, 3.0.0)`, the two `Microsoft.ML.Tokenizers` packages `[3.0.0, 4.0.0)`, and `System.IdentityModel.Tokens.Jwt` `[8.23.0, 9.0.0)`.

## Breaking changes in the 17.8 line (check an upgraded application)

- Indexing extension points are asynchronous:
  - `IIndexingConverter.AddContentComputedFieldsAsync` and `AddMediaComputedFieldsAsync` return `ValueTask`.
  - `IExternalIndexingConverter.GetIndexingModelsAsync` returns `Task<IEnumerable<ExternalIndexingModel>>`.
  - Converters with the same `Order` run concurrently; order groups run one after another.
  - Direct callers use `AddComputedFieldsAsync`; the synchronous `AddComputedFields` is removed.
- A custom `ILanguageResolutionService` implements `GetAllLanguagesAsync`, `GetDefaultLanguageIsoCodeAsync` and `ResolveLanguageIsoCodeAsync`. `ISearchRequestNormalizer` exposes `NormalizeAsync` overloads. Await them and forward cancellation tokens; no synchronous versions remain.
- `IndexFieldDefinition` constructors are not public. Create field definitions with `IndexFieldDefinitionFactory`.
- `IAzureIndexDefinitionProvider.BuildSearchIndexDefinition` takes the logical index name as a third argument (`fieldDefinitions, physicalName, indexName`). Custom implementations and decorators must be updated.

## Cancellation

- Cancellation propagates; a cancelled request is never converted into an empty successful result.
- Umbraco's language API has no cancellation parameter, so cancellation stops the caller's wait, not the underlying lookup.
- Azure results are paged asynchronously and stop at `Take` (a zero-hit facet request enumerates no hits). The Azure SDK may not cancel a continuation request that is already in flight; cancellation is checked again as each hit arrives.
