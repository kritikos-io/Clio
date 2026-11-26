# Microsoft.Extensions.AI as the LLM client abstraction

- **Question:** is Microsoft.Extensions.AI (MEAI) a proper abstraction for the LLM clients a Lorekeeper implementation needs, and which providers can it reach?
- **Sources:**
  - package metadata from nuget.org;
  - adapter source at the commits of the shipped packages;
  - vendor documentation;
  - the reference implementation's LLM client ([`llm.py`][llm.py]) at `9c7ebed`.
- **Method:**
  - reflecting over the shipped assemblies;
  - capturing the HTTP request each adapter sends, through a fake message handler with no network;
  - compiling and running a dependency-injection sample with three keyed providers.
- **Date:** 2026-11-23. Package versions are as of that date.

## Summary

- **Yes, MEAI is the right abstraction, with a thin provider-profile layer on top.** The profile layer handles what MEAI leaves to each provider: the token-limit field, structured-output fallback, schema rewriting, model listing, refusals and timeouts.
- **It is mature.**
  - Generally available since 9.5.0 (May 2025), and 11.0.1 is current.
  - A 10.11.x servicing line stays on Microsoft.Extensions 10.
  - Microsoft Agent Framework and Semantic Kernel are both built on it.
- **Every provider in scope has an `IChatClient`.** That covers the official Anthropic, Google and AWS SDKs, Microsoft's adapter for OpenAI and every OpenAI-compatible server, and OllamaSharp for Ollama.
- **It reproduces the reference on the wire.** Captured requests matched everything the reference sends: schema-constrained output on all four native paths, Ollama's context and keep-alive options, Claude's prompt caching with thinking disabled, and the JSON-mode fallback.

## 1. What the reference needs from a client

The reference's client ([`llm.py`][llm.py]) has one operation: complete a system prompt, a user prompt and a JSON Schema into a JSON object. A shared `prefix`, the passage, comes first so that backends can cache it. Per backend:

- **Ollama (`/api/chat`):**
  - `format` set to the schema;
  - `stream: true`, `think: false`, `keep_alive: "30m"`;
  - options `temperature: 0`, `num_ctx: 16384` and `num_predict: 4096`.
- **Claude:**
  - `max_tokens: 16000`, with the passage block marked `cache_control: ephemeral`;
  - `output_config.format` as a JSON Schema, with `additionalProperties: false` on every object;
  - thinking set per model: disabled for Haiku, `between_tools` with the server-side fallback beta for Sonnet, and `effort: low` with the same beta for Opus;
  - never `temperature`.
- **OpenAI-compatible servers (`/chat/completions`):**
  - `response_format` as a non-strict JSON Schema;
  - after a `400`, it falls back to `json_object` with the schema in the prompt, drops `temperature` when the error mentions it, and switches between `max_tokens` and `max_completion_tokens`;
  - it retries `429` and `5xx` with exponential backoff, and lists models with `GET /models`.

The presets cover Claude, OpenAI, Gemini (through its OpenAI-compatible endpoint), OpenRouter, Mistral, Groq, xAI, DeepSeek, and custom servers such as LM Studio, vLLM and llama.cpp. Usage accounting tracks calls, tokens, cache reads and writes, refusals, truncations and cost.

## 2. Status

- **Releases:**
  - 9.5.0 was the first generally available release (2025-05-16).
  - 10.0.0 shipped with .NET 10, and 11.0.0 shipped on 2026-11-11 with a `net11.0` target.
  - 11.0.1 (2026-11-21) is current, with minor releases monthly in between.
  - `Microsoft.Extensions.AI.OpenAI` was preview-only until 10.3.0 (2026-02-10).
- **Packages** ([overview][MEAI overview]):
  - Applications reference `Microsoft.Extensions.AI`: the middleware and dependency-injection helpers.
  - Provider libraries reference only `Microsoft.Extensions.AI.Abstractions`.
- **Stable surface:**
  - `IChatClient` and `IEmbeddingGenerator`.
  - `ChatOptions`, including `Reasoning`, `TopK`, `Seed`, `StopSequences`, `RawRepresentationFactory` and `AdditionalProperties`.
  - Structured output: `ChatResponseFormat.ForJsonSchema` and `GetResponseAsync<T>`. The latter has a built-in fallback that sends JSON mode plus the schema in a message.
  - Usage: `UsageDetails.CachedInputTokenCount` and `ReasoningTokenCount`.
  - Middleware: `UseOpenTelemetry`, which follows the GenAI semantic conventions v1.41, plus `UseLogging`, `ConfigureOptions`, `UseDistributedCache` and `AddKeyedChatClient`.
