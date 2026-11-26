# Book vectorization: what embeddings in PostgreSQL would add

- **Question:** what value would vectorizing books, and storing the vectors in PostgreSQL, add to a Lorekeeper implementation?
- **Sources:** the Lorekeeper reference implementation at commit `9c7ebed` (generation, entity merging and Ask), the [pgvector] documentation, embedding model cards, and research on inverting embeddings.
- **Method:** reading the reference pipeline; measuring the Dracula example with the reference's EPUB parser; benchmarking pgvector 0.8.7 on PostgreSQL 18.7 in Docker.
- **Date:** 2026-11-23.

## Summary

- **Verdict:** do not vectorize books at first.
  - Vectorize index items instead: entities, facets, events and recaps.
  - Keep book vectors for later, inside the index builder, as a quality aid.
- **The reference needs no embeddings.** Name rules and LLM judgements already identify entities well. A fresh Dracula build leaves no duplicate entities, and on the Empire of the Vampire series 137 candidate pairs narrow to 17 for review.
- **The most valuable use is Ask.** Ranking the items of the gated view by similarity:
  - replaces a 120,000-character roster prompt for cloud models;
  - gives local models a semantic pick, which they lack today;
  - is spoiler-safe by construction, because only visible items are ranked.
- **Book vectors are book text.** Embeddings can be inverted (vec2text recovers 92% of 32-token inputs exactly). Stored block vectors need the same protection as the book, and must never travel in indexes, share bundles or registries.
- **pgvector needs no approximate index for per-book search.** On 100,800 rows, an exact scan filtered by position returned the top 10 in 0.7–2.0 ms. HNSW under the same filter returned 1 row of 10 by default.
- **Storage is small:** about 3.3 MB per novel as `halfvec(768)`.

## 1. How the reference works without embeddings

- **Dependencies.** The reference's only runtime dependency is `jsonschema`.
- **Generation** ([`generate.py`][generate.py]):
  - The model sees one part of the book, the story-so-far recap, and what is already known about the entities named in that part.
  - It also sees the last 150 known entities (`MAX_KNOWN_LIST`).
  - Earlier passages are never retrieved; context is built from the index under construction.
- **Duplicate entities** ([`merge.py`][merge.py]), in three steps:
  1. Name rules propose candidates.
  1. An LLM proposes pairs "described alike".
  1. An LLM judge decides each pair, reading the book paragraphs where the names appear.

  The reference's notes ([`CLAUDE.md`][CLAUDE.md] §8) report "137 listed pairs to 17 for review" on the Empire of the Vampire series, and that "a fresh Dracula build leaves none".
- **Ask** ([`ask.py`][ask.py]) answers "from the notes of a gated view, never from the book's text".
  - It picks entries by name rules and word matches.
  - With a cloud model, an LLM also picks from a roster of every visible entry, up to 120,000 characters; that costs about 0.6 cents per question on Haiku.
  - Local models get no roster pick, because their context cannot hold it.
- **Scale**, for Dracula:
  - 2,016 blocks and 161,861 words, measured with the reference's parser.
  - The bundled index has 471 anchors, 137 entities, 360 facets, 349 events, 51 recaps and 32 relations: about 930 items and 160,000 characters.

## 2. Value by use case

| Rank | Use case | Vectors of | Value | Cost | When |
| --- | --- | --- | --- | --- | --- |
| 1 | Ask: a semantic pick over the gated view | Index items | High | Low | First |
| 2 | Candidate pairs for the duplicate-entity judge | Index items | Medium-high | Low | First, or soon after |
| 3 | Choosing which known entities to show the model in large series, beyond the reference's 150 | Index items | Medium | Low | After a benchmark |
| 4 | Hindsight triage: a claim whose best support lies after its `from` goes to review | Book and index | Medium-high | Low-medium | Later, builder only |
| 5 | Earlier evidence for claims the fact-checker marked `unsure` | Book | Medium | Low-medium | Later, builder only |
| 6 | Earlier blocks as context for small models | Book | Uncertain | Medium | Experiment |
| 7 | Ask answered from the book's text | Book | Medium | Medium | Later, opt-in, self-hosted only |
| 8 | Finding similar indexes across a registry | Index items | Low-medium | Low | Later, after lexical signals |
| 9 | Aligning the paragraphs and entities of translations | Book (multilingual) and index | Medium | High | With translation support |

