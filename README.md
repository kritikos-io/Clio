# Clio

A planned .NET implementation of the [Lorekeeper] open standard for spoiler-safe book companions, where people share gated indexes of books and reader apps show only what the reader has reached.

> [!NOTE]
> Clio is in its research phase: there is no implementation yet. The papers below are its starting point.

## Research

| Topic | Paper |
| --- | --- |
| What an implementation must reproduce to interoperate with Lorekeeper's reference implementation, registries and clients | [The Lorekeeper standard](docs/research/lorekeeper-standard.md) |
| What vectorizing books in PostgreSQL would add | [Book vectorization](docs/research/book-vectorization.md) |
| Microsoft.Extensions.AI as the LLM client abstraction, and the providers it reaches | [Microsoft.Extensions.AI](docs/research/microsoft-extensions-ai.md) |
| Parameters, determinism and quality for small local models | [Local models](docs/research/local-models.md) |

The [research index](docs/research/README.md) also covers:

- the findings across all four papers;
- the build order they imply;
- the decisions still open.

## Repository Template

Clio follows the [kritikos-io templates-dotnet][Template] conventions:

- GitVersion-driven versions;
- the artifacts output layout;
- central package management;
- a multi-stage `docker/Dockerfile`, where `RUNTIME_BASE` is `web`, `app` or `self-contained`;
- OpenAPI documents generated at build time and linted with [Spectral].

Pull template updates with `git pull template main`. The [Lorekeeper paper](docs/research/lorekeeper-standard.md#6-notes-for-a-net-implementation) lists the constraints the template's container build places on an implementation.

## Attribution

The Lorekeeper specification, JSON Schemas and conformance cases are by the Lorekeeper authors, under [CC BY 4.0]. The reference implementation is under the [Apache License 2.0][Apache-2.0].

[Lorekeeper]: https://github.com/nightspine/lorekeeper
[Template]: https://github.com/kritikos-io/templates-dotnet
[Spectral]: https://github.com/stoplightio/spectral
[CC BY 4.0]: https://creativecommons.org/licenses/by/4.0/
[Apache-2.0]: https://www.apache.org/licenses/LICENSE-2.0
