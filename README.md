[![NuGet version (IL.AttributeBasedDI)](https://img.shields.io/nuget/v/IL.UmbracoSearch.svg?style=flat-square)](https://www.nuget.org/packages/IL.UmbracoSearch/)
# IL.UmbracoSearch

Search for Umbraco 17 and 18 on top of your existing Examine indexes, with two interchangeable engines: **Lucene** (Examine) and **Azure AI Search**. Typed filters, facets, sorting, boosts and suggestions; custom indexing through `IIndexingConverter`; optional hybrid and chunk-level vector search (Azure); optional minimal API endpoints and MCP tools for AI clients.

> [!IMPORTANT]
> **Compatibility.** Pair the package with the Umbraco version that matches your runtime. Umbraco 13-16 and .NET 8-9 are not supported.
>
> | Application runtime | Umbraco CMS |
> | --- | --- |
> | .NET 10 | 17.8.1 or later, below 18 |
> | .NET 11 | 18.3.1 or later, below 19 |
>
> Both targets use EF Core 10. Versions from 17.8.0.1 on have asynchronous indexing extension points (`IIndexingConverter` computed-field methods return `ValueTask`, `IExternalIndexingConverter.GetIndexingModelsAsync` returns `Task`); see the [compatibility and upgrade notes](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/references/compatibility-and-upgrade.md).

## Features

- Full-text search with field targeting, boosts, wildcards and exact-phrase boosting.
- Typed filters (`FilterFor`, `FilterForCollectionField`, ranges, `And`/`Or`/`Not`), sorting, facets, suggestions, subtree search and language-aware fields.
- Hierarchical taxonomy facets and filters.
- Custom fields through `IIndexingConverter` and `IExternalIndexingConverter`; Roslyn analyzers that catch missing registrations and wrong filter helpers at build time.
- Result field projections: a source generator makes a result model fetch only the fields it reads.
- Azure only: hybrid keyword/vector search, chunk-level vectors, scoring profiles, blue/green index rebuilds.
- Optional: ready-made minimal API search endpoints, MCP search tools, Umbraco backoffice document search, and [Search Insights](https://www.nuget.org/packages/IL.UmbracoSearch.Analytics) analytics.

The package attaches to **existing** Umbraco indexes (normally `ExternalIndex`, optionally `InternalIndex`); it never creates Lucene indexes.

## Install

```bash
dotnet add package IL.UmbracoSearch
```

## Register an engine

Choose exactly one of `SearchOptions.Lucene` or `SearchOptions.Azure` in `Program.cs`:

```csharp
builder.AddServiceAttributeBasedDependencyInjection(options =>
{
    options.AddFeature(SearchOptions.Lucene);   // or SearchOptions.Azure
});
```

The extension comes from `IL.AttributeBasedDI` (`using IL.AttributeBasedDI.Extensions;`).

## Configure

Add a `SearchSettings` section to `appsettings.json` and keep secrets in secret storage:

```json
"SearchSettings": {
  "DefaultIndexName": "ExternalIndex",
  "Indexes": ["ExternalIndex", "InternalIndex"],
  "PreviewIndexes": ["InternalIndex"]
}
```

For Azure also set `LicenseToken` and `Azure:ServiceUrl` / `Azure:ApiKey` (and `OpenAi:*` for hybrid search). Azure requires a valid license token and valid Azure settings at startup; Lucene needs no license. Every setting and its default is in the [configuration reference](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/references/configuration.md).

## Your first search

Inject `ISearchService` and call `SearchAsync`:

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

`results.Items` holds the hits, `results.TotalItemCount` the total and `results.Facets` any requested facets. `Take` defaults to `int.MaxValue`, so always set it, especially for public input.

## Learn more

Each topic has one reference page:

| Topic | Page |
| --- | --- |
| Registration and every `SearchSettings` option | [configuration](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/references/configuration.md) |
| `SearchParameters`, filters, facets, suggestions, taxonomy, languages, result models and projections | [querying](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/references/querying.md) |
| Custom fields, converters, external data, scoring profiles | [indexing](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/references/indexing.md) |
| Azure lifecycle, blue/green rebuilds, hybrid and chunk vectors, background embeddings | [azure-and-vectors](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/references/azure-and-vectors.md) |
| Search endpoints and MCP tools | [endpoints-and-mcp](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/references/endpoints-and-mcp.md) |
| Diagnosis order and definition of done | [troubleshooting](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/references/troubleshooting.md) |
| Framework pairing, dependency ranges, breaking changes | [compatibility and upgrade](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/references/compatibility-and-upgrade.md) |

## Optional features

- **Backoffice document search:** add `SearchOptions.BackofficeDocumentSearch` next to your engine to route only the Umbraco backoffice document search through `InternalIndex`. Start-node rules are kept; media, member and other entity searches keep Umbraco's own implementation.
- **Search Insights:** install `IL.UmbracoSearch.Analytics` and add `SearchOptions.Analytics` next to exactly one engine; see its README and the [analytics skill](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search-analytics/SKILL.md).

## For AI coding agents

Install or point your agent at the [`umbraco-search` skill](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/SKILL.md) (and [`umbraco-search-analytics`](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search-analytics/SKILL.md) when the analytics package is used). Each skill is a short contract summary plus the reference pages above, so an agent can build search features without access to the package source.
