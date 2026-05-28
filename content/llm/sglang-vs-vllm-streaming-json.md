+++
title = 'Sglang vs Vllm Streaming Json'
date = 2026-05-28T14:53:13+08:00
draft = true
+++

# vLLM vs SGLang for gpt-oss-120B: Why “OpenAI-Compatible” Streaming JSON Still Needs a Compatibility Layer

Both vLLM and SGLang advertise OpenAI Chat Completions compatibility. In practice, once you enable `stream: true` against `gpt-oss-120b`, the SSE chunks are **not byte-identical**. Most of the differences are harmless, but two of them are subtle enough to silently corrupt client output if you move a parser from one engine to the other without adjustment.

This post turns a live capture into a practical compatibility guide. It shows the exact JSON differences, explains which ones are likely naming drift versus provider-specific extensions, and ends with a parser that safely handles both backends.


## Test setup

- **Model**: `openai/gpt-oss-120b`
- **Request mode**: OpenAI-compatible Chat Completions with `stream: true`
- **Request settings**: `max_tokens: 200`, `temperature: 0.1`
- **SGLang deployment in this repo**: `sglang:v0.5.12` with `--reasoning-parser gpt-oss` and `--tool-call-parser gpt-oss`
- **vLLM deployment in this repo**: `vllm:v0.21.0` with `--reasoning_parser openai_gptoss` and `--tool-call-parser openai`

The manifest evidence does not prove the runtime wire format by itself, but it does support the attribution of the captured streams as **SGLang-flavored** and **vLLM-flavored** OpenAI compatibility.

## TL;DR — the two fixes that actually matter

```python
# 1. The reasoning field has two names. Read both.
reasoning = delta.get("reasoning_content") or delta.get("reasoning") or ""

# 2. Don't break on finish_reason before consuming the delta.
#    vLLM can ship the last token + finish_reason in the same chunk.
for chunk in stream:
    delta = chunk.choices[0].delta

    if delta.content:
        text += delta.content

    if r := delta.get("reasoning_content") or delta.get("reasoning"):
        reasoning += r

    if chunk.choices[0].finish_reason:
        break
```

Everything else in this post is context for those two pieces of code.

## At a glance: where the streams differ

| Topic | vLLM-like stream | SGLang-like stream | Why it matters |
|---|---|---|---|
| `id` format | `chatcmpl-...` + UUID | bare 32-char hex | IDs should be treated as opaque strings |
| Reasoning field | `delta.reasoning` | `delta.reasoning_content` | Missing either one drops reasoning tokens |
| Finish behavior | last reasoning delta and `finish_reason` may share one chunk | captured run shows a final reasoning delta followed by a separate finish chunk | Wrong loop order can drop the final delta on vLLM |
| Stop metadata | `stop_reason` | `matched_stop` | Same concept, different keys |
| Extra per-chunk fields | `token_ids`, `prompt_token_ids`, `prompt_text`, `system_fingerprint` | fewer tracing fields | Strict schemas must allow unknown keys |
| Usage details | minimal usage object | `reasoning_tokens`, `prompt_tokens_details` | Billing and accounting logic must tolerate optional fields |

---

## 1. First chunk: same SSE envelope, different JSON shape

The very first chunk already shows the divergence.

**SGLang** declares nullable fields up front:

```json
{
  "id": "d3b406a9b33a435cb7a7bcc2266e48ac",
  "object": "chat.completion.chunk",
  "model": "openai/gpt-oss-120b",
  "choices": [{
    "index": 0,
    "delta": { "reasoning_content": null, "role": "assistant", "content": "" },
    "logprobs": null,
    "finish_reason": null,
    "matched_stop": null
  }]
}
```

**vLLM** emits a leaner object and adds top-level token-trace fields:

