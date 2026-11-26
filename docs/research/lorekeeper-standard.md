# The Lorekeeper standard: what an implementation must reproduce

- **Question:** what must a new implementation reproduce to interoperate with Lorekeeper's reference implementation, its registries and its clients?
- **Sources:** the [specification][SPEC] v0.1 (draft), its JSON Schemas and conformance suite, and the [reference implementation][Reference], all at commit `9c7ebed` (v0.13.0, 2026-11-21).
- **Method:** reading the specification and the reference source, then running Python 3.13 and .NET 10 and 11 to confirm the platform behaviour each finding depends on.
- **Verified:** 2026-11-26.

## Summary

- **Most of the standard needs no book text.** Validation (all but two checks), gating, the read protocol with anchor positions, and registries can all be built before any EPUB handling. The text model, locators, content rules, index requests and EPUB embedding need the book.
- **A digest covers the bytes the registry writes, not the bytes the publisher sends.** The reference re-serializes each shared index with Python's `json.dumps`. Two registries give the same bundle the same digest only if they serialize identically.
- **The reference client speaks only the static registry profile.** It looks books up by text hash through a fixed path, follows only links of one exact shape inside the registry, verifies digests, and skips books that belong to a series.
- **Order is part of conformance.** Views are compared with array order significant, and the reference algorithm derives that order from a single counter.
- **Python's semantics leak into the protocol.** Name matching uses Python's case folding and whitespace rules, the validator asserts only the `date` format, and the JSON parser keeps the last of duplicate member names.
- **A registry that holds no editions answers every locator with `409`.** The specification requires exactly that.

## 1. The standard in brief

- An **index** (§4) describes one book: entities with their names and facets, relations, events and recaps. Every item carries `from`, the anchor at which the reader learns it.
- **Anchors** (§3.5) point at blocks of the book's text through fingerprints, so they resolve in any edition with the same text (§3.1–§3.4).
- A reader's **position** (§3.7) is the last anchor passed. The **gating algorithm** (§7) computes the **view**: everything visible at that position, merged across the books of a **series** (§5).
- The **protocol** (§8) lets reader apps discover a server, look a book up and fetch its index. A server may also compute views and answer "who is this name?" for thin clients.
- **Registries** (§8.7) are servers where publishers share indexes with each other, either as dynamic servers or as plain files on a static host.

## 2. What needs the book's text

| Area | SPEC | Needs the book's text | Notes |
| --- | --- | --- | --- |
| Text model, normalisation, fingerprints, edition identity | §3.1–§3.4 | Yes | Defines `textHash`. |
| Anchors | §3.5–§3.6 | To resolve them | Their order comes from the index. |
| Reader position | §3.7 | No | Given as an anchor. |
| Locators | §3.8 | Yes | `block`, `cfi`, `xpointer`, and `spine` with `word` or `progress`. |
| Index documents and series | §4, §5 | No | |
| Content rules | §6.1–§6.3 | Yes | For example, no run of more than eight consecutive words from the book. |
| Validation | §6.4 | Two of eleven checks | Anchors resolving to blocks of six or more words with matching `seq` hints, and the eight-word rule. |
| Gating | §7 | No | |
| Discovery, lookup, fetch, view, entity lookup | §8.1–§8.4, §8.6 | Only for locator positions | |
| Requests | §8.5 | Yes | Asks a server to build an index for a book it holds. |
| Registries | §8.7 | No | "A registry need not hold the book." |
| Embedding in EPUB | §9 | Yes | |

Conformance (§10) is defined per role:

- **Index documents** pass every §6.4 check.
- **Clients** reproduce the expected view for each of the 19 gating cases in `conformance/`, compared by JSON equality with array order significant. The cases cover the opening of a book, replaced and refined facets, exhausted coverage, series merging, carried entities, unfinished and future books, profiles and arcs.
- **Servers** answer with documents that match `api.schema.json`, and their views pass the client cases.
- **Registries** serve listings that match `listing.schema.json`, every `digest` equals the digest of the bytes served at its `href`, and static registries serve the §8.7 paths.
- **Locators** have 32 cases in `conformance/locators/`. A client or server must pass the cases of every locator form it accepts.

## 3. The sharing model (§8.7)

- **Publishers and slugs.**
  - A `publisher` is 1 to 39 characters of `[A-Za-z0-9-]`, not starting or ending with `-`, compared case-insensitively and stored lowercased.
  - The `slug` is the index `id` lowercased, with every character outside `[a-z0-9-]` replaced by `-`.
  - A listing's id is `publisher/slug`, and it stays the same across versions.
