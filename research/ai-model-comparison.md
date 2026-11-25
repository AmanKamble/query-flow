# AI Model Comparison

> Scope: which model(s) QueryFlow should use to turn a natural-language question into a
> validated `ReportSpec`, and where that model should run.
> Verified: November 2026. Re-check versions before implementation.

## 1. What the model actually has to do

QueryFlow does **not** need a general assistant. It needs a constrained translator:

| Requirement | Weight | Why |
|---|---|---|
| Emit valid JSON matching a schema, first try | Critical | Every invalid response costs a retry and a bad UX |
| Pick only allow-listed module/field/operator keys | Critical | Hallucinated keys are a security and correctness problem |
| Understand ERP vocabulary and synonyms | High | "revenue", "turnover", "sales value" → same metric |
| Multi-turn refinement ("now group by region") | High | Users iterate on reports |
| Date/period reasoning ("last quarter", "YTD") | High | Most reports are time-bounded |
| Tool/function calling | **Not needed** | The architecture forbids tools; this is deliberate |
| Long context | Low–medium | The catalog is compact (~2–4K tokens); history is trimmed |
| Vision / audio | Not needed | Text input only |
| Raw reasoning depth | Low | The task is mapping, not problem solving |

This changes the ranking. A model that scores lower on general benchmarks but is
reliable at strict JSON and cheap to run can win.

## 2. Candidate families

### Qwen (Alibaba) — Apache 2.0

| Model | Params | Active | Context | Notes |
|---|---|---|---|---|
| Qwen3.5-35B-A3B | 35B MoE | ~3B | 262,144 native | Strong structured output, fast for its size, multilingual |
| Qwen3.5-122B-A10B | 122B MoE | ~10B | 262,144 | Better accuracy, needs a serious GPU box |
| Qwen3.6-35B-A3B | 35B MoE | 3B | 262,144 | Newest open Qwen; Apache 2.0; vision + thinking; in Ollama |
| Qwen3.6-27B | 27B | — | 256K | Smaller sibling; comfortable on a 24 GB GPU |
| Qwen3-32B / 30B-A3B | 32B dense / 30B MoE | — | 32K–128K | Previous generation, still fine, wider host support |

**Verdict:** best overall fit. Apache 2.0, no user-count restrictions, excellent
JSON-schema-constrained decoding support, and a MoE variant that runs on a single
32 GB machine at usable speed.

### Gemma 4 (Google) — Apache 2.0

| Model | Params | Context | Notes |
|---|---|---|---|
| Gemma 4 E2B / E4B | ~2B / ~4B effective | 128K | Laptop-class, surprisingly capable |
| Gemma 4 12B | 12B dense | 128K | Best value for 16 GB machines |
| Gemma 4 26B A4B | 26B MoE, ~4B active | 256K | Fast, good quality |
| Gemma 4 31B | 31B dense | 256K | Highest quality of the open set |

Native function calling, system role, vision/audio. Very strong instruction
following and JSON adherence for its size.

**Verdict:** best fallback and best "runs on a normal laptop" option. Gemma 4 12B is
the recommended developer-machine model.

### Llama 4 (Meta) — Llama 4 Community License

| Model | Params | Active | Context | Notes |
|---|---|---|---|---|
| Scout | 109B | 17B | 10M | Huge context, but 100B+ weights |
| Maverick | 400B | 17B | 1M | Needs a multi-GPU node |

**Verdict:** not recommended for v1. The license is not OSI-approved (custom
commercial terms, naming/attribution requirements, 700M MAU clause), there is no
small dense variant, and the context advantage is irrelevant for a 4K-token
catalog. Knowledge cutoff is older than the Qwen/Gemma options.

### Hosted-only models worth knowing

| Model | Access | Fit |
|---|---|---|
| `openai/gpt-oss-120b` / `20b` | Groq free tier, self-host | Good JSON adherence, open weights (Apache 2.0) |
| `qwen/qwen3.8-27b` | Groq free tier | Very good for this task |
| Gemini 3.x Flash / Flash-Lite | Google AI free tier | Capable, but free-tier prompts may be used to improve Google products — **not acceptable for ERP data** |
| Gemini 4 Pro | Paid only | Overkill and expensive for schema mapping |