- **Experimental (`MEAI001`):** routing and failover clients, chat reducers, and the image, speech, realtime and hosted-file clients.
- **Ecosystem** ([.NET AI ecosystem][MEAI ecosystem]): the abstractions were extracted from Semantic Kernel, and Microsoft Agent Framework builds on them. For single-shot structured extraction, neither framework adds anything over MEAI itself.

## 3. Provider coverage

| Provider | Package | Chat | Embeddings | Caveats |
| --- | --- | --- | --- | --- |
| Anthropic Claude | `Anthropic` 13.12.0, official | Yes | No such API | API version 2026-10-21 rejects `temperature`, `top_p` and `top_k` with a `400` ([migration][Anthropic migration]); the adapter drops them. Prompt caching through `WithCacheControl`. Bedrock, Vertex and Foundry variants exist. |
| OpenAI, Azure OpenAI | `Microsoft.Extensions.AI.OpenAI` 11.0.1 | Yes | Yes | Always sends `max_completion_tokens`. Does not map `TopK`. |
| Google Gemini | `Google.GenAI` 2.2.0, official | Yes | Yes | Automatic continuation is on by default, and every hop is billed. |
| AWS Bedrock | `AWS.Bedrock.MEAI` 1.1.3, official | Yes | Yes | Structured output defaults to a synthetic forced tool, which rules out streaming and user tools. Use native mode for newer models. |
| Ollama | `OllamaSharp` 5.5.2 | Yes | Yes | Native API with every option. Joins text parts without a separator, and ends a cancelled stream silently. |
| OpenAI-compatible: OpenRouter, Mistral, Groq, xAI, DeepSeek, Gemini's compatible endpoint, LM Studio, vLLM, llama.cpp, Foundry Local | `Microsoft.Extensions.AI.OpenAI` with a custom endpoint | Yes | Depends on the server | Mistral rejects `max_completion_tokens` with a `422`. DeepSeek has no `json_schema`. Groq supports `json_schema` on some models only. |
| In-process | ONNX Runtime GenAI 0.20.0, LLamaSharp 0.30.0 | Yes | LLamaSharp only | Both are pre-1.0. LLamaSharp ignores `ResponseFormat`. |

Avoid these packages:

- `Microsoft.Extensions.AI.Ollama` and `AWSSDK.Extensions.Bedrock.MEAI`, which are deprecated;
- `Microsoft.Extensions.AI.AzureAIInference`, which never left preview;
- the community packages `Anthropic.SDK` and `Mistral.SDK`, which are superseded or unmaintained.

## 4. Fidelity: the reference's requests through MEAI

Each row below was confirmed by capturing the request body the adapter sends.

| Reference behaviour | How MEAI expresses it |
| --- | --- |
| Ollama `num_ctx`, `keep_alive`, `think: false`, `num_predict`, `format` | `ChatOptions.AdditionalProperties` with Ollama's option names, or a raw `ChatRequest` through `RawRepresentationFactory`. `MaxOutputTokens` maps to `num_predict`. |
| Claude passage caching | `TextContent.WithCacheControl(Ttl.Ttl5m)` sends `cache_control: {type: ephemeral, ttl: 5m}`. |
| Claude thinking disabled | `Reasoning.Effort = None` sends `thinking: {type: disabled}`. |
| Claude `effort: low` (Opus) | `Reasoning.Effort = Low` sends adaptive thinking with `output_config.effort: low`. |
| Claude `between_tools` thinking and the server-side fallback beta | A beta `MessageCreateParams` through `RawRepresentationFactory`, with `Model` and `MaxTokens` set by hand. Not exercised against the live API. |
| Claude JSON output | `ChatResponseFormat.ForJsonSchema` sends `output_config.format`, adding `additionalProperties: false`. |
| OpenAI-compatible JSON Schema output | `ForJsonSchema` sends `response_format: json_schema`; `strict` only through `AdditionalProperties["strict"]`. |
| JSON-mode fallback | `GetResponseAsync<T>(…, useJsonSchemaResponseFormat: false)` sends `json_object`, plus a message with the schema. |
| `max_tokens` instead of `max_completion_tokens` | `RawRepresentationFactory` returning `ChatCompletionOptions` with `Patch.Set("$.max_tokens"u8, n)`. `Patch` is experimental (`SCME0001`). |
| Cached-token accounting | `UsageDetails.CachedInputTokenCount`. Cache writes appear only in provider-specific `AdditionalCounts`. For Anthropic, `InputTokenCount` already includes cache reads and writes. |

