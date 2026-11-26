# Small local models: parameters, determinism and quality

- **Question:** which optional parameters matter when small models run locally, and how can their results approach a cloud model's, even if they take longer?
- **Sources:**
  - [Ollama][Ollama] v0.40.17 and [llama.cpp][llama.cpp server] build b12257: source and documentation;
  - the vLLM and SGLang documentation;
  - Ollama's model library pages;
  - the [LLM-AggreFact] leaderboard and the papers cited below;
  - the generation code and benchmark runs of the Lorekeeper reference implementation at `9c7ebed`.
- **Method:** reading the runtimes' source at pinned versions, and analysing the reference's benchmark results.
- **Date:** 2026-11-23. Model names and runtime versions are as of that date.

## Summary

- **Ollama is now llama.cpp, or MLX on Apple Silicon.** Its behaviour is that of the underlying engine plus Ollama's launch flags, so pin and record the Ollama version.
- **Parameters shipped with a model still apply unless a request overrides them.**
  - `qwen3.5:9b` ships with a presence penalty of 1.5.
  - Penalties change token scores before greedy selection, so even at temperature 0 they push the model away from names and paragraph numbers it has just written.
  - Send penalties explicitly on every call.
- **Defaults are stale and failures are silent.**
  - The context window depends on VRAM: 4K tokens below 24 GiB.
  - An overlong prompt silently loses its middle, which includes the system prompt.
  - Set `num_ctx`, and send `truncate: false` and `shift: false`.
- **Identical output is not achievable on Ollama or llama.cpp.** The batch-invariant modes of vLLM and SGLang cost roughly 25–60% of throughput. Parity with a cloud model has to be shown statistically.
- **Output tokens dominate the time on local hardware**, at about 7 to 10 per second. Quality levers that add prompt tokens or single-pass checks are cheap; levers that multiply output tokens are not.
- **The best quality per minute comes from a dedicated fact-checker.** Bespoke-MiniCheck-7B (77.4) and Clef Flash 9B (76.6) match GPT-4o (75.9) on LLM-AggreFact, at one forward pass per claim.
- **Never let a model check its own claims.** On the same chapters, gemma4 rejected none of its own claims, while Claude Haiku rejected 6 and moved 5 later.

## 1. What the reference does today

**Requests.** The reference's Ollama client ([`llm.py`][llm.py]) sends:

- `/api/chat` with `format` set to the schema, `stream: true`, `think: false` and `keep_alive: "30m"`;
- the options `temperature: 0`, `num_ctx: 16384` and `num_predict: 4096`, and nothing else: no penalties, `top_k`, `top_p`, `seed`, `truncate` or `shift`;
- the passage first in the user message, so the cache can reuse it.

A reply cut off by the length limit returns an empty result.

**Generation** ([`generate.py`][generate.py]):

- Parts of 2,500 words for Ollama, and whole chapters of up to 8,000 words for cloud models.
- Each part gets one extraction call, a rephrasing call only when a claim copies nine or more words, and one verification call over the same passage.
- The verification schema puts the verdict before its reason.

**Benchmarks** in the reference ([`bench/`][bench]). The model bake-off over 15 paragraphs:

| Model | Wall time | Output tokens | Tokens per second | Entities | Facts | Recall against 7 hand-written |
| --- | --- | --- | --- | --- | --- | --- |
| gemma4:12b-it-qat (Q4_0) | 341 s | 2,313 | 6.8 | 28 | 34 | 7 of 7 |
| granite4.2:8b | 260 s | 2,554 | 9.8 | 31 | 44 | 6 of 7 |
| qwen3.5:9b (Q4_K_M) | 233 s | 2,227 | 9.5 | 32 | 32 | 7 of 7 |

All three listed non-entities such as "hansom" and "eleven shillings and sixpence".

Runs over chapters 1 and 2:

| Run | Model | Time | Calls | Fact-check result |
| --- | --- | --- | --- | --- |
| Local | gemma4:12b-it-qat | 636 s; 20,991 prompt and 4,160 output tokens | 4 | 1 claim moved later, none rejected |
| Cloud | claude-haiku-5-5 | 19 s; US$0.0026 | 4 | 6 rejected, 5 moved later |

At about 7 output tokens per second, the 4,160 output tokens account for roughly 590 of the 636 seconds. Generating output, not reading the prompt, sets the pace.

## 2. Runtime facts that change the picture