Never:

- book vectors in indexes, share bundles or registries;
- moving a `from` earlier automatically because retrieval found earlier support;
- resolving anchors semantically in reader apps. Lexical alignment, or a second fingerprint (SPEC §11), is safer.

## 3. Risks

### 3.1 Embeddings are partly invertible

- [vec2text] recovers 92% of 32-token inputs exactly, and reaches BLEU 97.3 against a black-box encoder. On a clinical data set it recovers 94% of first names and 95% of last names.
- [ALGEN] attacks black-box encoders with about 1,000 samples, transfers across domains and languages, and found none of the defences it tested effective.

Whether embeddings of a copyrighted book count as a copy is unsettled. Given these results, the safe default is to treat stored block vectors exactly like the text: keep them where the book is kept, and never share them.

### 3.2 Retrieval can leak spoilers

Three mechanisms leak later content into earlier positions:

- **Contextual embeddings.** Prepending document-level context to each chunk puts later context into the vectors of early blocks.
- **Windows.** A window of several blocks that is gated on its first block reveals the blocks after it.
- **Late filters.** HNSW applies filters after ranking, so it ranks invisible rows; a forgotten filter leaks outright.

The safe pattern is to filter before ranking, with one context-free vector per block, and to apply the §6.3 eight-word check to any generated answer.

### 3.3 Silent truncation and drift

- Ollama's embedding endpoint truncates inputs longer than the model's context by default (`truncate: true`). With `mxbai-embed-large` (512 tokens), about 40 of Dracula's blocks would be cut without any error.
- A new model, or a new version of one, changes every vector. Record the model with the vectors (provider, name, digest, dimensions, prefixes, token limit), and re-embed when it changes.

## 4. PostgreSQL and pgvector

- pgvector 0.8.7 (2026-10-01) is the current release; 0.9.0 is unreleased.
- Exact search is the default and has perfect recall. Approximate indexes trade recall for speed ([pgvector][pgvector]).
- With approximate indexes, filtering happens after the index scan. In the documentation's words, "If a condition matches 10% of rows, with HNSW and the default `hnsw.ef_search` of 40, only 4 rows will match on average" ([filtering][pgvector filtering]). Iterative index scans (0.8.0 and later) keep scanning until enough rows match.

**Setup.**

- Docker image `pgvector/pgvector:pg18`, running pgvector 0.8.7 on PostgreSQL 18.7.
- Random `halfvec(768)` vectors, a worst case for approximate recall.
- 50 synthetic books of 2,016 blocks each (100,800 rows), with primary key `(book_id, seq)`.
- Query: the top 10 blocks of one book with `seq <= 500`, about 0.5% of the rows.

**Query results:**

| Plan | Rows returned (of 10) | Time | Notes |
| --- | --- | --- | --- |
| Exact scan through the primary key, no approximate index | 10 | 0.70 ms | Filters before ranking; exact by definition. |
| Exact scan of a whole book (2,016 rows) | 10 | 2.0 ms | Same plan. |
| HNSW, default `ef_search` of 40 | 1 | 4.0 ms | 39 rows removed by the filter. |
| HNSW with `iterative_scan = strict_order` | 10 | 61.6 ms | |
| HNSW with `iterative_scan = relaxed_order` | 10 | 24.7 ms | Only 5 of the exact top 10. |

**Storage**, for Dracula (2,016 rows, including TOAST and the primary key):

