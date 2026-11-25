# Configuration

## Engine and registration

Pick exactly one engine feature, `SearchOptions.Lucene` or `SearchOptions.Azure`, from deployment needs (not per caller). Optional features are added next to it.

```csharp
builder.AddServiceAttributeBasedDependencyInjection(options =>
{
    options.AddFeature(SearchOptions.Lucene);   // or SearchOptions.Azure
    // Optional: options.AddFeature(SearchOptions.BackofficeDocumentSearch);
    // Optional (separate package): options.AddFeature(SearchOptions.Analytics);
});
```

`builder` is the `WebApplicationBuilder`; the extension comes from the `IL.AttributeBasedDI` package (`using IL.AttributeBasedDI.Extensions;`). There is no `AddUmbracoSearch` extension.

- The package attaches to existing Umbraco/Examine indexes (normally `ExternalIndex`, optionally `InternalIndex`). It never creates Lucene indexes.
- `SearchOptions.BackofficeDocumentSearch` routes only the Umbraco backoffice document search through `InternalIndex`, keeps start-node scoping, and returns nothing without a signed-in backoffice user. Media, member and other entity searches stay on Umbraco's own implementation.

## `SearchSettings` (appsettings section `SearchSettings`)

```json
"SearchSettings": {
  "LicenseToken": "<secret, required for Azure>",
  "DefaultIndexName": "ExternalIndex",
  "Indexes": ["ExternalIndex", "InternalIndex"],
  "PreviewIndexes": ["InternalIndex"],
  "Azure": { "ServiceUrl": "<url>", "ApiKey": "<secret>" }
}
```

Keep tokens and keys in secret storage.

| Setting | Default | Meaning |
| --- | --- | --- |
| `DefaultIndexName` | `ExternalIndex` | Index used when a request omits `IndexName`. |
| `Indexes` | `["ExternalIndex"]` | Existing Umbraco indexes the package may use. In Azure mode also the allow-list for index lifecycle (provisioning, rebuild). Names are compared case-insensitively. |
| `PreviewIndexes` | `[]` | Indexes where soft deletion (preview semantics) applies. |
| `ReadOnly` | `false` | Azure only: the node skips Azure index provisioning, rebuild and document writes. Azure write access also requires a Single or SchedulingPublisher server role. Lucene indexing ignores it. |
| `CustomIndexSuffix` | empty | Lower-cased suffix that separates physical Azure indexes per environment; keep it stable per environment. |
| `EnableBlueGreenIndexing` | `false` | Azure rebuilds build an alternate physical index and swap; see `azure-and-vectors.md`. |
| `DisableSwapDelay` | `false` | Skips the pre-swap wait (about a minute per 1,000 items); dev/test only. |
| `EnableTaxonomyFacetExpansion` | `false` | Hierarchical taxonomy facets (see `querying.md`). |
| `EnableTaxonomyFiltersExpansion` | `false` | Regroups taxonomy `OR` filters by category path; needs facet expansion on. |
| `LicenseToken` | none | Required for Azure only. |
| `EmbeddingsSqlPersistenceConnectionStringName` | none | Connection string name for the embedding cache; when set, pending embedding migrations apply at startup. |
| `Azure.*` / `OpenAi.*` | see `azure-and-vectors.md` | Azure Search connection, vector and batching settings; embedding service settings. |
| `Analytics.*` | see the analytics skill | Only with `SearchOptions.Analytics`. |

## Rules that bite

- The license is checked only with the Azure feature: an invalid or missing `LicenseToken` makes initialization throw `InvalidOperationException("License token is not valid.")`. Lucene needs no token.
- With Azure, host startup fails when `Azure.ServiceUrl` or `Azure.ApiKey` is missing or the URL is not absolute. When `Azure.UseHybridSearch` is on, startup also fails without `OpenAi.ApiKey`, an absolute `OpenAi.ServiceUrl`, or `OpenAi.EmbeddingsDeploymentName`. Values the runtime clamps (batch sizes, chunk bounds, concurrency) are not validated.
- A misspelled or missing configured index can surface as a logged error and empty results instead of a startup failure; cover this in operational tests.
- `ReadOnly`, the server role and the `Indexes` allow-list are hard safety constraints for Azure writes. Do not work around them.
- Vary settings per environment with normal ASP.NET configuration layering, not code branches.
