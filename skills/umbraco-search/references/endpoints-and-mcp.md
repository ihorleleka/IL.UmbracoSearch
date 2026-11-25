# Minimal API endpoints and MCP tools

Both surfaces are opt-in. The application that maps them owns authentication, authorization, rate limiting, caching, tenancy and input validation.

## Minimal API endpoints

```csharp
app.UseSearch();             // POST /api/search, JSON body = SearchRequest
app.UseSuggestionsSearch();  // POST /api/search/suggest, JSON body = SuggestionsSearchRequest
```

Defaults: `Path` `/api/search` and `/api/search/suggest`; the request types are `SearchRequest` and `SuggestionsSearchRequest`, mapped onto `SearchParameters` and `SuggestionSettings`. Unknown field names in the request are ignored; that is not validation.

```csharp
app.UseSearch(options =>
{
    options.Path = "/api/site-search";
    options.SearchParametersOverride = static (httpContext, request, parameters, ct) =>
    {
        parameters.Aliases = ["contentPage"];
        parameters.Take = Math.Min(parameters.Take, 50);          // cap public input
        return ValueTask.FromResult(parameters);
    };
});

app.UseSuggestionsSearch(options =>
{
    options.SuggestionSettingsOverride = static (httpContext, request, settings, ct) =>
    {
        settings.Size = Math.Min(settings.Size, 5);
        return ValueTask.FromResult(settings);
    };
});

app.UseSearch<ProductSearchResult>();             // same endpoint with your own result model
app.UseSuggestionsSearch<ProductSearchResult>();
```

- Options also expose `GroupName`, `Tags`, `EndpointName` and a `ResponseFactory` to shape the HTTP result.
- Both calls return a `RouteHandlerBuilder`, so you can chain `.RequireAuthorization(...)`, `.RequireRateLimiting(...)` and similar.
- When analytics is enabled and `SuggestionsSearchRequest.Analytics` is supplied, the suggestion response is the `SuggestionResult<SuggestionResultItem<T>>` envelope with analytics metadata, `TotalItemCount` and `Items`.
- Suggestion requests are normalized as described in `querying.md` (trimmed term, `Size` at least 1, plain highlight tag).
- Always override `Take`/`Size`, restrict `Aliases`/`IndexName` where relevant, and validate untrusted filters in your own code.

## MCP tools

```csharp
builder.Services.AddUmbracoSearchMcpTools().WithHttpTransport();   // WithHttpTransport needs the MCP ASP.NET Core package
// or add the tools to an existing MCP server builder:
builder.Services.AddMcpServer().WithHttpTransport().WithUmbracoSearchTools();
app.MapMcp("/searchMcp");
```

The package does not choose a transport; configure it in the application.

Built-in tools:

- `fetchAvailableIndexes` - returns `DefaultIndexName`, `Indexes` and `PreviewIndexes`.
- `search(indexName?, q?, top_k = 5)` - `top_k` is clamped to 1..50; `q` becomes the full-text query and the hybrid flag follows `Azure.UseHybridSearch`. It returns `id`, `title`, `description`, `url`, `languageIsoCode` and `additionalData`.

Customize the default query or the returned item:

```csharp
builder.Services.AddUmbracoSearchMcpTools(options =>
{
    options.SearchParametersOverride = static (request, parameters, ct) =>
    {
        parameters.Aliases = ["contentPage"];
        return ValueTask.FromResult(parameters);
    };
    options.ResultMapper = static (context, ct) => ValueTask.FromResult(new McpSearchItem
    {
        Id = context.Item.Id,
        Title = context.Item.SearchTitle ?? context.Item.NodeName,
        Description = context.Item.SearchDescription,
        Url = context.Item.Url,
        AdditionalData = new Dictionary<string, object?> { ["requestedIndex"] = context.Request.IndexName }
    });
}).WithHttpTransport();
```

- `AddUmbracoSearchMcpTools<TSearchItem>(...)` does the same with your own result model (`context.Item` is then a `TSearchItem`, so `ValueFor(...)` works).
- For a different contract, replace or decorate `IUmbracoSearchMcpService`; both tools call it.
- Authentication, authorization, client identity, rate limiting and which index names are exposed belong to the application and its hosting layer. An ASP.NET Core fixed-window limiter on the `MapMcp` route (`.RequireRateLimiting("name")`) is the usual approach.