- **Versions and digests.** Each share of a book by the same publisher is a new, immutable version numbered from 1. Its digest is `sha256:` followed by the lowercase hex SHA-256 of the exact stored bytes.
- **Listings** record one book shared by one publisher:
  - the work, the editions (only `textHash` and identifiers; never `fileHash`, `label` or `builtFrom`), every version, completeness, licence, generator, and whether a person edited it;
  - for a series book, the listings it `requires`: every earlier book in publication order, by the same publisher;
  - optionally `derivedFrom`, the digest of a shared index this one improves.
- **Share bundles** carry the index with labels and file hashes removed, the series manifest when the index belongs to a series, an optional `derivedFrom`, and the builder's attestations.
- **Attestations** say what was checked (`validates`, `book-checks`, `spoiler-gate`, `audit`), who vouches for it (`registry`, `builder` or `reader`) and when.
  - Their counts must be possible. A registry refuses impossible counts, and a client ignores them.
  - Nothing verifies builder or reader attestations; signatures are an open question (§11).
- **Intake.** Before accepting a bundle, a registry MUST:
  1. refuse a bundle over its size limit before parsing it (the reference uses 5 MiB);
  1. refuse a bundle that fails its schema, or an index that fails a §6.4 check that needs no book. A series book is checked against the latest versions of the publisher's listings in `requires`; a missing one is an error;
  1. refuse a licence it does not accept (the community registry accepts `CC0-1.0` and `CC-BY-4.0`);
  1. refuse a digest that is already a version of the same listing;
  1. refuse a stored index or listing over the size limit;
  1. build storage paths only from the validated publisher and the computed slug.
- **After acceptance:**
  - The registry writes the version, replaces the series manifest, and replaces the listing's attestations with the bundle's plus its own `validates`.
  - A new version of a book that later books require must leave their series checks passing.
  - Removing a listing must also remove the later books that require it, or be refused.
  - Removals go to a public log, and the registry SHOULD refuse new versions under a removed id.
- **Timestamps** are UTC, `YYYY-MM-DDTHH:MM:SSZ` with optional fractional seconds; a registry MUST refuse any other form.
- **Static profile.** A registry made only of files:
  - sets `"static": true` in discovery;
  - answers lookups by path, `{api}/lookup/{scheme}/{value}.json`, for the schemes `text`, `isbn` (ISBN-13), `gutenberg`, `openlibrary` and `wikidata`;
  - serves listings, versioned indexes, series manifests and the removal log at fixed paths;
  - offers neither views nor entity lookup.

  Clients MUST verify digests on a static registry, and SHOULD on any registry.
- **Privacy.** A lookup tells the registry that someone holds the book, so clients SHOULD look up only the book the reader opens.

## 4. Interoperability findings

### 4.1 Digests depend on how the registry serializes

**Finding.** The specification fixes the digest to the exact stored bytes, but not the format of those bytes. The reference stores `dumps(index)`: the `index` member of the bundle, re-serialized with `json.dumps(index, ensure_ascii=False, indent=1, allow_nan=False)` plus a final newline. The bytes the publisher uploaded are never stored ([`registry.py`][registry.py] `dumps`, `_check`).

That serialization behaves as follows (verified with Python 3.13):

