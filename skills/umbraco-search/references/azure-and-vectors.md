# Azure, hybrid and chunk vectors

Choose Azure only when its operational needs exist: service credentials, a managed-index allow-list, lifecycle ownership and monitoring. Lucene stays the simpler default; hybrid/vector behavior is Azure-only.

## Azure settings (`SearchSettings:Azure`, `SearchSettings:OpenAi`)

| Setting | Default | Meaning |
| --- | --- | --- |
| `Azure.ServiceUrl`, `Azure.ApiKey` | none | Required (absolute URL). Missing values do not stop the host: an error is logged at startup, searches return no results and indexing is skipped. |
| `Azure.UseHybridSearch` | `false` | Keyword plus vector search. Needs `OpenAi.*`. |
| `Azure.UseChunkedVectorSearch` | `false` | Also store chunk-level vectors (multi-vector field). |
| `Azure.ChunkedVectorMinTokens` / `MaxTokens` / `OverlapTokens` | 2000 / 3000 / 200 | Chunk size policy. |
| `Azure.ChunkedVectorEmbeddingMaxConcurrency` | 4 (clamped 1-16) | Concurrent embedding requests across the whole application. |
| `Azure.BackgroundEmbeddingsProcessing` | `false` | Generate vectors in a durable background queue (below). |
| `Azure.BackgroundEmbeddingsMaxConcurrency` | 4 | Vector jobs processed at once. |
| `Azure.QueryEmbeddingTimeoutMilliseconds` | 5000 | Longest a hybrid search waits for the query embedding. On timeout or failure the search continues as keyword-only (logged warning). `0` or negative waits indefinitely. |
| `Azure.DocumentBatchMaxActions` / `MaxPayloadBytes` / `FlushIntervalMilliseconds` | 100 / 14 MiB (14680064) / 250 | Bounds of each Azure write batch. |
| `Azure.DocumentBatchQueueCapacity` | 2000 | Accepted-but-unsent writes; indexing waits when full. |
| `Azure.DocumentBatchEnqueueTimeoutMilliseconds` | 30000 | Longest indexing waits for queue space (and a delete waits for queued writes to drain) while Azure is not draining. After that the document is dropped with an error log (a rebuild restores it) and a delete proceeds. `0` or negative waits forever. |
| `Azure.DocumentBatchMaxRetries` | 3 | Retries for transient failures (408, 409, 429, 5xx). |
| `OpenAi.ServiceUrl`, `ApiKey`, `EmbeddingsDeploymentName` | none | Embedding service; required when hybrid is on (for example deployment `text-embedding-3-large`). |

Azure document writes are batched and eventually consistent; a rebuild waits for its queued writes to drain, and the host drains them on shutdown within a bounded time.

## Index lifecycle

- The package provisions and rebuilds Azure indexes only for names in `SearchSettings.Indexes`. `CustomIndexSuffix` and blue/green naming are internal: never derive or swap physical index names in application code.
- With `EnableBlueGreenIndexing`, a rebuild writes to the alternate colour, swaps active search/indexing after success (waiting about a minute per 1,000 items unless `DisableSwapDelay`), then deletes the inactive index. Have a recovery and monitoring plan before enabling it. Deleted content is removed from both the live and the new index while a swap is in progress.
- `ReadOnly` nodes and non-writer server roles never run write or rebuild flows.
- For preview indexes (`PreviewIndexes`), soft deletion keeps the exclusion flag correct for both the base and language-specific fields.

## Hybrid vectors

Enable only after ordinary search works.

1. Register `SearchOptions.Azure`, set the Azure and `OpenAi` settings, and `Azure.UseHybridSearch = true`.
2. In an indexing converter, set the source text: `indexingObject.SetVectorContent(text)`.
3. Search with `new FullTextSearch(query, useHybridSearch: true)`. Optional knobs: `VectorBasedResultsWeight`, `VectorSimilarityThreshold`, `CustomVector` (find items similar to an existing vector).

Average vectors: content under 8,000 tokens is embedded in one request; larger content is chunked and the chunk embeddings are averaged. Unchanged content is not re-embedded (results are cached by a content hash). Hybrid requests drop score ordering because Azure rejects it together with vector queries. If the query embedding fails or times out, the request still returns keyword results (without the vector half) and logs a warning.

## Chunk (granular) vectors

Set `Azure.UseChunkedVectorSearch = true`, then optionally give different chunk text or bounds per item:

```csharp
item.SetVectorContent("short searchable summary");                       // average vector
item.SetVectorContentPrecise("long form content for chunk matching");    // chunk vectors, settings-driven sizes
item.SetVectorContentPreciseManual(content, minTokens: 600, maxTokens: 900, overlapTokens: 150);
```

- Without `SetVectorContentPrecise`, chunk vectors reuse the `SetVectorContent` text with settings-based sizes. The two texts may differ.
- Azure allows at most 100 vectors per item. If the configured sizes would exceed that, the chunk size is increased automatically and a warning containing "vectors count exceeds quota (100)" is logged.
- Chunk-size guidance: balanced default 2000/3000/200; higher precision for short sections or FAQs 600/900/100; very granular passage recall 400/500/50 (more vectors, more cost); long-form efficiency 2500/4000/200.

## Background embeddings

With `BackgroundEmbeddingsProcessing = true` (and hybrid on), indexing computes neither vector inline: it queues a durable job and the document is searchable before its vector fields exist. Results are therefore eventually consistent; a just-published item may not take part in vector matching yet. Failed jobs retry with backoff, then are dropped. If queueing a job fails (embeddings database unavailable) the document is still indexed and uploaded without vectors, with an error logged, until the next rebuild.

When `SearchSettings:EmbeddingsSqlPersistenceConnectionStringName` names a connection string, the embedding cache and job tables live in that database and pending embedding migrations are applied automatically at startup (the host waits at most 30 s for them, then continues while they finish in the background); without it an in-memory store is used (lost on restart, not shared between servers). Do not delete embedding data to fix a relevance issue.
