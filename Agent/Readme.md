# IL.UmbracoSearch - guide for AI coding agents

The agent guide for the `IL.UmbracoSearch` NuGet packages is now delivered as two skills. Each skill is a folder with a short `SKILL.md` (the non-negotiable contracts and a decision guide) and a `references/` folder with one page per topic.

| Skill | Use it when |
| --- | --- |
| [`umbraco-search`](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search/SKILL.md) | Building, extending or debugging search with `IL.UmbracoSearch`: queries, filters, facets, custom indexing, Azure/hybrid search, endpoints, MCP, configuration. |
| [`umbraco-search-analytics`](https://github.com/ihorleleka/IL.UmbracoSearch/blob/main/skills/umbraco-search-analytics/SKILL.md) | The optional `IL.UmbracoSearch.Analytics` package: Search Insights, click tracking, backoffice reports, synonyms. |

## How to use them

- If your agent supports skills, install the two folders from the [`skills/`](https://github.com/ihorleleka/IL.UmbracoSearch/tree/main/skills) directory (for example into `.claude/skills/` or `.agents/skills/`).
- If it does not, read the relevant `SKILL.md` first, then open only the reference page for the task at hand. Do not load every reference.
- The skills apply to `IL.UmbracoSearch` 17.8.0.1 and later, including 18.0.x; the source of the package is not needed.