| Aspect | Behaviour |
| --- | --- |
| Layout | One space per nesting level; a comma and a newline between items; `": "` after keys; `{}` and `[]` when empty; keys in document order, not sorted. |
| Strings | Non-ASCII characters are written as they are, including U+2028 and emoji. Only `"`, `\`, `\b`, `\f`, `\n`, `\r`, `\t` and other control characters are escaped, the latter as lower-case `\u00xx`. |
| Integers | Written as parsed, with arbitrary precision; `-0` becomes `0`. |
| Floats | Re-formatted as Python's `repr`: the shortest round-trip digits, with an exponent below 1e-4 and from 1e16 (`1e-05`, `1e+16`). Integral values keep `.0` (`31.0`, and `100.0` for `1.0e2`), and `-0.0` stays. |
| Not allowed | `NaN` and the infinities. |

How the reference writes each file:

- Index versions are written in binary mode, so they end in LF on every platform.
- `listing.json`, `series.json` and `removed.json` are written in text mode (`Path.write_text`), which turns each newline into CRLF on Windows.
- The static build (`registry build`) re-serializes listings, lookup files and discovery with sorted keys. It copies index versions, series manifests and the removal log byte for byte, so a static registry built on Windows serves CRLF series manifests and removal logs.

**Implication.**

- An implementation reproduces reference digests only with an exact port of these rules.
- §10 requires less: only that each digest matches the bytes served. Parity matters when the same index should carry the same digest on several registries, for mirrors or to recognise one index across registries.
- Only index versions are digest-addressed, so the formatting of listings, series manifests and the removal log is cosmetic.

### 4.2 The reference client speaks only the static profile

**Finding.** The reference reader's community client ([`community.py`][community.py]) describes itself as looking up "ready-made indexes for a book in a static registry". It defaults to the [community registry], which is hosted on GitHub Pages. It accepts a registry only under these conditions:

- **Lookup.** It requests only `{registry}/v0/lookup/text/{textHash}.json`, never the query-string lookup of §8.2. A `404` means no results.
- **Lookup body.** It must be exactly `{"results": […]}`: any other top-level member rejects the whole response. At most 200 results are considered.
- **Results.** A result is used only when all of these hold:
  - it is valid against the API schema, and its `listing` passes the listing checks;
  - `listing.id` and `listing.digest` equal the result's;
  - the listing has an edition with the requested text hash;
  - neither the result nor its listing belongs to a series ("a series set is not offered in this version").
- **Links.** An `href` is followed only inside the registry's own origin and base path. After the base path it must match `^/v0/indexes/([A-Za-z0-9-]{1,128})/([A-Za-z0-9-]{1,128})/v([0-9]{1,6})\.lore\.json$`, with the listing's publisher, slug and version.
- **Digests.** Downloaded bytes must hash to the listed digest, or the download is discarded.
- **Transport and consent:**
  - HTTPS only; plain HTTP works only for localhost;
  - bodies up to 6 MiB, 10 seconds per fetch, at most 3 redirects;
  - lookups cached for 10 minutes;
  - nothing is fetched until the reader turns the feature on.

**Implication.** For the reference client to use a dynamic registry, the registry must also serve:

- the static lookup path, with a body whose only member is `results`;
- versioned index paths of exactly that shape.

Books in a series cannot be installed through the reference client yet, wherever they are hosted.

### 4.3 Lookup differs between the reference's two implementations

**Finding.** The reference implements lookup twice, with different rules.

| | Library protocol server ([`protocol.py`][protocol.py]) | Static registry build ([`registry.py`][registry.py] `build`) |
| --- | --- | --- |
| Query | `fileHash`, `textHash`, `isbn`, `gutenberg`, `openlibrary`, `wikidata`; at least one | One file per key: `text/{textHash}`, and `isbn`, `gutenberg`, `openlibrary` and `wikidata` identifiers |
| Hashes | Compared exactly | Exact file names |
| Identifiers | Gutenberg, Open Library and Wikidata compared case-insensitively | Written as stored: "the value keeps its case" (§8.7) |
| ISBN | Only letters and digits are kept, then upper-cased; matched against `work.isbns` and the `isbn` identifier. No ISBN-10/13 conversion. | Values of both the `isbn` and `isbn13` schemes, unconverted, although §8.7 defines the scheme as ISBN-13 |
| Keys | — | Only values matching `^[A-Za-z0-9]{1,64}$`. Intake merely warns about others, so a hyphenated ISBN never becomes a key. |
| Order | Match (`file`, then `text`, then `identifier`), then complete first, then `id` | Complete first, then most recently `updated`, then `id` (§8.7) |

**Implication.** A dynamic registry has to choose:

- whether identifiers match case-insensitively, as the library server does, or exactly, as static paths must;
- whether to convert ISBN-10 to ISBN-13, as the specification's wording suggests but neither reference implementation does;
- how to order results. §8.2 asks only for "best first"; matching the static order keeps the two routes consistent.

### 4.4 Intake order and limits in the reference registry

**Finding.** The reference checks a bundle in this order ([`registry.py`][registry.py] `_check`):

1. Normalise the publisher.
1. Refuse a bundle over 5 MiB, before parsing it.
1. Parse it within limits:
   - **Refused:** nesting deeper than 32 levels, `NaN` and the infinities, lone surrogates, and a byte-order mark.
   - **Accepted:** duplicate member names, with the last value winning (Python's default).
1. Refuse anything but a JSON object.
1. Run the preflight checks, which give clearer messages than the schema:
   - editions still carrying `label` or `fileHash`;
   - attestations not `by: "builder"`, of kind `validates`, with a malformed `at`, or with impossible counts;
   - a series manifest sent with an index that has no `series`.
1. Validate against the share schema.
1. Check the licence.
1. Compute the slug, and refuse a removed listing id.
1. Run the index checks:
   - **Series book:** resolve `requires` from the manifest's publication order (or its default order), then run the series checks against the latest versions.
   - **Standalone book:** run the index checks.
1. Re-serialize the index, check its stored size, and compute its digest.
1. Check slug ownership: an existing listing must belong to the same index `id`.
1. Refuse a second listing by the same publisher for the same series book.
1. Refuse a digest that is already a version of this listing.
1. Check that the books requiring this one still pass.
1. Check that every attested text hash is one of the index's editions.
1. Warn about identifiers that cannot be lookup keys.
1. Build the listing, validate it against the listing schema, and check its stored size.

The first six steps stop at the first failure. From the licence on, errors accumulate, so the publisher sees them together.

**Implication.** Reporting the same first error as the reference needs the same order and the same limits. Refusing duplicate member names is stricter than the reference; see [design options](#5-design-options-beyond-the-specification).

### 4.5 Series rules only read one publisher's listings

**Finding.** §8.7 makes a series "shared as a set: one publisher's third book cannot sit on another publisher's first". Every rule that looks beyond the submitted listing reads the same publisher's listings ([`registry.py`][registry.py] `requires_for`, `_dependents_still_hold` and the duplicate-book check):

- the books a series book requires;
- the uniqueness of each series book;
- the books that depend on a new version;
- the books a removal would cascade to.

**Implication.** A database-backed registry can make check-then-write atomic by serializing writes per publisher, for example with a PostgreSQL transaction-scoped advisory lock keyed by the publisher. Other publishers are not blocked.

### 4.6 The reference validator asserts only the `date` format

**Finding.**

- The reference validates with the `jsonschema` package's default format checker, and depends on nothing else (`jsonschema>=4.21`; [`validate.py`][validate.py]).
- Without the optional format packages, that checker asserts `date` but not `date-time`. With jsonschema 4.26, an invalid `date-time` passes.
- The schemas use `date` once (index) and `date-time` seven times (index twice, listing three times, API twice).
- Registry timestamps are checked separately, by the §8.7 rule.

**Implication.** A validator that asserts every format refuses some indexes the reference accepts. Asserting only `date`, and checking registry timestamps by rule, keeps the two in agreement.

### 4.7 Gating order is part of conformance

**Finding.** §10 compares views with array order significant, and §7.5 requires a server's views to equal the client algorithm's output. The reference algorithm ([`gating.py`][gating.py] `view`) orders its output like this:

- Every visible item gets the sort key *(book rank, anchor ordinal, file order)*.
- *File order* is one counter. It is incremented before each name, facet, relation, event and recap is visited, whether or not that item turns out visible.
- An entity takes the counter's current value without incrementing it. Entities are then sorted by their smallest key, then by `id`.
- A standalone index has the book key `None`.
- A position without an anchor means "before the first anchor": nothing from the current book is visible, while finished books are visible in full.

**Implication.**

- Ties between items at the same anchor are broken by the counter, so a port must visit items and increment the counter exactly as the reference does.
- Some languages cannot use a null key in a dictionary; they need a sentinel for standalone books.

### 4.8 Positions, locators and errors on servers

**Finding.** §8.4 requires exactly one position:

- either an `anchor`, or one locator form together with `textHash` or `fileHash`;
- sending both, or several locator forms, is `400`.

A server resolves locators only against editions it holds; for any other edition it answers `409`, and the client falls back to local gating. The reference library server ([`protocol.py`][protocol.py]) adds these details:

- **Requests without a position** are refused: `400`, "give a position: anchor, block, cfi, xpointer, spine with word, or spine with progress".
- **Locator syntax:**
  - `cfi` must look like `epubcfi(…)`, and `xpointer` must start with `/`;
  - `spine` takes exactly one of `word` or `progress`;
  - `progress` lies between 0 and 1, and integers are non-negative.
- **`inclusive`** accepts `true`, `false`, `1` or `0`, in any case. It defaults to `false` for views (the `finished` mode of §3.7) and to `true` for entity lookup.
- **Problems** use the type `https://lorekeeper.dev/problems/{slug}`, with slugs `bad-request`, `not-found`, `edition-not-held`, `unauthorized` and `method-not-allowed`.
- **CORS:**
  - `Access-Control-Allow-Origin: *`;
  - methods `GET` and `OPTIONS`;
  - request headers `Authorization` and `If-None-Match`;
  - `ETag` exposed;
  - preflights cached for 600 seconds.
