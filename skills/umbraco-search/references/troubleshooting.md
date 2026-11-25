# Troubleshooting and definition of done

## Diagnose in this order

1. The registered feature matches the intended engine (`SearchOptions.Lucene` or `Azure`, exactly one).
2. Startup log: Azure settings and license errors (`License token is not valid`, missing `Azure.*`/`OpenAi.*`) fail the host before any search.
3. `DefaultIndexName`, the request `IndexName` and `Indexes` all name existing Umbraco indexes exactly. A wrong name usually means a logged error and empty results.
4. The content is published, actually indexed, and your converter is discovered (it carries a registration attribute; analyzer `ILUS0001` is clean).
5. Field declaration versus usage: scalar vs collection helper, the generic type, and the facetable/sortable/raw/multi-language flags.
6. Language: the requested culture resolves to a configured language, and values exist for it (the default language reads the unsuffixed field).
7. Lucene full-text returns nothing: no field is declared `isSearchable: true` and no `SearchFields` were supplied.
8. Taxonomy results are empty: filtering tags where category paths were meant (or the reverse), or the expansion settings do not match what the UI expects.
9. Azure: `ReadOnly`, the `Indexes` allow-list, credentials, server role, suffix and blue/green settings, and the batch/rebuild logs.
10. Hybrid: both `UseHybridSearch` and the `OpenAi` settings are present; then tell missing vector text apart from background-embedding delay (`azure-and-vectors.md`).
11. Endpoint or MCP problems: check the application's own mapping overrides, authorization, limits and request validation before suspecting the package.

## Definition of done for a consumer feature

- The chosen engine supports every feature the UI advertises.
- The right feature is registered and configuration is valid and kept in secret storage.
- Custom fields have typed definitions, a registered converter, correct population, and queries that match their shape.
- Default-index and explicit-index paths are both tested.
- Default and non-default cultures are tested wherever language-aware behavior is used.
- Filters, facets, sorting and the result model are exercised against representative content.
- With Azure, lifecycle constraints (allow-list, `ReadOnly`, blue/green) and vector consistency are covered in operations and tests.
- Public HTTP/MCP surfaces enforce authorization, rate and paging limits, and input constraints in the application.
- Examples and configuration in the application match the installed package version.
