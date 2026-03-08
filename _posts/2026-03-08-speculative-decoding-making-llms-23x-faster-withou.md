---
layout: post
title: Speculative Decoding: Making LLMs 2–3x Faster Without Breaking Anything
date: 2026-03-08
description: Technical diagram comparing standard autoregressive decoding (one token per forward pass) vs. speculative decoding (draft + parallel verify for 2-3x speedup)
tags: LLM inference optimization speculative-decoding performance vLLM SGLang
giscus_comments: true
---

There's a real problem with running large language models: token generation is *slow*. Like, genuinely slow. The bigger the model, the longer each token takes because each one requires a full forward pass through billions of parameters.

But here's the thing—your GPU is *bored* most of the time. It can predict hundreds of tokens in parallel, but the algorithm forces it to do them one at a time. Speculative decoding is basically a hack to keep your GPU busy by having a tiny model make guesses about future tokens, then checking them with the big model all at once.

No smoke, no mirrors, no hallucination tax. Just pure throughput gains.

{%- include figure.html path="assets/img/blog/speculative-decoding-making-llms-23x-faster-withou/01_illustration.png" alt="Standard vs. Speculative Decoding" class="img-fluid rounded z-depth-1" -%}

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

## Model Pairing Strategies: Beyond "Just Use a Small Model"

Not all draft models are created equal. Your acceleration gains hinge entirely on how well your draft and target models agree. Here are the common strategies teams are actually using in production:

### Same-Family Matryoshka Pairing

This is the gold standard: take the *same* architecture, just smaller. Example: target model is **Llama 3.1 70B**, draft model is **Llama 3.1 8B**.

**Why it works:**
- Both models have the same tokenizer, embedding space, and architectural biases.
- The draft model learned the same patterns at a smaller scale, so predictions align naturally.
- Acceptance rates often hit 75–85%, which is where speculative decoding really sings.

**Downside:** You need to find or fine-tune a smaller sibling. Not every model family has open-weight smaller versions.

### Quantized Draft Models

Instead of a smaller model, use a **quantized version of the target model itself**. E.g., target is **Llama 3.1 70B full precision**, draft is **Llama 3.1 70B INT8 or INT4**.

**Why it works:**
- Identical architecture and token predictions (mostly).
- Quantization is cheap—same model weights, just lower precision.
- Fits on even a single smaller GPU.

**Downside:**
- Quantization introduces small divergences in predictions. Acceptance rates drop to ~60–70%.
- Less speedup than same-family smaller models, but still 1.5–2x.
- You're essentially running the same model twice (once in low precision, once in high), which is weird but sometimes makes sense on a tight budget.

### Cross-Family Draft Models

Pair your big model with a *different architecture* that's just fast. Example: target is **Llama 3.1 70B**, draft is **Qwen 2.5 7B**.

**Why you'd try it:**
- Qwen is vectorized and optimized for speed; it might be faster per token than a Llama 8B.
- If they're close in capability level, prediction agreement can still hit 60–75%.

**Why it often disappoints:**
- Different tokenizers mean different token boundaries. Misalignment tanks acceptance rates.
- Different training data means different "next token" intuitions. High disagreement = low speedup.
- Real-world results: expect 40–60% acceptance rates and 1.2–1.8x speedup. Not terrible, but weaker than same-family.

**When it works:** Mostly when you have a very fast, very capable draft model available and no access to smaller siblings of your target.

### Speculative Models Trained Specifically for Drafting

Some teams go further and train a model *specifically to be a good draft partner*. This is rare in open-source but shows up in papers and closed APIs.

**Strategy:**
- Collect a dataset of (draft prediction, target model prediction) pairs.
- Fine-tune or distill the draft model to maximize agreement on *that* specific pair.
- Acceptance rates can hit 80–90%.

**Cost:** Requires training / fine-tuning. Only practical if you're running speculative decoding at massive scale and the infrastructure investment pays off.

## Adaptive Acceptance Thresholds and Tuning

Once your models are paired, you need to decide: **when does the draft model's prediction count as "good enough" to accept?**

The naive approach: just accept if the draft model's token probability is above some threshold, say 0.8. The smarter approach: make that threshold *adaptive* based on context.

### Dynamic Temperature Scaling

If the draft model is very confident (high probability on its top token), accept it eagerly. If it's uncertain (low-probability distribution), be strict.

Example:
- Draft model says "the" with p=0.92 → accept unconditionally.
- Draft model says "the" with p=0.55 → check if target model also prefers "the".
- Draft model says "the" with p=0.15 → reject and resample.