- **Caching:**
  - `ETag` is the quoted digest;
  - `If-None-Match` accepts `*`, weak tags and lists;
  - `Cache-Control` is `private, max-age=0, must-revalidate`.
- **Bearer tokens** are compared in constant time.
- **Discovery:** a server that accepts `cfi` and `xpointer` advertises the capability `view-cfi`.

**Implication.** A registry holds no editions, so it answers every well-formed locator with `409 edition-not-held`, which is conformant. Accepting locators therefore depends on holding editions, which in turn needs the book's text (section 2).

### 4.9 Name matching follows Python's Unicode rules

**Finding.** §8.6 matches a name by "an exact, case-insensitive comparison" with a visible name or alias, or by a unique surname. The reference implements this as follows ([`protocol.py`][protocol.py] `get_entities`):

- Names are normalised with `" ".join(name.split()).casefold()`.
- A one-word query also matches the last word of a multi-word visible name, when exactly one visible entity has that surname.

Python's rules differ from .NET's (both verified):

- **Case folding:** [`str.casefold()`][Python casefold] maps `Straße` to `strasse`, while `ToLowerInvariant` keeps `straße`.
- **Whitespace:** [`str.split()`][Python isspace] splits on U+001C–U+001F, which [`char.IsWhiteSpace`][char.IsWhiteSpace] does not treat as whitespace.