- **Ollama runs on llama.cpp, or on MLX.**
  - Since v0.30, Ollama runs GGUF models on llama.cpp's own server; v0.40.17 pins build b12257 ([v0.30.0][Ollama v0.30.0]).
  - Since v0.40, Apple Silicon uses MLX for supported architectures, including gemma4 and qwen3.5/3.6 ([v0.40.0][Ollama v0.40.0]).
- **Model-shipped parameters apply unless overridden.**
  - [`qwen3.5:9b`][qwen3.5] ships with `presence_penalty 1.5`, `temperature 1`, `top_k 20` and `top_p 0.95`.
  - [`gemma4:12b-it-qat`][gemma4] ships with `temperature 1`, `top_k 64` and `top_p 0.95`.
  - Request options override them, but only when sent.
- **The context window depends on VRAM** ([context length][Ollama context length]): 4K tokens below 24 GiB, 32K up to 48 GiB, and 256K above. The documented defaults (2,048 or 4,096) are stale.
  - The reference's 2,500-word part plus about 3,000 tokens of scaffolding needs about 7,000 tokens, so it is truncated or rejected below 24 GiB.
  - Ollama's OpenAI-compatible `/v1` endpoint cannot set `num_ctx`, and it ignores `max_completion_tokens`. Use the native `/api/chat` endpoint.
- **Prompt overflow is silent.**
  - With the defaults `truncate: true` and `shift: true`, Ollama keeps the first 4 tokens and the last half of the context, and drops the middle: the system prompt and the start of the passage. It only logs a warning.
  - Models with a vision projector, such as gemma4 and qwen3.5, cannot shift and fail instead.
- **The seed defaults to −1**, not the documented value. At temperature 0 it is unused anyway.

## 3. Parameters to send

On every call, using Ollama's native API:

| Parameter | Value | Why |
| --- | --- | --- |
| `temperature` | 0. A retry may use 0.3 with a new seed. | Greedy decoding for extraction and judging. |
| `top_k`, `top_p`, `min_p` | 40, 1.0, 0 | Neutral at temperature 0. Set explicitly so model defaults never apply above 0. |
| `repeat_penalty`, `presence_penalty`, `frequency_penalty` | 1, 0, 0 | Overrides shipped penalties that bias names and citations. |
| `num_ctx` | 8192 for parts of about 1,000 words; 16384 for whole chapters | Never rely on the VRAM-dependent default. |
| `num_predict` | Derived from the schema's bounds, such as 3072 for a 1,000-word extraction | Bounds runaway output. |
| `truncate`, `shift` | `false`, `false` | Overflow becomes an error instead of silent loss. Check the token budget in the client. |
| `think` | `false` | No measured gain for extraction, at two to five times the tokens. |
| `seed` | A hash of the part and the attempt | Unused at temperature 0; makes retries above 0 reproducible. |
| `keep_alive` | `30m` during a job | Avoids reloads, and keeps the prompt cache warm. |
| `format` | The JSON Schema | Grammar-constrained decoding. |

Per call type:

| Call | Temperature | Output limit | Schema notes |
| --- | --- | --- | --- |
| Roster of names | 0 | 1,024 | Names with type and first paragraph; up to about 60 items. |
| Extraction | 0 | From the schema's bounds | Integer paragraphs in [1, n], `maxItems` per list, text up to about 240 characters. |
| Gleaning ("only what you missed") | 0 | 768 | The extraction schema; lists may be empty. |
| Re-ask after a failed check | 0, then 0.3 with a new seed | Small | Only the failing items, with the exact error. |
| Fact-check with a dedicated checker | — | 1 or 2 tokens | The cited paragraphs, plus one either side, and one claim. |
| Fact-check with a general model | 0 | 1,536 | Evidence first, then the verdict. |
| Duplicate-entity judge | 0 | 512 | A reason, then the choice. |
| Recap from checked events | 0 | 1,024 | Up to 10 sentences of up to 220 characters. |

**Schemas for small models:**

- **Describe the schema in the prompt too.** Decoding is constrained by a grammar the model never sees ([grammars][llama.cpp grammars]).
- **Order keys alphabetically.** Ollama's native chat path sorts schema keys alphabetically ([#18717][Ollama #18717], still open), so name keys so that alphabetical order is the order they should be generated in (for example `a_evidence`, then `b_verdict`).
- **Evidence before verdicts.** [Answer-first schemas hurt small models most][Structure Tax], and the reference's verification schema is answer-first.
- **Bound everything.** Constrain citations with integer `minimum` and `maximum`, lists with `maxItems`, and text with `maxLength`. Out-of-range citations and runaway replies then become impossible.