**Implementation:** Multiply the draft logits by a dynamic temperature parameter that adjusts per token based on entropy or probability mass.

### Workload-Specific Thresholds

Different tasks accept different risks:

- **Code generation:** Be strict. A wrong token in a function call breaks the whole thing. Use threshold 0.75+.
- **Narrative text:** Be lenient. "The" vs. "A" doesn't sink the ship. Use threshold 0.5+.
- **JSON / structured output:** Medium-strict (0.65+). One misplaced bracket ruins parsing.

**Implementation:** Let users specify per-request acceptance thresholds, or auto-detect by monitoring the request type.

### Measuring and Tuning in Production

Trace the key metrics:

```
acceptance_rate = (accepted_tokens / total_draft_tokens)
speedup = (time_normal_generation / time_speculative_generation)
effective_speedup_threshold = ~0.6 acceptance => ~1.5x speedup
```

If your acceptance rate drops below 50%, your draft model is too weak and you're better off just running the target model. Bump it or find a better pair.

## Batch-Level vs. Sequence-Level Speculation

Speculative decoding can be applied at different granularities:

### Sequence-Level: One Batch, One Active Sequence

This is the classic approach: the draft model generates tokens for *one* sequence while the target model verifies them all at once.

```
Batch size: 1 sequence, up to K draft tokens
Draft: Seq1 → tokens 1–K
Target: Verify Seq1 tokens 1–K in single forward pass
```

**Pros:**
- Simple to implement.
- Memory usage is predictable (target + draft both running on the same sequence).

**Cons:**
- Only one sequence can use speculative decoding at a time.
- If you have a batch of 32 sequences, you're running speculative decoding for seq 1, then normal generation for seqs 2–32.
- Batch utilization is poor.

### Batch-Level: Multiple Sequences, Speculative Decoding in Parallel

More advanced schedulers (like those in SGLang) apply speculative decoding to *multiple sequences in the same batch*:

```
Batch size: 32 sequences
Draft phase: Draft model generates tokens for all 32 seqs in parallel
Target phase: Target model verifies all 32 seqs' draft tokens in one huge batch
```

**Pros:**
- Much better GPU utilization.
- Amortizes the draft model's overhead across many sequences.
- Real-world speedup scales better.

**Cons:**
- Requires a scheduler that can track which tokens were accepted vs. rejected per sequence.
- Memory usage is higher (one batch slot per sequence, even if they're at different generation lengths).
- Harder to implement correctly.

**Realistic impact:** 
- Sequence-level: 1.5–2x speedup if your acceptance rate is 70%+.
- Batch-level with good scheduling: 2–3x speedup on the same acceptance rate, because you're not bottlenecked on draft model throughput.

## Practical Implementation: vLLM and SGLang

Enough theory. Here's how to actually use speculative decoding in two major inference frameworks.

### vLLM: Enable Speculative Decoding

vLLM baked speculative decoding into its core serving logic. Enabling it is straightforward:

**Step 1: Prepare Your Model Pair**

You need two models:
- **Target model:** The big one you care about (e.g., Llama 3.1 70B).
- **Draft model:** A smaller version (e.g., Llama 3.1 8B).

Both should be compatible architectures. Download them to local disk:

```bash
# Download target
huggingface-cli download meta-llama/Meta-Llama-3.1-70B-Instruct \
  --local-dir ./models/llama-70b

# Download draft
huggingface-cli download meta-llama/Meta-Llama-3.1-8B-Instruct \
  --local-dir ./models/llama-8b
```

**Step 2: Launch vLLM with Speculative Decoding**

```bash
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Meta-Llama-3.1-70B-Instruct \
  --tensor-parallel-size 4 \
  --speculative-model meta-llama/Meta-Llama-3.1-8B-Instruct \
  --num-speculative-tokens 5 \
  --speculative-draft-tensor-parallel-size 1
```

**Breakdown:**
- `--model`: Target model (70B).
- `--tensor-parallel-size 4`: Shard the target model across 4 GPUs.
- `--speculative-model`: Draft model (8B).
- `--num-speculative-tokens 5`: Draft model generates up to 5 tokens before verification.
- `--speculative-draft-tensor-parallel-size 1`: Draft model runs on 1 GPU (it's small).

**Step 3: Call It Like Normal**

From the client side, nothing changes. The API is 100% compatible with OpenAI format. Using the modern OpenAI SDK:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="dummy"  # vLLM requires a fallback key string
)

response = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3.1-70B-Instruct",
    messages=[
        {"role": "user", "content": "Write a function to compute Fibonacci."}
    ],
    max_tokens=256,
)