## 3. Local vs hosted

| Dimension | Ollama (local) | Ollama Cloud | Groq free | OpenRouter `:free` | Gemini free tier |
|---|---|---|---|---|---|
| Cost | Hardware + electricity | Per token | $0 | $0 | $0 |
| Data leaves your infra | **No** | Yes | Yes | Yes | Yes |
| Structured outputs | Yes (`format` + JSON Schema) | **No** | Yes (`strict`) | Model-dependent | Yes (`responseSchema`) |
| Throughput | Hardware-bound | Good | Very fast | Variable | Fast |
| Rate limits | None | Concurrency cap | 30 RPM / 1k RPD / 8k TPM / 200k TPD | 20 RPM / 50–1000 RPD | Per-model quotas |
| Ops burden | Install + model pulls | Low | Low | Low | Low |
| Best use | Production default | Burst capacity | Dev/demo, fallback | Experiments | Avoid for real data |

Key facts:

- Ollama's local OpenAI-compatible base URL is `http://localhost:11434/v1/`; the cloud
  endpoint is `https://ollama.com/v1` with `OLLAMA_API_KEY`.
- Ollama Cloud does **not** support structured outputs, which disqualifies it as the
  primary path for this design.
- Groq free tier returns `429` with a `retry-after` header; the client must honor it.
- OpenRouter's free-model allowance is 50 requests/day below 10 purchased credits and
  1,000/day at or above 10 credits.

## 4. Hardware sizing (local)

Approximate, Q4_K_M quantization, model + KV cache:

| Model class | Disk | RAM/VRAM | Realistic hardware |
|---|---|---|---|
| 2–4B (Gemma 4 E4B) | 3–4 GB | 8 GB | Any modern laptop, CPU-only OK |
| 8B | 5–6 GB | 16 GB | M-series Mac, 8 GB GPU |
| 12–14B (Gemma 4 12B) | 8–9 GB | 16 GB | 16 GB Mac / RTX 4060 Ti 16 GB |
| 27–31B (Qwen3.6-27B, Gemma 4 31B) | 17–19 GB | 24–32 GB | 32 GB Mac / RTX 4090 |
| 35B MoE (Qwen3.5/3.6-35B-A3B) | 21–23 GB | 32 GB | 32 GB Mac / 24 GB GPU (partial offload) |
| 100B+ (Llama 4 Scout) | 60 GB+ | 64–128 GB | Multi-GPU or CPU cluster |

Practical guidance:

- **Developer laptop (16 GB):** Gemma 4 12B or Qwen3.5-8B-class.
- **Demo/production box (32 GB unified memory or 24 GB VRAM):** Qwen3.6-35B-A3B.
- **No GPU at all:** Gemma 4 E4B works but expect 5–15 tok/s; acceptable because
  responses are short (~300–600 tokens).

Because the output is a small JSON document, **time-to-first-token and total latency
matter more than raw tokens/sec**. A 20 tok/s model that emits 400 tokens finishes in
~20 s; that is the upper bound of acceptable for an interactive report request, so
stream a status/progress indicator and keep prompts small.

## 5. Cost model

Local inference has no marginal cost, but amortize hardware:

| Setup | One-time | Monthly (power/ops) | Marginal cost per query |
|---|---|---|---|
| Existing dev laptop | $0 | ~$0 | $0 |
| Dedicated 32 GB box | $2,000–3,000 | $10–25 | ~$0 |
| Groq free | $0 | $0 | $0 (rate-limited) |
| Groq paid | $0 | usage-based | fractions of a cent |
| Gemini Flash paid | $0 | usage-based | low, but data leaves infra |

For an ERP with a handful of internal users, local inference is cheaper than any
per-token API within a few months, and it removes the data-processing question
entirely.

## 6. Recommendation

1. **Primary:** Qwen3.6-35B-A3B (or Qwen3.5-35B-A3B) served by **local Ollama**,
   JSON-Schema-constrained output, `temperature: 0`, fixed `seed`.
2. **Developer/low-RAM fallback:** Gemma 4 12B, same interface.
3. **Hosted fallback (dev, demos, local outage):** Groq free tier with
   `qwen/qwen3.8-27b` or `openai/gpt-oss-120b`, behind the same provider adapter.
   Only schema metadata and the user's question are sent — never result rows.