## 5. Gaps a provider-profile layer must cover

1. **Token-limit field.** The OpenAI adapter always sends `max_completion_tokens`. Mistral rejects it, and DeepSeek documents only `max_tokens`, so patch `max_tokens` in for them.
1. **Structured-output negotiation.** MEAI never downgrades on its own. A `DelegatingChatClient` can catch the first `400` and retry with JSON mode plus the schema in the prompt, remembering the downgrade per provider, as the reference does.
1. **One schema contract.** The same schema means different things per adapter:
   - The OpenAI adapter rewrites every schema to strict form, even without `strict`: all properties become required, and constraints such as `maxLength` move into descriptions.
   - Anthropic adds `additionalProperties: false`.
   - Ollama and Gemini pass schemas through unchanged.

   Designing schemas to be strict-compatible, with nullable required properties instead of optional ones, gives every provider the same contract.
1. **Model listing.** MEAI has none; each SDK has its own call.
1. **Refusals and cache writes.** Anthropic reports refusals as `ContentFilter`, OpenAI as an `ErrorContent` with code `Refusal`. Cache writes have provider-specific keys. Normalise both.
1. **Timeouts and retries.**
   - The SDKs retry on their own: OpenAI three times, Anthropic twice.
   - The default `AddStandardResilienceHandler` budget, 10 seconds per attempt and 30 seconds in total, is far too short for LLM calls ([HTTP resilience]).
   - Raise both timeouts, keep the circuit breaker's sampling duration at least twice the attempt timeout (validation fails otherwise), and retry a stream only before its first chunk.
1. **Name collisions.** `OpenAI.Chat` and `Microsoft.Extensions.AI` both define `ChatMessage`, `ChatClient` and `ChatResponseFormat`, so never import both namespaces in one file.

Escape hatches exist for everything else:

- **`RawRepresentationFactory`** returns the provider's own request object, and raw values win over `ChatOptions`.
- **`AdditionalProperties`** carries provider-specific options by name.
- **`GetService<T>()`** reaches the underlying SDK client.
- **`RawRepresentation`** exposes the raw response.

## 6. Recommended packages

| Purpose | Package | Version |
| --- | --- | --- |
| Abstractions, middleware, dependency injection | `Microsoft.Extensions.AI` | 11.0.1, or 10.11.1 to stay on Microsoft.Extensions 10 |
| OpenAI and every OpenAI-compatible endpoint | `Microsoft.Extensions.AI.OpenAI` (brings `OpenAI` 2.15.0) | 11.0.1 |
| Claude | `Anthropic` | 13.12.0 |
| Ollama | `OllamaSharp` | 5.5.2 |
| Gemini, natively (optional) | `Google.GenAI` | 2.2.0 |
| Bedrock (optional) | `AWS.Bedrock.MEAI` | 1.1.3 |
| HTTP resilience | `Microsoft.Extensions.Http.Resilience` | 11.0.1 |
| Telemetry | `OpenTelemetry.Extensions.Hosting` and an exporter | 1.21.0 |

> [!NOTE]
> On `net10.0`, MEAI 11.x pulls in the 11.0 Microsoft.Extensions abstractions, while 10.11.x stays on 10.0. The vendor SDKs compile against older abstractions, so smoke-test every MEAI upgrade.

## 7. Recommendation

Adopt MEAI. Start with three keyed clients, Ollama, Anthropic and a generic OpenAI-compatible one, plus a provider profile that records:

- the token-limit field;
- the structured-output mode;
- the reasoning setting;
- whether and where to mark cache breakpoints;
- prices, for cost accounting.

Wrap the clients with `UseOpenTelemetry` and `UseLogging`, and give them timeouts measured in minutes.