print(response.choices[0].message.content)
```

Speculative decoding runs silently in the background.

**Step 4: Monitor Performance**

vLLM's metrics endpoint logs speculative decoding stats:

```bash
curl http://localhost:8000/metrics | grep speculative
```

Look for:
- `speculative_decoding_accepted_tokens`: How many draft tokens the target model agreed with.
- `speculative_decoding_rejected_tokens`: How many it rejected.
- Overall `speculative_decoding_acceptance_rate`: Should be >50% to see real speedup.

### SGLang: Batch-Level Speculative Decoding

SGLang takes a different approach: it offers **more fine-grained control** over speculative decoding and makes batch-level usage the default.

**Step 1: Install and Set Up**

```bash
pip install sglang[all]
```

**Step 2: Start SGLang Server with Speculation**

```bash
python -m sglang.launch_server \
  --model-path meta-llama/Meta-Llama-3.1-70B-Instruct \
  --speculative-draft-model-path meta-llama/Meta-Llama-3.1-8B-Instruct \
  --num-speculative-tokens 8 \
  --disable-flashinfer 0
```

**Key flags:**
- `--speculative-draft-model-path`: Path to your draft model.
- `--num-speculative-tokens 8`: Verify up to 8 draft tokens in one shot (higher than vLLM's default).
- `--disable-flashinfer 0`: Use Flash Attention for both draft and target (faster).

**Step 3: Use SGLang's Python API**

SGLang's API is slightly more opinionated but gives you more control:

```python
import sglang as sgl

@sgl.function
def code_gen(s, prompt):
    s += prompt
    s += sgl.gen(
        "response",
        max_new_tokens=256,
        temperature=0.7,
    )

# Launch the SGLang engine
sgl.set_default_backend(sgl.RuntimeEndpoint("http://localhost:30000"))

result = code_gen(
    prompt="Write a Python function to check if a number is prime."
)
print(result["response"])
```

**What's Different:**
- SGLang uses a **functional DSL** to define generation workflows.
- You can interleave Python logic with LLM calls, and SGLang batches them efficiently.
- Speculative decoding happens *across entire batches* of workflow executions, not just individual requests.

**Example: Batch-Level Dispatch**

```python
@sgl.function
def batch_code_gen(s, prompts):
    """Generate code for multiple prompts in a single batch."""
    responses = []
    for prompt in prompts:
        s += f"\nPrompt: {prompt}"
        s += sgl.gen(
            f"response_{len(responses)}",
            max_new_tokens=128,
        )
        responses.append(s[f"response_{len(responses)}"])
    return responses

# SGLang will schedule all generations in a single batch,
# with speculative decoding applied across the entire batch.
prompts = [
    "Write a sort function.",
    "Write a hash function.",
    "Write a search function.",
]

results = batch_code_gen(prompts=prompts)
for prompt, result in zip(prompts, results):
    print(f"{prompt}\n{result}\n---")
```

SGLang's scheduler automatically:
1. Batches all three code_gen calls together.
2. Runs the draft model on all three in parallel.
3. Verifies all three's draft tokens in one target model forward pass.
4. Achieves near-3x speedup (if acceptance rate is >70%).

### Tuning for Your Hardware

Both frameworks expose tuning knobs. Here's a practical tuning process:

**For a single 80GB GPU (e.g., H100):**

```bash
# vLLM: Run 70B target on 1 GPU, 8B draft on same GPU
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Meta-Llama-3.1-70B-Instruct \
  --tensor-parallel-size 1 \
  --speculative-model meta-llama/Meta-Llama-3.1-8B-Instruct \
  --num-speculative-tokens 5 \
  --speculative-draft-tensor-parallel-size 1
```

Run a quick throughput test:

```python
import time
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="dummy"
)

start = time.time()
for i in range(10):
    response = client.chat.completions.create(
        model="meta-llama/Meta-Llama-3.1-70B-Instruct",
        messages=[{"role": "user", "content": "Count to 100."}],
        max_tokens=256,
    )
elapsed = time.time() - start
print(f"10 requests in {elapsed:.2f}s = {10 / elapsed:.1f} req/s")
```

If you see >5 requests/sec, speculative decoding is working. If <2, your acceptance rate is too low; try a better draft model or adjust the threshold.

**For multi-GPU setups (4–8 GPUs):**

Run the target model on 2–4 GPUs and the draft on 1 GPU. Adjust `--num-speculative-tokens` based on memory—higher is better (8–16) if you have room.

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