**Server settings:**

- `OLLAMA_NUM_PARALLEL=1`, and serialise calls in the client;
- `OLLAMA_FLASH_ATTENTION=1` where the GPU supports it, and record the setting;
- `OLLAMA_KV_CACHE_TYPE=f16`;
- `OLLAMA_CONTEXT_LENGTH` equal to `num_ctx`;
- `OLLAMA_NOPRUNE=1`;
- `OLLAMA_MAX_LOADED_MODELS=2` or more, when the extractor and the checker both fit.

## 4. Determinism

Temperature 0 selects the most likely token every time and ignores the seed. Output still varies, because the token scores themselves vary with numerics:

| Source | Mechanism | Mitigation |
| --- | --- | --- |
| Prompt cache | A cache hit evaluates the rest of the prompt with a different batch shape than a cold start. Ollama always caches prompts. | A fixed request order from a fresh load, or `keep_alive: 0` per test request. |
| Batch shape | Kernels choose tiling by shape, so the order of reductions changes and near-ties flip. | Pin `num_batch`. |
| Parallel requests | The batch depends on neighbouring requests. | `OLLAMA_NUM_PARALLEL=1`. |
| Automatic tuning | Context size, batch size and layer placement depend on free VRAM at load time. | Set `num_ctx`, `num_batch` and `num_gpu` explicitly. |
| Kernels and precision | Flash attention, KV-cache quantisation and the CPU/GPU split pick different kernels. | Set them explicitly; run fully on the GPU. |
| Hardware and versions | Kernels are chosen per architecture, and Ollama updates llama.cpp in almost every release. | Pin and record everything; re-baseline after any change. |
| Model tags | A tag can be re-pushed with new weights or templates. | Record the model digest and template; set `OLLAMA_NOPRUNE=1`. |

Batch-invariant modes exist elsewhere, at a cost:

- **Thinking Machines Lab** ([blog][Thinking Machines]) showed the root cause is missing batch invariance. Qwen3-235B produced 80 distinct completions out of 1,000 at temperature 0; with batch-invariant kernels, all 1,000 were identical.
- **vLLM** offers `VLLM_BATCH_INVARIANT=1` (beta) on supported GPUs ([vLLM batch invariance]). Its documentation only says it "may impact performance"; one third-party measurement found a 22–35% throughput cost.
- **SGLang** offers `--enable-deterministic-inference`, at about 34% more latency on average ([SGLang deterministic inference]).
- **llama.cpp and Ollama** have no deterministic mode; llama.cpp has only draft pull requests.

On one machine, one request at a time, output is usually repeatable for an identical model, build, hardware, options, prompt and cache state, but that is not guaranteed. Nothing is guaranteed across machines, versions or cache states, and cloud models are not bit-reproducible either: Anthropic's API no longer accepts a temperature.

So "similar results" must be shown statistically:

- **Pin and record:** the Ollama version, the model digest and template, the GPU and driver, the server environment, and the full request with the exact schema bytes.
- **Measure stability:** run each case at least 10 times from a cold start (30 for release gates).
- **Report:**
  - the share of runs giving the most common output;
  - the number of distinct outputs;
  - agreement field by field.
- **Compare fields, not bytes.**

## 5. Spending time for quality

