---
layout: post
title: "Speculative Decoding: Making LLMs 2–3x Faster Without Breaking Anything"
date: 2026-03-08
description: Comparison diagram showing serial autoregressive decoding vs. parallel speculative decoding.
tags: LLM inference optimization speculative-decoding performance
giscus_comments: true
---

# Speculative Decoding: Making LLMs 2–3x Faster Without Breaking Anything

There's a real problem with running large language models: token generation is *slow*. Like, genuinely slow. The bigger the model, the longer each token takes because each one requires a full forward pass through billions of parameters.

But here's the thing—your GPU is *bored* most of the time. It can predict hundreds of tokens in parallel, but the algorithm forces it to do them one at a time. Speculative decoding is basically a hack to keep your GPU busy by having a tiny model make guesses about future tokens, then checking them with the big model all at once.

No smoke, no mirrors, no hallucination tax. Just pure throughput gains.

## The Core Idea: Draft Now, Verify Later

Here's how it works:

1. **Draft phase:** A tiny model (think 0.5B to 2B parameters) generates *k* tokens super fast. Like, while you're still reading this sentence.
2. **Verification phase:** The big model (7B, 13B, 70B, whatever) scores all *k* draft tokens in *parallel* in a single forward pass. This is the trick—instead of doing *k* separate forward passes, you do one.
3. **Accept or reject:** If a draft token matches what the big model thinks should come next (with high enough probability), you keep it. If not, you reject it, resample from the big model, and move on.

The output distribution is **identical** to normal autoregressive generation. You're not trading quality for speed. You're just being smarter about the computation.

Concrete example:

```
Input: "The capital of France is"

Normal generation (autoregressive):
  Token 1: "Paris" (1 forward pass through big model)
  Token 2: "." (1 forward pass)
  Total: 2 forward passes

Speculative decoding:
  Draft predicts: "Paris", ".", "It", "is"
  Big model scores all 4 tokens in ONE parallel pass
  Result: Keep "Paris" ✓, keep "." ✓, reject "It" ✗, resample
  Total: 1 big forward pass + 1 small draft pass (cheap) + 1 resample
  Speedup: ~2–3x depending on acceptance rate
```

## When Does It Actually Work?

Here's where people get excited and then disappointed. Speculative decoding isn't magic. It's a *conditional* speedup.

**Performance depends almost entirely on token acceptance rates.** If the draft model makes predictions that the big model agrees with 80% of the time, you get huge wins. If it only agrees 20% of the time, you're wasting GPU cycles on a draft model that's just dead weight.

Where it *does* work:

- **Generating repetitive or formulaic text** (code, JSON, structured logs). The draft model learns the pattern and nails the next few tokens.
- **Completing templates or boilerplate.** "Here's a function signature, now fill in the docstring."
- **Dialogue with consistent personalities.** If your model has settled into a rhythm, the draft model catches that.

Where it *doesn't* work:

- **Creative or unpredictable generation.** Poetry, fiction, or "surprise me" requests. The draft model guesses wrong constantly.
- **Low-latency, single-token scenarios.** If you only need one token, the overhead of running both models outweighs the gain.
- **Mismatched model pairs.** If your draft model is too weak (or too different in size), it'll give bad predictions and you'll reject tokens constantly.

## The Real Cost: Memory and Orchestration

Here's what the papers don't scream about: **running two models simultaneously is expensive.**

You're doubling your model memory footprint. If your 70B model barely fits on your GPU with attention caching, adding a 7B draft model means you're now at 77B effective model size. Attention caches stack up. Batch management gets complicated.

The vLLM team addressed this with **PagedAttention**—a technique that lets you evict and reload attention blocks dynamically, so you're not keeping everything in VRAM at once. But it's not free. It trades compute for memory, and memory bandwidth is often the bottleneck anyway.

So the honest take: speculative decoding is best when you're:
- Running in a data center with lots of GPU memory.
- Generating long sequences (so the fixed overhead of running both models is amortized).
- Using matryoshka-style model pairs—a fine-tuned smaller version of the same architecture family.

On a consumer GPU with limited VRAM? You might actually get *worse* performance because the memory swapping cost exceeds the decoding speedup.

## The Emerging Twist: Eliminate the Draft Model

Some teams are pushing back on the "run two models" approach. A newer direction, sometimes called **Speculative Streaming** or **Lookahead Decoding**, trains the *main* model itself to predict n-grams—i.e., the next 4–8 tokens all at once, rather than delegating to a separate draft model.

Advantages:
- No need to manage two models in production.
- All the training data and capacity goes into one model.
- Better alignment between prediction and verification (since they're the same model).

Disadvantage:
- Requires retraining your base model, which is... a big ask.
- Still experimental; unclear if it generalizes beyond specific architectures.

But the idea is sound: if you're building a model from scratch, baking in this multi-token prediction capability is smarter than bolting on a separate model at inference time.

## Should You Actually Use This?

If you're serving LLMs at scale, yes. Google, vLLM, and others report consistent 2–3x throughput gains in production. That's real money saved on GPU clusters.

If you're hacking on a local project? Probably not yet. The setup is fiddly. The gains depend heavily on your workload. And honestly, just using a smaller model (like a 7B instead of a 70B) often gets you better latency-quality tradeoffs than betting on speculative decoding.

The real value is for teams building inference engines—vLLM, SGLang, TensorRT-LLM, etc. They're baking speculative decoding in as an *option*, not a requirement. If your use case suits it, you get the speedup for free. If not, you just run normal generation.

That's the right engineering move: make the optimization invisible and conditional.