```json
{
  "id": "chatcmpl-6ca2ec78-dac2-4759-8ffc-aa13d8b470bf",
  "object": "chat.completion.chunk",
  "model": "openai/gpt-oss-120b",
  "choices": [{
    "index": 0,
    "delta": { "role": "assistant", "content": "" },
    "logprobs": null,
    "finish_reason": null
  }],
  "prompt_token_ids": null,
  "prompt_text": null
}
```

Two practical consequences fall out immediately:

- **Do not assert `id.startswith("chatcmpl-")`**. That works on many OpenAI-style systems, but it breaks on SGLang here.
- **Do not overfit a strict schema to one backend**. Both engines add fields the other one does not use.

---

## 2. Reasoning deltas: same idea, different field names

This is the easiest silent migration bug to introduce.

**SGLang** uses `reasoning_content`:

```json
{ "delta": { "reasoning_content": "We" }, "finish_reason": null, "matched_stop": null }
{ "delta": { "reasoning_content": " need" }, "finish_reason": null, "matched_stop": null }
{ "delta": { "reasoning_content": " to" }, "finish_reason": null, "matched_stop": null }
```

**vLLM** uses `reasoning`:

```json
{ "delta": { "reasoning": "We" }, "finish_reason": null, "token_ids": null }
{ "delta": { "reasoning": " need" }, "finish_reason": null, "token_ids": null }
{ "delta": { "reasoning": " to" }, "finish_reason": null, "token_ids": null }
```

If your client only reads one name, **all reasoning tokens can vanish** when you switch engines.

For these captures, the safest interpretation is that `reasoning_content` vs `reasoning` is a **functionally equivalent naming difference**. vLLM's public reasoning-output guidance treats `reasoning` as the current name and `reasoning_content` as a deprecated compatibility naming. Current OpenAI-compatible docs still do not provide a stable, universal spec for raw streamed reasoning fields, so tolerant parsing is the only safe choice.

---

## 3. Final chunk semantics: where parsers silently lose data

The most dangerous difference is not the reasoning field name. It is the **termination pattern**.

**In this SGLang capture**, the last reasoning token is followed by a separate finish frame:

```json
{ "delta": { "reasoning_content": " IDs" }, "finish_reason": null, "matched_stop": null }
{ "delta": { "reasoning_content": null }, "finish_reason": "length", "matched_stop": null }
```

**vLLM** can merge the last reasoning token and `finish_reason` into one chunk:

```json
{
  "delta": { "reasoning": "STATE" },
  "finish_reason": "length",
  "stop_reason": null,
  "token_ids": null
}
```

That makes this loop wrong:

```python
if chunk.choices[0].finish_reason:
    break
text += chunk.choices[0].delta.content or ""
```

It looks correct under SGLang, but on vLLM it can drop the final delta in the same chunk. The safe rule is simple: **consume the delta first, then inspect `finish_reason`**.

This is also where you see another provider-specific split:

- **vLLM** exposes `stop_reason`
- **SGLang** exposes `matched_stop`

Treat those as analogous backend-specific stop metadata, not guaranteed semantic equivalents and not stable OpenAI-standard fields.

---

## 4. The `usage` chunk: both compatible, not equally rich

In this run, both engines end with a usage-only chunk with `choices: []`, but the payloads are still different.

**SGLang** exposes reasoning accounting explicitly:

```json
{
  "choices": [],
  "usage": {
    "prompt_tokens": 2677,
    "total_tokens": 2877,
    "completion_tokens": 200,
    "prompt_tokens_details": null,
    "reasoning_tokens": 200
  }
}
```

**vLLM** keeps the usage object minimal and adds `system_fingerprint` at the chunk level:

```json
{
  "choices": [],
  "usage": {
    "prompt_tokens": 2674,
    "total_tokens": 2874,
    "completion_tokens": 200
  },
  "system_fingerprint": "vllm-0.1.dev1+gc06ff9ec0-tp2-59a10424"
}
```

Three details matter here:

- In this SGLang capture, `reasoning_tokens` is already reflected inside `completion_tokens`. Do not double-count it in client-side accounting unless your provider documents otherwise.
- `system_fingerprint` leaks backend build and topology detail. Strip it if you do not want to expose runtime metadata externally.
- The `prompt_tokens` counts differ by three tokens (`2677` vs `2674`) even for the same logical request. Do not assume token accounting will match exactly across engines.

---

## 5. Which differences are naming drift, and which are provider behavior?

The cleanest way to think about these captures is:

### Likely naming drift

- `delta.reasoning_content` vs `delta.reasoning`

### Safer to treat as provider-specific extensions

- `matched_stop` vs `stop_reason`
- `token_ids`, `prompt_token_ids`, `prompt_text`
- `system_fingerprint`
- `reasoning_tokens`, `prompt_tokens_details`
- merged-vs-separate finish frame behavior
- `id` formatting differences

That distinction matters because it changes the right client strategy. Naming drift means “read both.” Provider extensions mean “accept if present, ignore if absent.”

---

## 6. The reasoning itself is not reproducible across engines

Even with the same model and nearly deterministic settings (`temperature: 0.1`), the reasoning traces are not the same.

- **SGLang-style capture**: `"We need to produce JSON with reasoning_steps,."`
- **vLLM-style capture**: `"We need to extract signals: tables: ..."`

That is not surprising. Different engines change enough of the runtime path to perturb generation. Likely contributors include:

- different attention kernels
- different tensor-parallel topology
- different batching and paged-attention implementations

The operational takeaway is simple: **if you are running quality A/B tests, lock the engine as well as the model**. Otherwise you are partly measuring engine artifacts.

---

## 7. A minimal parser for the captured variants

```python
def parse_chat_stream(stream):
    content, reasoning, finish_reason, usage = "", "", None, None

    for chunk in stream:
        # Final usage chunk has empty choices on both engines
        if not chunk["choices"]:
            usage = chunk.get("usage", {})
            continue

        choice = chunk["choices"][0]
        delta = choice.get("delta", {})

        # Dual-field reasoning — required for cross-engine correctness
        r = delta.get("reasoning_content") or delta.get("reasoning")
        if r:
            reasoning += r

        if delta.get("content"):
            content += delta["content"]

        # Collect first, THEN record finish_reason
        if choice.get("finish_reason"):
            finish_reason = choice["finish_reason"]

    return {
        "content": content,
        "reasoning": reasoning,
        "finish_reason": finish_reason,
        "usage": {
            "prompt_tokens": usage.get("prompt_tokens", 0),
            "completion_tokens": usage.get("completion_tokens", 0),
            "reasoning_tokens": usage.get("reasoning_tokens", 0),
        },
    }
```

This parser does not branch on provider identity. It simply tolerates the field and finish-frame differences that matter in the captured variants.

---

## 8. Migration checklist

| Priority | Change | If you skip it |
|:---:|---|---|
| **P0** | Read both `reasoning_content` and `reasoning` | All reasoning tokens can be dropped |
| **P0** | Consume delta before checking `finish_reason` | Final token can be lost on vLLM |
| P1 | Allow extra keys and optional reasoning fields in the schema | Strict validators reject one backend |
| P1 | Stop asserting `id.startswith("chatcmpl-")` | Hard failure on SGLang |
| P2 | Read `stop_reason` and `matched_stop` as optional equivalents | You lose stop diagnostics |
| P2 | Treat `system_fingerprint` and `reasoning_tokens` as optional | `AttributeError` or accounting bugs |

The two P0 items are the real compatibility boundary. Everything else is cleanup and resilience.

---

## Appendix A: verbatim raw SSE excerpts from the live capture

The full captures are long and repetitive, so this appendix keeps the blog readable by showing **representative raw lines from one live run exactly as captured**. The vLLM excerpt includes `system_fingerprint`; redact it if you do not want to publish backend build metadata.

### A.1 First chunk

**vLLM-style capture**