| Technique | Evidence | Extra time | Verdict |
| --- | --- | --- | --- |
| Parameter and schema hygiene | The mechanisms in sections 2 and 3. | None | Do first. |
| A dedicated grounded fact-checker | On [LLM-AggreFact], Bespoke-MiniCheck-7B scores 77.4 (CC BY-NC 4.0), Clef Flash 9B 76.6 (Apache-2.0), GPT-4o 75.9, Qwen2.5-7B 72.8 and Llama-3.1-8B 70.3. [MiniCheck] reaches GPT-4-level accuracy at 400 times lower cost. | Saves time: one forward pass per claim replaces generated verdicts. | The biggest win. Validate on about 200 hand-labelled fiction claims, since the benchmark is not fiction. |
| Mechanical validators, then one targeted re-ask | Self-correction without external feedback hurts ([Huang et al.][Self-correction]); feedback from tools helps ([CRITIC]). | The failure rate times one short call | Check copies of more than eight words, citation range, the reveal point, and that the cited paragraph names the entity. |
| Smaller parts, about 1,000 words | GraphRAG found almost twice as many entity references at 600 tokens as at 2,400 ([GraphRAG]); Llama-3.1-8B's effective context is about 1K tokens ([NoLiMa]). | 1–2 minutes per 2,500 words, less with a cached prefix | Do it; try 600, 1,200 and 2,500 words on the benchmark. |
| Two or three cached examples | Five examples raised Llama-3.1-8B's NER F1 by 7 to 12 points ([many-shot NER][Many-shot NER]). | Prompt tokens only | Do it, with examples from a different public-domain book. |
| One gleaning pass ("only what you missed") | More entities from gleaning ([GraphRAG], [SLIDE]); an evidence-based prune restores precision ([self-verification][Self-verification]). | 0.5–1 minute | Worth it for names and events; one pass only. |
| A roster pass, then extraction with the roster fixed | Decomposition helps large models; the gain is smaller for 7B models. | 0.5–1 minute | Keep it to two stages. |
| Several samples, or two models' union | Gains need a verifier, and the evidence is mostly from mathematics and code. | Two to three times the stage | Only for the cheap names pass. |
| Thinking | No gain found for information extraction; reasoning models can do worse at text understanding ([A-Eval-2.0][A-Eval]). | Two to five times the output | Off. |
| 8-bit instead of 4-bit weights | At most 0.3 benchmark points on Llama-3.1-8B ([quantization study][Quantization]). | 40–70% slower decoding where memory bandwidth limits | Not for the extractor; perhaps for a small checker. |
| A cloud model auditing borderline claims | Cascades match a large model's quality at a fraction of the cost ([FrugalGPT]). Use a different model family than the extractor. | No local minutes; cents per book | The best quality per minute, if a cloud call is allowed. |

## 6. Proving parity with a cloud baseline

- **Gold set:**
  - the reference's hand-written Sherlock Holmes index, plus three or four public-domain books of different shapes, such as Dracula (told in letters and diaries) and Pride and Prejudice (a large cast);
  - two or three chapters each, with about 200 labelled claims per book, seeded with the claims the cloud model rejected or moved.
- **Systems:** the cloud baseline and the local variants, on the same parts with the same validators.
- **Metrics:** the reference's five bake-off metrics, plus:
  - hard-rule violations in the final index;
  - entity precision and recall;
  - claim-support precision, and citation precision;
  - hindsight violations;
  - the checker's accuracy against human labels;
  - recap faithfulness;
  - stability across runs;
  - minutes and tokens per 10,000 words.
- **Statistics:**
  - pair results by part, and bootstrap confidence intervals over parts;
  - run each system at least three times;
  - report non-inferiority, not point wins.

Proposed acceptance thresholds, to be confirmed:

| Metric | Threshold |
| --- | --- |
| Hard-rule violations in the final index | 0 |
| Entity recall and precision | At least the cloud's, minus 5 points |
| Claim-support precision | At least 95%, and at least the cloud's minus 2 points |
| Citation precision | At least 95% |
| Hindsight violations | At most 0.5% of claims |
| Claim recall | At least 85% of the cloud's |
| The checker's "unsupported" verdicts | Recall at least 0.80, precision at least 0.70 |
| Recap sentences supported | At least 95% |
| Stability | At least 90% field agreement across 3 runs |
| Runtime | At most 3 times today's local minutes per 10,000 words |

Add the levers in this order, and keep each only if its measured gain is worth its minutes:

1. hygiene;
1. the checker;
1. part size;
1. examples;
1. gleaning;
1. the roster pass;
1. an optional cloud audit.

## 7. Recommendation

A local quality profile:

1. Pin the runtime; set the server environment and every request parameter of section 3.
1. Split books into parts of about 1,000 to 1,200 words, with a 150-word bridge used only as context. Keep a byte-identical prefix: the rules, a description of the schema, and two or three examples.
1. Run a roster pass, then extraction with the roster fixed.
1. Validate mechanically. Re-ask once with the exact error, then drop whatever still fails.
1. Run one gleaning pass, and validate again.
1. Check every claim with a dedicated checker. To catch claims revealed too early, re-check against only the paragraphs up to the cited one.
1. Write recaps from checked events, and check each sentence.
1. Merge duplicate entities: candidates come from rules, and a judge decides each pair.

