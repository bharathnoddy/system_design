# ChatGPT

> LLM serving — inference at scale, GPU scheduling, streaming responses

## What to Study

- Transformer inference — autoregressive decoding, batching strategies (static vs continuous batching)
- KV cache — how key/value activations are cached across decoding steps and memory implications
- GPU cluster scheduling — job placement, tensor parallelism vs pipeline parallelism, MFU optimization
- SSE streaming — how token-by-token responses are streamed to the client over HTTP
- Rate limiting — token-bucket and sliding window approaches applied to per-user and per-org GPU budget

## Why This Matters for Platform Architecture

ChatGPT's serving infrastructure represents a new class of system design challenge: the unit of compute is a GPU, the unit of work is a variable-length sequence, and latency is dominated by memory bandwidth rather than CPU cycles. These constraints produce architecture patterns that differ substantially from classical web serving.

## Study Checklist

Use [TEMPLATE.md](../../TEMPLATE.md) as your study guide.

- [ ] Business context & scale documented
- [ ] Requirements defined
- [ ] Architecture diagram created
- [ ] Data model analyzed
- [ ] Key trade-offs articulated
- [ ] At least 2 primary sources read
- [ ] Failure modes analyzed
- [ ] Can compare to a previously studied system

## Status

Not started