| Type | Bytes per vector | Total |
| --- | --- | --- |
| `halfvec(768)` | 1,544 | 3.3 MB |
| `vector(768)` | 3,076 | 8.4 MB, stored out of line |
| `bit(768)`, binary-quantised | 101 | 0.38 MB |

Extrapolated, 100 novels take about 0.32 GB without an approximate index.

**Conclusions:**

- **Per-book search:** use no approximate index. A B-tree on `(…, seq)` with an exact scan was 35 to 90 times faster than iterative HNSW at this selectivity, and it never ranks rows past the position.
- **HNSW:** worth having only for registry-wide searches without a selective filter. There, use `relaxed_order` iterative scans and partial indexes per item kind.
- **IVFFlat:** it is trained on the data present when the index is built, so it suits incrementally loaded data poorly.

A sketch of the tables. Vectors are a cache, keyed by the model that produced them:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE embedding_model (
  id           smallint PRIMARY KEY,
  provider     text     NOT NULL,              -- 'ollama', 'openai', ...
  name         text     NOT NULL,              -- 'embeddinggemma:300m'
  digest       text     NOT NULL,              -- model digest or API model version
  dims         int      NOT NULL,
  doc_prefix   text,                           -- task prompt for documents, if the model needs one
  query_prefix text,                           -- task prompt for queries
  max_tokens   int      NOT NULL,              -- longer inputs are split, never truncated
  UNIQUE (provider, name, digest, dims)
);

-- Index items: licensed CC0 or CC-BY, so safe on any server or registry.
CREATE TABLE index_item_embedding (
  model_id     smallint NOT NULL REFERENCES embedding_model (id),
  index_digest text     NOT NULL,              -- digest of the index version
  item_id      text     NOT NULL,              -- entity, facet, event, recap or relation id
  item_kind    text     NOT NULL,
  book_key     text,                           -- series book, for visibility (SPEC §7.2)
  from_ordinal int      NOT NULL,              -- ordinal of the item's `from` anchor
  embedding    halfvec(768) NOT NULL,
  PRIMARY KEY (model_id, index_digest, item_id)
);