## 8. Open questions

1. **Version line:** MEAI 11.0.x or 10.11.x? It depends on whether the application targets .NET 11 or stays on .NET 10.
1. **Gemini:** the native `Google.GenAI` SDK (explicit caching, thinking control, cached-token reporting) or the OpenAI-compatible endpoint the reference uses? The latter is in beta and silently ignores unknown parameters.
1. **Schemas:** redesign them to be strict-compatible, or bypass the OpenAI adapter's rewriting by supplying the response format through `RawRepresentationFactory`?
1. **Claude features:** are the server-side fallback beta and `between_tools` thinking needed? Both work only through raw request parameters.
1. **Model presets:** the reference's presets look stale against current provider documentation; for example, DeepSeek's current models are `deepseek-flash` and `deepseek-v4-pro`. They need re-verifying.
1. **Live checks:** confirm against each real endpoint:
   - LM Studio's handling of `max_completion_tokens`;
   - which servers accept the array form of message content;
   - how ONNX Runtime GenAI maps `ResponseFormat`.

## References

- Microsoft: [MEAI overview], [IChatClient], [structured output quickstart][Structured output], [.NET AI ecosystem][MEAI ecosystem] and [HTTP resilience].
- Adapters: [`OpenAIChatClient.cs`][OpenAIChatClient] at the 11.0.1 commit, the [Anthropic C# SDK][Anthropic SDK] and its [cache-control extensions][Anthropic cache], [`GoogleGenAIChatClient.cs`][GoogleGenAIChatClient], [OllamaSharp's mapper][OllamaSharp mapper] and [`AWS.Bedrock.MEAI`][Bedrock MEAI].
- Providers: [Anthropic API versions][Anthropic versioning] and its [2026-10-21 migration][Anthropic migration], [Gemini OpenAI compatibility][Gemini compat], [DeepSeek chat completions][DeepSeek], [Mistral chat API][Mistral] and [Groq structured outputs][Groq].
- Reference: [`llm.py`][llm.py] at `9c7ebed`.

[llm.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/llm.py
[MEAI overview]: https://learn.microsoft.com/en-us/dotnet/ai/microsoft-extensions-ai
[IChatClient]: https://learn.microsoft.com/en-us/dotnet/ai/ichatclient
[Structured output]: https://learn.microsoft.com/en-us/dotnet/ai/quickstarts/structured-output
[MEAI ecosystem]: https://learn.microsoft.com/en-us/dotnet/ai/dotnet-ai-ecosystem
[HTTP resilience]: https://learn.microsoft.com/en-us/dotnet/core/resilience/http-resilience
[OpenAIChatClient]: https://github.com/dotnet/extensions/blob/1e8c914781cf8378f3bbb547bcb024518a2be74c/src/Libraries/Microsoft.Extensions.AI.OpenAI/OpenAIChatClient.cs
[Anthropic SDK]: https://platform.claude.com/docs/en/api/sdks/csharp
[Anthropic cache]: https://github.com/anthropics/anthropic-sdk-csharp/blob/0cb7408849fb0fc3277c6c3f35fced7d26c2bf4e/src/Anthropic/AIContentCacheExtensions.cs
[GoogleGenAIChatClient]: https://github.com/googleapis/dotnet-genai/blob/d388e083a6a4a7a9e4bcfa69a07090e66ed8e3bb/Google.GenAI/GoogleGenAIChatClient.cs
[OllamaSharp mapper]: https://github.com/awaescher/OllamaSharp/blob/a1ce0bd5f1021efd8d9dd1818a319575bb7b31a1/src/OllamaSharp/MicrosoftAi/AbstractionMapper.cs
[Bedrock MEAI]: https://github.com/aws/aws-dotnet-ai/blob/main/src/AWS.Bedrock.MEAI/README.md
[Anthropic versioning]: https://platform.claude.com/docs/en/api/versioning
[Anthropic migration]: https://platform.claude.com/docs/en/api/migration-2026-10-21
[Gemini compat]: https://ai.google.dev/gemini-api/docs/openai
[DeepSeek]: https://api-docs.deepseek.com/api/create-chat-completion
[Mistral]: https://docs.mistral.ai/api/endpoint/chat
[Groq]: https://console.groq.com/docs/structured-outputs