**Implication.** Matching the reference for every input means porting Python's case-folding and whitespace tables, generated from the Python version the reference runs on.

## 5. Design options beyond the specification

| Option | What the specification and the reference say | Trade-off |
| --- | --- | --- |
| A registry that also computes views and entity lookups | Allowed: §8.7 rules them out only for static registries. The reference has no such server; its registry is static, and only its library server computes views. | Serves thin clients, but sends reading positions to the registry, which §8.4 calls "a privacy cost". The registry must also choose the indexes to merge. The library server takes the manifest and sibling books from the same source; for a registry that maps to the same publisher's latest versions, in manifest order. |
| A request without a position | §8.4 requires exactly one position, and the reference answers `400`. | Answering with the finished book's view is convenient for "show me everything", but turns a forgotten parameter into a full spoiler. An explicit parameter keeps the convenience without the risk. |
| Duplicate member names in bundles | The reference keeps the last value. | Refusing them removes a meaning that depends on the parser (others keep the first value), at the cost of refusing bundles the reference accepts. |
| ISBN-10 in lookups | §8.7 defines the static `isbn` scheme as ISBN-13; neither reference implementation converts. | Converting on intake and lookup finds more books, but disagrees with the reference's matching. |
| Identifier case | The library server ignores case; static paths keep it. | Choose per route, and document the choice. |
| An extra discovery path, such as `/.well-known/clio` | Not in the specification. | Harmless if it returns the same document. Renaming `/.well-known/lorekeeper`, the `/v0` paths, the `"lorekeeper"` member or the problem types would break clients and digests. |

## 6. Notes for a .NET implementation

Every row below was verified on 2026-11-26 by running .NET 10.0.15 and 11.0.0. The JSON Schema row comes from the published licence.