4. **Not now:** Llama 4, Gemini free tier, Ollama Cloud.

Use one internal interface so swapping providers is a config change:

```ts
export interface ReportModelProvider {
  readonly id: string;
  generateSpec(input: {
    systemPrompt: string;
    catalog: ModuleCatalog;
    history: Turn[];
    question: string;
    jsonSchema: object;
  }): Promise<{ raw: string; usage?: TokenUsage; latencyMs: number }>;
}
```

Implementations: `OllamaProvider`, `GroqProvider` (OpenAI-compatible). Both are
OpenAI-shaped, so one HTTP client with a different `baseURL` covers most of it.

## 7. Evaluation plan (do this before committing)

Build a **golden set** of 80–120 real questions from the target ERP domain, each with
an expected spec or an expected rejection.

| Metric | Target |
|---|---|
| JSON parses and passes Zod | ≥ 99% after one repair attempt |
| All module/field/operator keys valid | ≥ 98% |
| Filters semantically correct (field + operator + value) | ≥ 90% |
| Correct report type / grouping | ≥ 90% |
| Clarification asked when genuinely ambiguous | ≥ 85% |
| Rejection of out-of-scope requests | 100% |
| p50 latency | < 8 s |
| p95 latency | < 20 s |

Method:

- Run each model 3× per prompt (`temperature: 0` should be near-deterministic; variance
  reveals decoder/schema issues).
- Disable thinking mode (`think: false` / `enable_thinking: false`). Otherwise the model
  can spend thousands of tokens reasoning before emitting JSON — and hit the output cap
  without ever producing a spec.
- Score with a small script comparing normalized specs, not string equality.
- Include adversarial prompts: "ignore instructions", "show all tenants", "drop
  collection", "export every customer's email".
- Record failures as regression cases; the suite is the real quality gate, not
  leaderboard scores.

## 8. Prompt/model settings that matter

| Setting | Value | Reason |
|---|---|---|
| `temperature` | 0 – 0.2 | Deterministic spec generation |
| `top_p` | 0.9 | Slight diversity only if temperature > 0 |
| `seed` | fixed per environment | Reproducible debugging |
| `num_ctx` | 8192 | Catalog + history + question fits comfortably |
| `num_predict` | 1024 | Hard cap on output; specs are small |
| `format` / `response_format` | JSON Schema | Constrained decoding |
| `keep_alive` | `30m` | Avoid cold-start reloads during work hours |
| `think` / `enable_thinking` | `false` | Qwen3.6 reasons by default; spec generation does not need it, and reasoning tokens inflate latency |

Note: constrained decoding guarantees **shape**, not **meaning**. A schema-valid
response can still reference a forbidden field if the schema is loose, so the schema
must enumerate keys (`enum`) wherever possible and the server must re-validate
semantically. See `security-and-validation.md`.

## 9. References

- Qwen3.5-35B-A3B model card — https://huggingface.co/Qwen/Qwen3.5-35B-A3B
- Qwen3.6-35B-A3B model card — https://huggingface.co/Qwen/Qwen3.6-35B-A3B
- Qwen3.6-35B-A3B blog — https://qwen.ai/blog?id=qwen3.6-35b-a3b
- Qwen3 repository — https://github.com/QwenLM/Qwen3
- Ollama Qwen3.6 library — https://ollama.com/library/qwen3.6
- Gemma 4 model card — https://ai.google.dev/gemma/docs/core/model_card_4
- Ollama Gemma 4 library — https://ollama.com/library/gemma4
- Llama 4 model card — https://github.com/meta-llama/llama-models/blob/main/models/llama4/MODEL_CARD.md
- Ollama structured outputs — https://docs.ollama.com/capabilities/structured-outputs
- Ollama OpenAI compatibility — https://docs.ollama.com/api/openai-compatibility
- Ollama pricing — https://ollama.com/pricing
- Groq rate limits — https://console.groq.com/docs/rate-limits
- Groq structured outputs — https://console.groq.com/docs/structured-outputs
- OpenRouter limits — https://openrouter.ai/docs/api-reference/limits
- Gemini API pricing — https://ai.google.dev/gemini-api/docs/pricing