-- Builder only, never shared: one context-free vector per block of an edition the server holds.
CREATE TABLE block_embedding (
  model_id   smallint NOT NULL REFERENCES embedding_model (id),
  text_hash  char(64) NOT NULL,                -- SPEC §3.4 text hash
  seq        int      NOT NULL,                -- SPEC §3.1 block seq
  embedding  halfvec(768) NOT NULL,
  PRIMARY KEY (model_id, text_hash, seq)       -- the gating index: filter by seq, then rank exactly
);
```

## 5. .NET stack and embedding models

- **Storage:** `Pgvector.EntityFrameworkCore` 0.3.0, which supports EF Core 9 to 11, with Npgsql's EF Core provider. Map `HalfVector` properties to `halfvec(768)`.
- **Generation:** `IEmbeddingGenerator<string, Embedding<float>>` from Microsoft.Extensions.AI 11.x, backed by OllamaSharp 5.5.2 or an OpenAI-compatible client ([IEmbeddingGenerator]).
- **Store abstraction:** add `CommunityToolkit.VectorData.PgVector` 1.0.2 only if a store-agnostic seam is wanted; it expects to own its tables, which competes with EF Core migrations. The Semantic Kernel PostgreSQL connector is deprecated.
- **No database at all:** for search within one index, about a thousand items, [`TensorPrimitives.CosineSimilarity`][TensorPrimitives] over the already-gated view is enough.

| Model | Size | Dimensions | Context in Ollama | Languages | Notes |
| --- | --- | --- | --- | --- | --- |
| [`embeddinggemma`][embeddinggemma] | 300M parameters | 768; can be truncated to 512, 256 or 128 | 2K | 100+ | Default choice. MTEB Multilingual v2: 61.15. |
| [`qwen3-embedding:0.6b`][qwen3-embedding] | 0.6B parameters | 1024 | 32K | 100+ | Step up if a benchmark shows a gain. MTEB Multilingual: 64.33. |
| [`bge-m3`][bge-m3] | 567M parameters | 1024 | 8K | 100+ | Cross-lingual alignment; bitext mining 79.11. |
| [`nomic-embed-text`][nomic-embed-text] | 137M parameters | 768 | 2K | English | Requires `search_document:` and `search_query:` prefixes. |
| [`mxbai-embed-large`][mxbai-embed-large] | 335M parameters | 1024 | 512 | English | Avoid: silently truncates long blocks. |

> [!NOTE]
> The quality scores come from different MTEB versions and are not comparable. None measures fiction retrieval or spoiler-sensitive questions, so choose with a benchmark on Lorekeeper data, such as the Dracula index.

## 6. Recommendation

1. **First:** an `IEmbeddingGenerator` seam, and index-item search for Ask and for duplicate-entity candidates.
1. **Later, measured on a benchmark:** builder-side book vectors, for hindsight triage and for earlier evidence behind `unsure` claims.
1. **Ask from book text:** only if the product allows it, and then self-hosted and opt-in.

## 7. Open questions

1. **Ask from book text:** should Ask ever answer from the book's text, or keep the reference's promise of "never from the book's text"? This decides whether block vectors exist beyond quality checks.
1. **Recording the embedder:** the index format has no field for it. Encode it in `provenance.generator.promptSet`, keep it in a tool sidecar, or propose a specification field?
1. **Benchmark:** which corpus and which metrics will judge use cases 3 to 6? Candidates:
   - recall of duplicate pairs;
   - the rate of "the index doesn't say" answers;
   - claims converted from `unsure` to `supported`;
   - the precision of hindsight flags.
1. **Deployment shape:** a single-user self-hosted server, a hosted multi-user service, or a registry? This decides the copyright exposure of storing anything derived from book text.
1. **Translations (SPEC §11):** if translated works are linked, multilingual embeddings (`bge-m3`, Qwen3) are needed rather than English-only models.
1. **Search in reader apps:** semantic search on the device would mean shipping item vectors with indexes and a model on the device. Lexical search over visible names may stay the only portable option.

## References

- Reference implementation at `9c7ebed`: [`generate.py`][generate.py], [`merge.py`][merge.py], [`ask.py`][ask.py] and [`CLAUDE.md`][CLAUDE.md].
- pgvector: [README][pgvector] and [filtering][pgvector filtering].
- .NET: [IEmbeddingGenerator] and [TensorPrimitives.CosineSimilarity][TensorPrimitives].
- Models: [embeddinggemma], [qwen3-embedding], [bge-m3], [nomic-embed-text], [mxbai-embed-large], and the [EmbeddingGemma model card].
- Inversion: [vec2text] and [ALGEN].

[generate.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/generate.py
[merge.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/merge.py
[ask.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/ask.py
[CLAUDE.md]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/CLAUDE.md
[pgvector]: https://github.com/pgvector/pgvector
[pgvector filtering]: https://github.com/pgvector/pgvector#filtering
[IEmbeddingGenerator]: https://learn.microsoft.com/en-us/dotnet/ai/iembeddinggenerator
[TensorPrimitives]: https://learn.microsoft.com/en-us/dotnet/api/system.numerics.tensors.tensorprimitives.cosinesimilarity
[embeddinggemma]: https://ollama.com/library/embeddinggemma
[qwen3-embedding]: https://ollama.com/library/qwen3-embedding
[bge-m3]: https://ollama.com/library/bge-m3
[nomic-embed-text]: https://ollama.com/library/nomic-embed-text
[mxbai-embed-large]: https://ollama.com/library/mxbai-embed-large
[EmbeddingGemma model card]: https://ai.google.dev/gemma/docs/embeddinggemma/model_card
[vec2text]: https://arxiv.org/abs/2310.06816
[ALGEN]: https://aclanthology.org/2025.acl-long.1185/