## 8. Open questions

1. **Hardware:** Apple Silicon (how much memory) or NVIDIA (how much VRAM)? This decides the runner, the context tier, and whether the extractor and the checker fit together.
1. **Cloud calls:** none at all, a benchmark judge only, or an auditor for borderline claims?
1. **Checker licence:** is a non-commercial checker acceptable (Bespoke-MiniCheck, CC BY-NC 4.0), or Apache-2.0 only (Clef Flash 9B)?
1. **Budget:** how long may one book take (is overnight acceptable?), and who labels the gold claims?
1. **Regression tests:** bit-exact golden outputs, which need a pinned machine and a cold cache, or statistical stability?
1. **MLX:** does Ollama's MLX path honour integer bounds, `maxItems`, `maxLength` and the declared key order?
1. **Penalties:** how much does qwen3.5's shipped presence penalty hurt citation accuracy? This needs an A/B test.

## References

- Reference implementation at `9c7ebed`: [`llm.py`][llm.py], [`generate.py`][generate.py] and [`bench/`][bench].
- Runtimes: the [Ollama] repository, [releases v0.30.0][Ollama v0.30.0] and [v0.40.0][Ollama v0.40.0], [context length][Ollama context length] and [issue #18717][Ollama #18717]; the [llama.cpp server][llama.cpp server] and its [grammars][llama.cpp grammars]; [vLLM batch invariance]; [SGLang deterministic inference].
- Models: [qwen3.5], [gemma4], [bespoke-minicheck] and [clef-flash].
- Determinism: [Thinking Machines Lab][Thinking Machines].
- Quality: [LLM-AggreFact], [MiniCheck], [Structure Tax], [NoLiMa], [GraphRAG], [SLIDE], [many-shot NER][Many-shot NER], [self-verification][Self-verification], [CRITIC], [Huang et al.][Self-correction], [A-Eval-2.0][A-Eval], [quantization study][Quantization] and [FrugalGPT].

[llm.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/llm.py
[generate.py]: https://github.com/nightspine/lorekeeper/blob/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/src/lorekeeper/generate.py
[bench]: https://github.com/nightspine/lorekeeper/tree/9c7ebed838b7ecd553e358d470ff6096b3aafdf0/bench
[Ollama]: https://github.com/ollama/ollama/tree/v0.40.17
[Ollama v0.30.0]: https://github.com/ollama/ollama/releases/tag/v0.30.0
[Ollama v0.40.0]: https://github.com/ollama/ollama/releases/tag/v0.40.0
[Ollama context length]: https://github.com/ollama/ollama/blob/v0.40.17/docs/context-length.mdx
[Ollama #18717]: https://github.com/ollama/ollama/issues/18717
[llama.cpp server]: https://github.com/ggml-org/llama.cpp/blob/b12257/tools/server/README.md
[llama.cpp grammars]: https://github.com/ggml-org/llama.cpp/blob/b12257/grammars/README.md
[vLLM batch invariance]: https://docs.vllm.ai/en/latest/features/batch_invariance/
[SGLang deterministic inference]: https://docs.sglang.io/advanced_features/deterministic_inference.html
[qwen3.5]: https://ollama.com/library/qwen3.5:9b
[gemma4]: https://ollama.com/library/gemma4:12b-it-qat
[bespoke-minicheck]: https://ollama.com/library/bespoke-minicheck
[clef-flash]: https://ollama.com/library/clef-flash
[Thinking Machines]: https://thinkingmachines.ai/blog/defeating-nondeterminism-in-llm-inference/
[LLM-AggreFact]: https://llm-aggrefact.github.io/
[MiniCheck]: https://arxiv.org/abs/2404.10774
[Structure Tax]: https://arxiv.org/abs/2610.12056
[NoLiMa]: https://arxiv.org/abs/2502.05167
[GraphRAG]: https://arxiv.org/abs/2404.16130
[SLIDE]: https://arxiv.org/abs/2503.17952
[Many-shot NER]: https://arxiv.org/abs/2606.21890
[Self-verification]: https://arxiv.org/abs/2306.00024
[CRITIC]: https://arxiv.org/abs/2305.11738
[Self-correction]: https://arxiv.org/abs/2310.01798
[A-Eval]: https://arxiv.org/abs/2502.11164
[Quantization]: https://arxiv.org/abs/2601.14277
[FrugalGPT]: https://arxiv.org/abs/2305.05176
