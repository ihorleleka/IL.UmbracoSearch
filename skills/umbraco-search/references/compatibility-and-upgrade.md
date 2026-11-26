# Compatibility and upgrade

Written against `IL.UmbracoSearch` 18.0.0.5, which serves both Umbraco 17 and 18 through two target frameworks. Earlier 17.8.x builds differ; check the installed version before relying on the sections below.

## What differs by installed version

Read the version from the application's `*.csproj`/`Directory.Packages.props` (or the restored `project.assets.json`), then use this table.

| Installed | Target frameworks | Umbraco floor in package metadata | Pinned dependency minimums | Analyzers shipped |
| --- | --- | --- | --- | --- |
| 17.8.1.x - 17.8.5.x | `net10.0` only | `Umbraco.Cms.Web.Common [17.0.0, 18.0.0)` | exact versions, no upper bounds: `IL.AttributeBasedDI` 2.9.1.1 to 2.10.1.1, `Azure.Search.Documents` 12.1.0-beta.1, `Microsoft.ML.Tokenizers` 2.0.0, `ModelContextProtocol` 2.2.0 | registration (`ILUS0001`), filter shape (`ILUS0002` and `ILUS0003`, the latter reporting `FilterForCollectionField` on a scalar field) and the projection generator |
| 18.0.0.x up to 18.0.0.5 | `net10.0` and `net11.0` | 17.8.1+ on .NET 10, 18.3.1+ on .NET 11 | the ranges listed below | the same, plus code fixes from 18.0.0.5 |
| releases after 18.0.0.5 | as 18.0.0.5 | as 18.0.0.5 | as 18.0.0.5 | additionally `ILUS0004` (untyped `FilterFor` on a collection field), `ILUS0005` (projection on a non-partial class), `ILUS0006` (default-member signature mismatch), all warnings (`ILUS0006` also covers abstract converter base classes); `ILUS0003` is removed (the compiler already rejects that call) and `ILUS0002` no longer flags dictionary-typed fields; `FacetFor`/`FacetOptionsFor` helpers |

The package major (17 or 18) does not track the Umbraco major: 18.0.x runs on Umbraco 17.8.1+ with .NET 10. Verify any API named in the other references against the installed version if it is not in the 17.8 core behavior (async converters, `IndexFieldDefinitionFactory`).

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