```text
data: {"id":"chatcmpl-6ca2ec78-dac2-4759-8ffc-aa13d8b470bf","object":"chat.completion.chunk","created":1779866853,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"role":"assistant","content":""},"logprobs":null,"finish_reason":null}],"prompt_token_ids":null,"prompt_text":null}
```

**SGLang-style capture**

```text
data: {"id":"d3b406a9b33a435cb7a7bcc2266e48ac","object":"chat.completion.chunk","created":1779866808,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning_content":null,"role":"assistant","content":""},"logprobs":null,"finish_reason":null,"matched_stop":null}]}
```

### A.2 Early reasoning chunks

**vLLM-style capture**

```text
data: {"id":"chatcmpl-6ca2ec78-dac2-4759-8ffc-aa13d8b470bf","object":"chat.completion.chunk","created":1779866853,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning":"We"},"logprobs":null,"finish_reason":null,"token_ids":null}]}
data: {"id":"chatcmpl-6ca2ec78-dac2-4759-8ffc-aa13d8b470bf","object":"chat.completion.chunk","created":1779866853,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning":" need"},"logprobs":null,"finish_reason":null,"token_ids":null}]}
data: {"id":"chatcmpl-6ca2ec78-dac2-4759-8ffc-aa13d8b470bf","object":"chat.completion.chunk","created":1779866853,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning":" to"},"logprobs":null,"finish_reason":null,"token_ids":null}]}
```

**SGLang-style capture**

```text
data: {"id":"d3b406a9b33a435cb7a7bcc2266e48ac","object":"chat.completion.chunk","created":1779866808,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning_content":"We"},"logprobs":null,"finish_reason":null,"matched_stop":null}]}
data: {"id":"d3b406a9b33a435cb7a7bcc2266e48ac","object":"chat.completion.chunk","created":1779866808,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning_content":" need"},"logprobs":null,"finish_reason":null,"matched_stop":null}]}
data: {"id":"d3b406a9b33a435cb7a7bcc2266e48ac","object":"chat.completion.chunk","created":1779866808,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning_content":" to"},"logprobs":null,"finish_reason":null,"matched_stop":null}]}
```

### A.3 Terminal reasoning and finish frames

**vLLM-style capture**

```text
data: {"id":"chatcmpl-6ca2ec78-dac2-4759-8ffc-aa13d8b470bf","object":"chat.completion.chunk","created":1779866853,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning":"STATE"},"logprobs":null,"finish_reason":"length","stop_reason":null,"token_ids":null}]}
```

**SGLang-style capture**

```text
data: {"id":"d3b406a9b33a435cb7a7bcc2266e48ac","object":"chat.completion.chunk","created":1779866809,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning_content":" IDs"},"logprobs":null,"finish_reason":null,"matched_stop":null}]}
data: {"id":"d3b406a9b33a435cb7a7bcc2266e48ac","object":"chat.completion.chunk","created":1779866809,"model":"openai/gpt-oss-120b","choices":[{"index":0,"delta":{"reasoning_content":null},"logprobs":null,"finish_reason":"length","matched_stop":null}]}
```

### A.4 Final usage chunk and stream terminator

**vLLM-style capture**

```text
data: {"id":"chatcmpl-6ca2ec78-dac2-4759-8ffc-aa13d8b470bf","object":"chat.completion.chunk","created":1779866853,"model":"openai/gpt-oss-120b","choices":[],"usage":{"prompt_tokens":2674,"total_tokens":2874,"completion_tokens":200},"system_fingerprint":"vllm-0.1.dev1+gc06ff9ec0-tp2-59a10424"}
data: [DONE]
```

**SGLang-style capture**

```text
data: {"id":"d3b406a9b33a435cb7a7bcc2266e48ac","object":"chat.completion.chunk","created":1779866809,"model":"openai/gpt-oss-120b","choices":[],"usage":{"prompt_tokens":2677,"total_tokens":2877,"completion_tokens":200,"prompt_tokens_details":null,"reasoning_tokens":200}}
data: [DONE]
```