| Topic | Finding | Implication |
| --- | --- | --- |
| JSON writer | `Utf8JsonWriter` can match Python's layout (`IndentSize = 1`, `NewLine = "\n"`), but not its escaping or its numbers. The default encoder escapes non-ASCII characters and `<`. The relaxed encoder still escapes U+2028 and characters beyond the Basic Multilingual Plane, with upper-case hex. Numbers are written exactly as parsed (`1.0e2`, `-0`, `1e16`). | Digest parity needs a custom string escaper and a Python-compatible float formatter. |
| Case folding and whitespace | `ToLowerInvariant` is not `casefold()`, and `char.IsWhiteSpace` differs from Python's `isspace()` (section 4.9). | Port Python's tables. |
| Timestamps | .NET 11's `DateTime.TryParseExact` accepts hour 24 and rolls over to the next day; .NET 10 refuses it. | Check the date and time components explicitly, since §8.7 requires refusing malformed timestamps. |
| System.Text.Json source generation | On .NET 10, a source-generated deserializer sets a missing init-only collection to `null`, discarding its initializer. .NET 11 keeps the initializer. | On .NET 10, guard optional collections, for example `{ get => field ?? []; init; }`. |
| JSON Schema 2020-12 | Since 1 February 2026, [JsonSchema.Net][OSMF EULA] binaries carry the OSMF EULA. Revenue-generating users with annual revenue of US$10,000 or more pay a fee; the source stays MIT. Its root namespace `Json` is shadowed inside any namespace that ends in `.Json`. | Budget for the fee or choose another validator. Use `global::Json.Schema` where shadowed, and assert only `date` (section 4.6). |
| Null keys | `Dictionary` keys cannot be null, while the reference keys standalone books by `None`. | Use a sentinel key. |
| Target framework | .NET 10 (LTS) and .NET 11 (STS) both reach end of support on 2028-11-14 ([support policy][.NET support policy]). | Support length does not decide between them. |

The [repository template][Template] also constrains the container build, whatever the code:

- One `DOTNET_VERSION` argument selects both the SDK image and the runtime image. A solution that targets both `net10.0` and `net11.0` therefore cannot build in the .NET 10 SDK image, and a .NET 11 runtime image does not run `net10.0` applications without roll-forward.
- The `test` stage runs every test project inside `docker build`, where no Docker daemon is available. Integration tests that start containers fail there, and the image is published only after the tests pass.
- The final stage starts `/app/${PUBLISHED_PROJECT}.dll`, while the template names assemblies `Kritikos.<Project>`, so the entry point names a file that does not exist.

## 7. Open questions

1. **Views on a registry:** should a registry compute views and entity lookups for thin clients, at the cost of learning reading positions?
1. **Missing position:** what should a request without a position return: `400` as in the reference, or a view of the finished book?
1. **Byte parity:** is digest parity with reference registries a requirement, or is §10's guarantee enough?
1. **Lookup rules:** should identifiers match case-insensitively, and should ISBN-10 be converted to ISBN-13?
1. **Duplicate member names:** refuse them, or accept them as the reference does?
1. **Schema validator:** JsonSchema.Net under the OSMF EULA, or another JSON Schema 2020-12 validator?
1. **The specification's own open questions (§11):**
   - a second fingerprint for blocks that editions split or merge;
   - signed indexes and attestations;
   - translations;
   - formats other than EPUB.

   All four affect editions and registries later.

## References

- [Lorekeeper specification at `9c7ebed`][SPEC], and the [reference implementation at the same commit][Reference]. The specification text, schemas and conformance cases are CC BY 4.0; the code is Apache-2.0.
- Reference modules: [`registry.py`][registry.py], [`gating.py`][gating.py], [`validate.py`][validate.py], [`web/protocol.py`][protocol.py] and [`web/community.py`][community.py].
- [Schemas] and [conformance suite].
- Python: [`json`][Python json], [`str.casefold`][Python casefold] and [`str.isspace`][Python isspace]. .NET: [`char.IsWhiteSpace`][char.IsWhiteSpace] and the [.NET support policy].
- [json-everything OSMF EULA][OSMF EULA].

[SPEC]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/SPEC.md
[Reference]: https://github.com/nightspine/lorekeeper/tree/9c7ebed838b7ecd553e358d470ff6096b3aafdf0
[registry.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/registry.py
[gating.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/gating.py
[validate.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/validate.py
[protocol.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/web/protocol.py
[community.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/web/community.py
[Schemas]: https://github.com/nightspine/lorekeeper/tree/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/schema
[conformance suite]: https://github.com/nightspine/lorekeeper/tree/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/conformance
[community registry]: https://nightspine.github.io/lorekeeper-registry
[Python json]: https://docs.python.org/3/library/json.html
[Python casefold]: https://docs.python.org/3/library/stdtypes.html#str.casefold
[Python isspace]: https://docs.python.org/3/library/stdtypes.html#str.isspace
[char.IsWhiteSpace]: https://learn.microsoft.com/en-us/dotnet/api/system.char.iswhitespace
[OSMF EULA]: https://github.com/json-everything/json-everything/blob/master/OSMFEULA.txt
[.NET support policy]: https://dotnet.microsoft.com/en-us/platform/support/policy
[Template]: https://github.com/kritikos-io/templates-dotnet
