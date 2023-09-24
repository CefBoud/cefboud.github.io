---
title: "How Profitable is LLM Inference? Doing the Math on Kimi K3"
author: cef
date: 2026-07-29
categories: [Technical Writing, Open Source]
tags: [AI, Open Source, LLM, tokenomics]
render_with_liquid: false
description: "A look at LLM inference economics (batch size, GPU count, and the Pareto frontier that sets token prices) applied to Kimi K3 with back-of-the-envelope math."
---


<iframe
  class="embed-video"
  loading="lazy"
  src="https://www.youtube.com/embed/UkyQo3Zld9E"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>


## Intro

How profitable is LLM inference?

This is a central question sitting at the heart of the AI mania.

In this post, we'll cover token pricing basics, build up some intuition, then apply it to the recently released Kimi K3 to answer the profitability question.

## LLM Pricing TLDR

To produce tokens, we need GPUs and model weights. We load the weights onto one or many GPUs. Large frontier models won't fit on a single GPU: Kimi K3 needs 16 B200s or 8 B300s at minimum.

GPUs can be rented by the hour. Right now we can head to Lambda or Runpod and rent a B300 for ~$7/hour (at least 8 of those to run Kimi).

![runpod-b200-b300-gpu-price.png](/assets/runpod-b200-b300-gpu-price.png)

Tokens are billed by the million. Output tokens are what the model predicts, input tokens are the prompt. Both need processing: one forward pass can process all input tokens at once, while each output token needs its own pass (spec decoding changes this, though). Once input tokens are processed, they generate an internal state (KV cache) that gets reused, so it's never recomputed. Cached input tokens are cheapest: no GPU compute, just a load from cache (GPU VRAM, RAM, or even colder storage like SSD).

Here's Kimi K3 pricing on OpenRouter:
![kimi-k3-openrouter.png](/assets/kimi-k3-openrouter.png)

1M input tokens cost $3, output is $15, cached tokens are $0.30.

Where do these prices come from? Here's a simple model.

GPUs produce tokens. At the margin, some amount of GPU time produces some amount of tokens:

```
cost per M tokens = hourly_GPU_price / tokens_per_gpu_hour * 1,000,000
```

That's the whole business. Now the central question is: **what determines `tokens_per_gpu_hour`?**

## The b and N_GPU Situationship and Pareto Frontier

There are two crucial knobs:
- **batch size (b):** how many requests you process together through the same weights in the same forward pass
- **number of GPUs (N_GPU):** how many chips we spread one model across

```
token_latency = fn(b, N_GPU)
```

`token_latency` is a function of the batch size and number of GPUs. `token_latency` is how long, in seconds, one forward pass takes, the time to produce one new token for every sequence in the batch, i.e. `b` tokens.

We can then determine how many tokens a fleet of GPUs cranks out per hour:

```
tokens_per_gpu_hour = (b / token_latency) / N_GPU * 3600
```


We divide by `N_GPU` because that latency was bought with `N_GPU` chips at once, so each chip is only accountable for its share of the output.

One more crucial thing:

`token_latency` also determines speed. `1/token_latency` is how many tokens/second each user experiences: lower latency, faster things are. Usually, lowering `b` (batch size) at a fixed number of GPUs increases speed. So we trade throughput (total tokens) for speed. And fewer tokens per unit time means more expensive tokens.

Each `(b, N_GPU)` pair maps to a `(token_latency, cost)` pair, since together they set both speed/throughput and price.


`b` and `N_GPU` are in what you could call a situationship:

- Bigger `N_GPU` means more aggregate compute and memory bandwidth  -> bigger batch.
- Bigger `N_GPU` also means more aggregate memory, so more room for KV cache -> bigger batch.
- At the same time, bigger `N_GPU` means more network overhead, since coordination has to travel across all the GPUs -> higher latency.
- And bigger `b` means more data that has to move across the network -> higher latency.

The Pareto frontier is the set of `(b, N_GPU)` and their corresponding`(token_latency, cost)` pairs where you can't move one without making the other worse: slower or more expensive.

The cost/speed frontier is probably the key graph for determining profitability. Luckily, the[ vLLM announcement blog ](https://vllm.ai/blog/2026-07-27-k3#serving-performance)has one:

![pareto-frontier-vllm-kimi-k3.png](/assets/pareto-frontier-vllm-kimi-k3.png)

Each point on the frontier gives the optimal price for that speed. Looking at the OpenRouter pricing above, Fireworks offers two versions of Kimi: regular and fast, with fast costing $22 per million tokens instead of $15  but at higher tok/s (lower `token_latency`). They're likely running a smaller batch, or more GPUs for the same batch size, buying speed at a higher cost.

## How profitable is Kimi K3 inference

Kimi K3 is a ~2.8 trillion parameter model. Quantized to MXFP4, the weights come out to about 1.4TB: too big for one GPU. A B300 has 288GB of memory, 8 of them gives 2.3TB, enough for the weights plus KV cache headroom.

At the Pareto-optimal point on the cost/speed frontier, this setup gets roughly 30 tok/s for a single user, and about 1,500 tok/s of aggregate throughput per GPU (that's batching at work — no single user sees 1,500 tok/s, but each GPU serves that many tokens/sec across everyone combined).

```
tokens_per_gpu_hour = 1,500 * 3,600 = 5,400,000 tokens/hour/GPU
```

B300s rent for about $7.39/hour:

```
cost per M tokens = $7.39 / 5,400,000 * 1,000,000 ≈ $1.37 per million tokens
```

## Too good to be true

$1.37 per million tokens is very far from the advertised $15. Are inference providers laughing their way to the bank? Maybe a chuckle, not a laugh. Two big things dampen the joy:

- **Utilization won't be 100%.** You provision for peak demand, not average. Demand is bursty: coding models see less activity nights, weekends, holidays. The larger the provider, the more demand smooths out across many peaks. But assume 50% utilization and cost doubles immediately.
- **This is only marginal cost** the cost of producing extra tokens. There's also salaries, other IT costs, and most importantly, training and fine-tuning. A model is a depreciating asset. Open weights are everywhere right now, but someone shoulders the training cost, and that has to be accounted for somewhere.

The vLLM post mentions speculative decoding with DSpark giving a ~3x speedup which is tremendous and that's just one optimization vector. KV cache offloading, prefill/decode disaggregation, and others are actively being worked on. Any inference provider might be sitting on a secret edge that makes the economics far more lucrative than this back-of-the-envelope math suggests.
