---

title: "Deep Dive into Efficient LLM Inference with nano-vLLM"
author: cef
date: 2026-04-05
categories: [Technical Writing, Open Source]
tags: [AI, Open Source, Opencode, LLM, vLLM]
render_with_liquid: false
description: "A look inside a lightweight implementation of vLLM. KV cache, paged attention, tensor parallelism  &multi-GPU support, etc."

---


<iframe
  class="embed-video"
  loading="lazy"
  src="https://www.youtube.com/embed/DNrIu_EZz5k"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>



## Intro
vLLM is an open source LLM inference engine, and it’s an impressive piece of software.

In this post, we’ll look at **nano-vLLM**: a lightweight reimplementation that keeps the essential ideas. It can run a Qwen model with **paged attention**, supports **multi-GPU tensor parallelism**, and is optimized to handle multiple requests efficiently with a very approchable codebase.

## Quick start

```sh
# install nano-vllm
git clone https://github.com/GeeeekExplorer/nano-vllm.git
cd /root/nano-vllm/
# fix some quirky bug with rope_scaling default param (dict instead of None)
sed -i 's/rope_scaling=rope_scaling/rope_scaling=None/' nanovllm/models/qwen3.py
pip install .

# install HF CLI
curl -LsSf https://hf.co/cli/install.sh | bash
# add it to path e.g. :
export PATH="/root/.local/bin:$PATH"
# download model weights
export model=Qwen3-0.6B 
hf download --force-download "Qwen/$model"  --local-dir "/root/huggingface/$model/"
```

Run the example in a Python REPL:

```py
from nanovllm import LLM, SamplingParams

llm = LLM("/root/huggingface/Qwen3-0.6B/", enforce_eager=True, tensor_parallel_size=1)
sampling_params = SamplingParams(temperature=0.6, max_tokens=256)

prompts = ["The sky was"]
outputs = llm.generate(prompts, sampling_params)
outputs[0]["text"]
```

## Why Inference Engines like vLLM?

At a high level, an LLM is “just” a Python class with a `forward()` method.

We define a model class, load weights, and those weights are just tensors—numbers. We plug them into the model and call `forward()`. For each input token ID, we get logits—scores over the next token. We turn those into probabilities, sample a token, append it autoregressively, and repeat until a stopping condition is met (EOS or max tokens / context length).

PyTorch makes this clean and straightforward. So why do we need an inference engine?

Well… because the cool kids use one and we want to be cool!

But more seriously, inference engines solve a number of important problems. Let’s walk through some essential ones.

#### 1. The KV Cache

Because of their architecture and autoregressive nature (feeding outputs back as inputs), LLMs end up recomputing the same work many times. For token 2, we need the K and V vectors (from all attention blocks) for tokens 0 and 1. For token 3, we need them for tokens 0, 1, and 2, and so on.

Naively, this means rerunning large parts of the network repeatedly, including expensive matrix multiplications in the FFN layers. But the only place we actually need information from previous tokens is in the attention blocks—specifically, the K and V vectors.

So instead of recomputing them, we cache them.

By storing the K/V tensors for each previous token, we avoid recomputing them and only run the newly generated token through the network. All other layers—including the expensive FFNs—are effectively skipped for tokens with cached KV. This dramatically reduces compute.

#### 2. Memory Fragmentation

Caching introduces a new problem: memory management.

We don’t know in advance how long a generated sequence will be. So how much memory should we allocate for the KV cache?

* If we allocate for the maximum possible sequence length, we waste memory through internal fragmentation (waste within allocated chunks).
* If we allocate incrementally as needed, we risk uneven gaps, leading to external fragmentation.

This is where *paged attention*—introduced by the creators of vLLM—comes in. It manages KV cache memory in fixed-size blocks (“pages”), avoiding fragmentation while allowing flexible growth. Memory waste drops from more than 50% to less than 5%. This means more requests can be served, increasing throughput.

#### 3. Continuous Batching

If we load a 14B or 70B model and run inference on a single request, we end up with very low arithmetic intensity—the ratio of FLOPs to memory access (i.e., how many operations we run per byte accessed). That’s the holy grail of GPU efficiency.

However, if we run inference for many requests with the same weights, we still load the weights only once but perform more computation. This increases arithmetic intensity. So one thing is clear: batching is good.

But naive batching isn’t enough. With static batching, we bundle N requests together, run them, and wait until the longest request finishes. That’s inefficient. If one request finishes at token 1,000 and another at 50,000, the first sits idle for 49,000 steps—that’s a lot of waste.

Additionally, requests arrive at different times, so an urgent request might be stuck waiting for a batch to complete.

Inference engines provide much more flexibility. Batching happens at the step level: we continuously add new requests, remove completed ones, and can even preempt low-priority requests. This leads to much better compute and memory utilization, increasing overall throughput.

#### 4. Prefill vs. Decode

There are two distinct phases in LLM inference:

* **Prefill**: When the prompt is first submitted, we run all prompt tokens through the model to compute and store the KV cache. This phase is compute-bound.
* **Decode**: After prefill, with the KV cache stored, we generate tokens one at a time. This phase is less compute-bound and more sensitive to latency.

These phases have different characteristics and benefit from different optimizations (e.g., specialized attention kernels). Inference engines are aware of this and handle them accordingly.


Beyond solving these problems, inference engines also provide features like multi-GPU support, chunked prefill, and speculative decoding. Each fascinating in its own right.


## Architecture
From the example above, you can see `LLM` is the entry point. You instantiate it with a model path and options—most notably `tensor_parallel_size`, which controls whether weights are split across multiple GPUs (useful when a model can’t fit on a single GPU). We’ll come back to that.

The path points to weights in `safetensors` format, and nano-vLLM loads them into `Qwen3ForCausalLM` (`models/qwen3.py`). Conceptually:

```py
class Qwen3ForCausalLM:
    def forward(input_ids, pos):
        x = Embedding(input_ids)          # token -> vector
        for _ in layers:
            x = x + Attention(x, pos)     # self-attention (needs KV cache)
            x = x + MLP(x)                # nonlinear feature transform
        x = FinalNorm(x)                  # stabilize outputs
        return LMHead(x)                  # logits over vocab (then softmax + sampling)
```

The Qwen3-0.6B weights we’re using are on Hugging Face [here](https://huggingface.co/Qwen/Qwen3-0.6B?show_file_info=model.safetensors).

Calling `Qwen3ForCausalLM(input_ids, pos)` and then sampling is “LLM inference”. vLLM-style engines add the bells and whistles that make this efficient at scale.

## The `generate()` loop
The entry point is `generate()`. Here’s a trimmed down snippet:

```py
class LLMEngine:  # really the engine
    def generate(
        self,
        prompts: list[str] | list[list[int]],
        sampling_params: SamplingParams | list[SamplingParams], ...
    ) -> list[str]:
        if not isinstance(sampling_params, list):
            sampling_params = [sampling_params] * len(prompts)

        for prompt, sp in zip(prompts, sampling_params):
            self.add_request(prompt, sp)

        outputs = {}
        while not self.is_finished():
            t = perf_counter()
            output, num_tokens = self.step()
            for seq_id, token_ids in output:
                outputs[seq_id] = token_ids
                if use_tqdm:
                    pbar.update(1)

        outputs = [outputs[seq_id] for seq_id in sorted(outputs.keys())]
        outputs = [{"text": self.tokenizer.decode(token_ids), "token_ids": token_ids} for token_ids in outputs]
        return outputs

    def add_request(self, prompt: str | list[int], sampling_params: SamplingParams):
        if isinstance(prompt, str):
            prompt = self.tokenizer.encode(prompt)
        seq = Sequence(prompt, sampling_params)
        self.scheduler.add(seq)
```

So what’s happening?

Prompts come in with `sampling_params` (temperature to dial “creativity”, `max_tokens` to cap output length). Each prompt becomes a request: tokenized and wrapped into a `Sequence`.

`LLMEngine` holds a `Scheduler`, which has two queues (`deque`): it tracks sequence state and decides what runs next.

```py
class Scheduler:
    def __init__(self, config: Config):
        self.max_num_seqs = config.max_num_seqs
        self.max_num_batched_tokens = config.max_num_batched_tokens
        self.eos = config.eos
        self.block_manager = BlockManager(config.num_kvcache_blocks, config.kvcache_block_size)
        self.waiting: deque[Sequence] = deque()
        self.running: deque[Sequence] = deque()

    def add(self, seq: Sequence):
        self.waiting.append(seq)

    def is_finished(self):
        return not self.waiting and not self.running
```

`add_request()` pushes into `waiting`. The main loop keeps stepping as long as there are sequences in either queue.

A prompt/sequence starts in `waiting` (needs prefill). Once admitted, it moves into `running` and is decoded token-by-token. That “life cycle” happens inside `LLMEngine.step()`:

```py
def step(self):
    seqs, is_prefill = self.scheduler.schedule()
    token_ids = self.model_runner.call("run", seqs, is_prefill)
    self.scheduler.postprocess(seqs, token_ids)
    outputs = [(seq.seq_id, seq.completion_token_ids) for seq in seqs if seq.is_finished]
    num_tokens = sum(len(seq) for seq in seqs) if is_prefill else -len(seqs)
    return outputs, num_tokens
```

This is the beating heart of the engine:

- `schedule()` picks the next sequences and tells us whether we’re doing **prefill** or **decode** this step.
- `model_runner` runs the model.
- `postprocess()` updates sequences (append token, stop on EOS, etc.).

In newer vLLM versions, prefill and decode can be mixed in the same step. In nano-vLLM, we keep them separate for simplicity.

Blog post over. Let’s call it a day.

Still here? Ok. Let’s keep going.

### Scheduler

```py
def schedule(self) -> tuple[list[Sequence], bool]:
    # prefill
    scheduled_seqs = []
    num_seqs = 0
    num_batched_tokens = 0
    while self.waiting and num_seqs < self.max_num_seqs:
        seq = self.waiting[0]
        if num_batched_tokens + len(seq) > self.max_num_batched_tokens or not self.block_manager.can_allocate(seq):
            break
        num_seqs += 1
        self.block_manager.allocate(seq)
        num_batched_tokens += len(seq) - seq.num_cached_tokens
        seq.status = SequenceStatus.RUNNING
        self.waiting.popleft()
        self.running.append(seq)
        scheduled_seqs.append(seq)
    if scheduled_seqs:
        return scheduled_seqs, True

    # decode
    while self.running and num_seqs < self.max_num_seqs:
        seq = self.running.popleft()
        while not self.block_manager.can_append(seq):
            if self.running:
                self.preempt(self.running.pop())
            else:
                self.preempt(seq)
                break
        else:
            num_seqs += 1
            self.block_manager.may_append(seq)
            scheduled_seqs.append(seq)
    assert scheduled_seqs
    self.running.extendleft(reversed(scheduled_seqs))
    return scheduled_seqs, False
```

Before talking about `schedule()`, it helps to say what it’s trying to guarantee.

We’re scheduling because **KV cache is the scarce resource**. With paged attention, we don’t give each request one big contiguous KV buffer—we give it a **block table** (a list of fixed-size KV blocks, default `kvcache_block_size=256`). That reduces fragmentation and enables **prefix caching**: if two sequences share identical full blocks, they can share the same KV blocks (via hashes/refcounts).

nano-vLLM also separates **prefill** vs **decode** at the scheduler level:

- `waiting`: sequences that haven’t been admitted (or got preempted)
- `running`: sequences that are admitted and can be decoded token-by-token

`Scheduler.schedule()` returns `(scheduled_seqs, is_prefill)`.

#### Prefill path
In prefill, we keep pulling from `waiting` (leftmost / oldest) and batch as many prompts as we can.

We stop if we’d exceed:

- `max_num_seqs`
- `max_num_batched_tokens`
- or we can’t allocate KV blocks for a sequence: `block_manager.can_allocate(seq)`  
  (meaning KV cache is tied up by sequences already in `running`, so we decode those first)

When we accept a waiting sequence:

- `block_manager.allocate(seq)` assigns it a `seq.block_table` (and can reuse cached blocks if hashes match)
- `num_batched_tokens += len(seq) - seq.num_cached_tokens`

That last line matters: if prefix-cache hits happen, some prompt tokens are already “in KV”, so we only count the *new* (uncached) tokens we actually need to prefill in this step.

If we scheduled at least one waiting sequence, we immediately return `(scheduled_seqs, True)`. Prefill is prioritized.

#### Decode path
If no prefills were scheduled, we decode from `running`.

Decode here means: one forward pass per sequence for exactly one token position. We run the model on `seq.last_token` and sample the *next* token. The actual `seq.append_token(token_id)` happens later in `postprocess()`, not in `schedule()`.

So in decode, the scheduler’s job is:

1) pick up to `max_num_seqs` sequences to decode this step  
2) make sure each selected sequence has a valid KV slot for the token we’re about to run  
3) preempt other sequences if we don’t have enough free KV blocks  

`can_append(seq)` is:

```py
return len(self.free_block_ids) >= (len(seq) % self.block_size == 1)
```

`(len(seq) % block_size == 1)` becomes `1` or `0`.

- If `len(seq) % block_size != 1`, decode for this sequence does **not** require allocating a brand new KV block right now, so we’re fine even with `0` free blocks.
- If `len(seq) % block_size == 1`, the sequence’s current last token sits at **position 0 of a new block** (typically right after `postprocess()` appended a token that crossed a block boundary). In that case, we **must** allocate one new block so the model can write KV for that token.

This is tightly coupled to `ModelRunner.prepare_decode()`: it builds `slot_mapping` using `seq.block_table[-1]` and `seq.last_block_num_tokens`. If you don’t allocate the new block before decode, there’s nowhere to map that KV write.

#### Preemption policy
In the decode loop we take `seq = self.running.popleft()` (oldest first). If `can_append(seq)` is false, we try to free blocks by preempting:

- If there are other running sequences, we preempt `self.running.pop()` (the **newest**) first.
- If `seq` is the only one left, we preempt `seq` itself and give up on it for this step.

So it’s basically protecting old sequebces and sacrificing new ones under memory pressure.

Preemption sets status back to `WAITING`, calls `block_manager.deallocate(seq)` (frees its KV blocks / drop its block table), then pushes it to the *front* of `waiting` so it gets retried soon.

#### KV block bookkeeping during decode
Once `can_append(seq)` is satisfied, the scheduler calls `block_manager.may_append(seq)` right before scheduling the sequence for decode.

`may_append` is basically “do any KV block-table bookkeeping needed for the sequence”:

- If `len(seq) % block_size == 1`:
  - allocate one new KV block
  - append its id to `seq.block_table`
- If `len(seq) % block_size == 0`:
  - the last block is now full, so compute its hash and record it in `hash_to_block_id`
  - this is what makes full blocks eligible for prefix caching later
- Else:
  - nothing to allocate or hash yet

Finally, after collecting `scheduled_seqs`, the scheduler pushes them back onto the left of `running` (preserving order) so they remain “active” in the next rounds.


### KV Cache
The Attention block is the **crucial** part of LLMs that enables the next token to be predicted based on all the previous tokens in a sequence.

Each token is mapped to:

* 1 or multiple Query vectors (1 in Multi-Head Attention (MHA) and multiple in Grouped Query Attention (GQA))
* 1 Key vector
* 1 Value vector

To calculate the output of token *t*, we need to know the K and V vectors of all tokens [0, t−1]. We can naively calculate this during each inference run, or we can store the K and V vectors for all the previous tokens in what we'll call the KV cache.

For a model with 13B parameters, we can have:

* 40 layers
* 40 attention heads
* 128-dimensional head size
* FP16 precision

Each token's KV cache requires:
2 (K and V) × 40 (layers) × 40 (heads) × 128 (dim) × 2 (bytes for FP16) = 800 KB per token

For larger models, we're talking MBs per token, and for a sequence with thousands of tokens, we need GBs of VRAM just for the KV cache.

Now let's suppose we need to allocate memory for a request. Say we know the maximum sequence length is 2k. We can simply allocate 2k × 800 KB, which is ~1.6 GB, and call it a day. But if our request generates only a few hundred tokens, more than half the space would have been wasted. That's what's referred to as **internal fragmentation**.

Okay, say we decide to avoid that and allocate only what we need in smaller chunks, growing it when needed. Each request would occupy a variable-length chunk of memory, but once it's completed, it would be cleared. After a while, we end up with more available memory, but it won't be contiguous, which means it's not really usable. That's what's referred to as **external fragmentation**. 

This very same problem of variable-size memory allocation has been solved by operating systems when allocating RAM for various processes. The solution is virtual memory and paging.



A process only sees contiguous memory, referred to as virtual memory. Behind the scenes, the OS divides the actual memory into blocks (pages) of fixed size, such as 4 KB or 8 KB. Each page from the process’s virtual memory is mapped to a physical block and freed when finished. The physical pages/blocks don’t have to follow any order or be contiguous. So virtual block 1 could map to physical block 233, and virtual block 2 could map to physical block 4.

Paging almost completely solves internal fragmentation. At most, we waste less than a page. There’s no external fragmentation because any virtual page can be satisfied by any available physical page.

Now, the same exact technique can be applied to the KV cache. We split the contiguous memory of the KV cache into blocks, each block contains the K and V vectors for `block_size` tokens (say, 16 or 256). The added complexity is maintaining a mapping for each sequence between its virtual blocks and the corresponding physical blocks.

Dividing the KV cache into blocks enables a very interesting capability: **prefix caching**.

In practice, a lot of requests share the same first tokens, for example the same system prompt (and often the same tool defs). For those early tokens, the KV cache is identical. So instead of having every request recompute the KV vectors for that shared prefix, we compute it once, store it, and reuse it.

Concretely, after we compute the KV cache for some prefix, we keep a reference to the corresponding KV blocks, and we index them in a lookup table keyed by a hash of the prefix tokens. When a new request comes in, we hash its prefix and check the table. If there’s a match, we can reuse the already-computed blocks and start attention from there, skipping all the work for that prefix.

One subtle but important detail: we can’t just hash a single block in isolation. To be correct, we need the *entire prefix* to match, and the hash for a given block needs to reflect the tokens that came before it (because the block represents a specific position range in a specific sequence). So the key is effectively a chained hash over the prefix, not a per-block content hash i.e. 

```
# recursive relationship
Hash_Block(0) = Hash(tokens[0] + -1)
Hash_Block(i+1) = Hash(tokens[i+1] + Hash_Block(i))
```



This turns out to work pretty well, because shared prefixes are extremely common in real assistant traffic.

#### Implementation of Paged Attention

When we initialize our `ModelRunner` (the class responsible for loading the model and running inference), we call `allocate_kv_cache`:

```py
    def allocate_kv_cache(self):
        config = self.config
        hf_config = config.hf_config
        free, total = torch.cuda.mem_get_info()
        used = total - free
        peak = torch.cuda.memory_stats()["allocated_bytes.all.peak"]
        current = torch.cuda.memory_stats()["allocated_bytes.all.current"]
        num_kv_heads = hf_config.num_key_value_heads // self.world_size
        head_dim = getattr(hf_config, "head_dim", hf_config.hidden_size // hf_config.num_attention_heads)
        block_bytes = 2 * hf_config.num_hidden_layers * self.block_size * num_kv_heads * head_dim * hf_config.torch_dtype.itemsize
        config.num_kvcache_blocks = int(total * config.gpu_memory_utilization - used - peak + current) // block_bytes
        assert config.num_kvcache_blocks > 0
        self.kv_cache = torch.empty(2, hf_config.num_hidden_layers, config.num_kvcache_blocks, self.block_size, num_kv_heads, head_dim)
        layer_id = 0
        for module in self.model.modules():
            if hasattr(module, "k_cache") and hasattr(module, "v_cache"):
                module.k_cache = self.kv_cache[0, layer_id]
                module.v_cache = self.kv_cache[1, layer_id]
                layer_id += 1
```

There’s a lot going on here, so let’s break it down. First, we compute how many bytes a single KV “block” (page) needs. Each block holds `block_size` tokens. For each token, we store KV cache data for all `num_hidden_layers`, each layer has `num_kv_heads` heads, and each head has dimension `head_dim`. The size of one block is therefore:

`2 * num_hidden_layers * block_size * num_kv_heads * head_dim * dtype_size`

The leading `2` is because we store both **K** and **V**, which have the same shape. `hf_config.torch_dtype.itemsize` gives the number of bytes per element (e.g. FP16 = 2 bytes, INT8 = 1 byte).

We can control how much GPU memory to reserve for KV cache via `gpu_memory_utilization` (default `0.9`), leaving some headroom instead of consuming all available memory.

Next, we compute how many blocks we can fit by dividing the estimated available memory by `block_bytes`. The available space is roughly `total * util - used`, where `used` accounts for memory currently used in the GPU, this comprises model weights, CUDA kernels and libraries buffers, etc.

Right before calling `allocate_kv_cache`, we call `warmup_model()`, which runs inference at the maximum batch size and sequence length. This forces the model to allocate the memory it needs at full capacity, so the remaining budget can be used for the KV cache.

In addition to the currently `used` GPU memory,  we subtract `(peak - current)`  i.e. `- peak + current`. Here, `current` and `peak` come from Pytorch’s own stats (not the whole cuda GPU): `current` is what Pytorch is currently using, and `peak` is the historical maximum. Substracting the difference is saying, let's assume Pytorch might reach its previous peak which is higher than the current usage, let's keep enough space for that.

Once we know `num_kvcache_blocks`, we can allocate our cache:

`self.kv_cache = torch.empty(2, hf_config.num_hidden_layers, config.num_kvcache_blocks, self.block_size, num_kv_heads, head_dim)`

For K and V, for each hidden layer, we allocate `num_kvcache_blocks` blocks, each of which holds `block_size * num_kv_heads * head_dim` values.

The loop at the end is extremely interesting. For any module that has `k_cache` and `v_cache`—and only the Attention module does—we assign `self.kv_cache[0, layer_id]` to `k_cache` and `self.kv_cache[1, layer_id]` to `v_cache`. This is very important: it means a “block” is not a single contiguous chunk across the whole model. Block `i` is split across multiple layers (one slice per layer). Also, within a given layer, consecutive blocks are `block_size * num_kv_heads * head_dim` elements apart.  Block `i` and Block `i+i` are however contiguous within the same layer. 

If we’re using multiple GPUs, each GPU is only responsible for a subset of the KV heads (`num_kv_heads = num_key_value_heads // world_size`).

This memory layout—and the fact that a KV cache block spans multiple layers in a non-contiguous way—was one of the most surprising things to me. I’m not sure why I assumed it had to be contiguous.

The Attention module is as follows:

```py
class Attention(nn.Module):

    def __init__(self, num_heads, head_dim, scale, num_kv_heads):
        super().__init__()
        self.num_heads = num_heads
        self.head_dim = head_dim
        self.scale = scale
        self.num_kv_heads = num_kv_heads
        self.k_cache = self.v_cache = torch.tensor([])

    def forward(self, q: torch.Tensor, k: torch.Tensor, v: torch.Tensor):
        context = get_context()
        k_cache, v_cache = self.k_cache, self.v_cache
        if k_cache.numel() and v_cache.numel():
            store_kvcache(k, v, k_cache, v_cache, context.slot_mapping)

        # call flash_attn
```

Notice that `k_cache` and `v_cache` are initialized with a dummy tensor first, and they’ll be set later in `allocate_kv_cache`. This Attention module is called after the module that calculates q, k, and v. So during each model run, new k and v vectors are available, and we add them to the cache via `store_kvcache`.

Let’s step back for a second. We need to store the newly calculated k and v in our cache, and we said the cache is paged. So how do we know where each new k and v should go? We said we’re mapping virtual contiguous blocks to physical blocks.

Each sequence has a `block_table` attribute, which is basically a list. If `block_table = [22, 4, 43]`, this means virtual block 0 maps to physical block 22, virtual block 1 to physical block 4, and virtual block 2 to physical block 43. Let’s say each block has 256 tokens (that’s nano-vllm’s default).

Where should the KV cache for token 257 in this sequence be stored? Token 257 is in virtual block 1, so physical block 4, and it’s the second token in that block. So: physical block 4, index 1.

So, knowing the virtual-to-physical mapping (via `block_table`), we can locate the physical location of a token’s KV cache. This mapping is computed and made available in `slot_mapping`, which is the last argument passed to `store_kvcache` and tells it where to store the newly calculated K and V data.

Once we understand how the virtual-to-physical mapping and block size are used, the following code used to populate `slot_mapping` becomes easy to grasp:

```py
    def prepare_prefill(self, seqs: list[Sequence]):
        # ...
        slot_mapping = []
        for seq in seqs:
            for i in range(seq.num_cached_blocks, seq.num_blocks):
                start = seq.block_table[i] * self.block_size
                if i != seq.num_blocks - 1:
                    end = start + self.block_size
                else:
                    end = start + seq.last_block_num_tokens
                slot_mapping.extend(list(range(start, end)))
        set_context(..., slot_mapping=slot_mapping, ...)
```

`seq.block_table[i]` grabs the physical block storing virtual block `i`. We multiply that by `block_size` to get the physical slot/index for the first token in the block, then we use `range` to populate the rest sequentially. Notice the `if` that checks whether we reached the last block, which might not be full, so we determine its end position using `seq.last_block_num_tokens`.

For decode, it’s even simpler, because we only have 1 slot per sequence to calculate:

```py
    def prepare_decode(self, seqs: list[Sequence]):
        # ...
        slot_mapping = []
        for seq in seqs:
            slot_mapping.append(
                seq.block_table[-1] * self.block_size + seq.last_block_num_tokens - 1
            )
        # ...
        set_context(..., slot_mapping=slot_mapping, ...)
```

`prepare_prefill` and `prepare_decode` are called right before the model run and provide the necessary `slot_mapping` to store KV values in the cache by putting that info in a global context, which is retrieved in the Attention module via `get_context()`.


So we know the physical slot for each token’s KV cache, and we need to write the token’s key/value vectors into that slot. To do this efficiently, we go a layer below regular PyTorch tensor ops and use Triton for low-level GPU manipulation. Triton sits between high-level PyTorch and low-level C/C++ CUDA.

```py
@triton.jit
def store_kvcache_kernel(
    key_ptr,
    key_stride,
    value_ptr,
    value_stride,
    k_cache_ptr,
    v_cache_ptr,
    slot_mapping_ptr,
    D: tl.constexpr,
):
    idx = tl.program_id(0)
    slot = tl.load(slot_mapping_ptr + idx)
    if slot == -1:
        return

    key_offsets = idx * key_stride + tl.arange(0, D)
    value_offsets = idx * value_stride + tl.arange(0, D)
    key = tl.load(key_ptr + key_offsets)
    value = tl.load(value_ptr + value_offsets)

    cache_offsets = slot * D + tl.arange(0, D)
    tl.store(k_cache_ptr + cache_offsets, key)
    tl.store(v_cache_ptr + cache_offsets, value)


def store_kvcache(
    key: torch.Tensor,
    value: torch.Tensor,
    k_cache: torch.Tensor,
    v_cache: torch.Tensor,
    slot_mapping: torch.Tensor,
):
    N, num_heads, head_dim = key.shape
    D = num_heads * head_dim

    assert key.stride(-1) == 1 and value.stride(-1) == 1
    assert key.stride(1) == head_dim and value.stride(1) == head_dim
    assert k_cache.stride(1) == D and v_cache.stride(1) == D
    assert slot_mapping.numel() == N

    # Launch N programs: one program per token
    store_kvcache_kernel[(N,)](
        key,
        key.stride(0),
        value,
        value.stride(0),
        k_cache,
        v_cache,
        slot_mapping,
        D=D,
    )
```

The Python function `store_kvcache` dispatches `N` `store_kvcache_kernel` programs on the GPU, where `N` is the number of tokens. Each program handles one token: it reads that token’s freshly-computed key/value vectors and writes them into the physical cache slot specified by `slot_mapping`.

- `idx = tl.program_id(0)` is the token index within the `key`/`value` tensors.
- `slot = tl.load(slot_mapping_ptr + idx)` loads the physical slot for token `idx`. If `slot == -1`, we skip the write.
- `idx * key_stride + tl.arange(0, D)` computes the offsets for the `D` elements of token `idx`’s key (and similarly for value).
- `cache_offsets = slot * D + tl.arange(0, D)` computes the destination offsets inside the cache for that physical slot.
- `tl.store(k_cache_ptr + cache_offsets, key)` and `tl.store(v_cache_ptr + cache_offsets, value)` write the vectors into the cache.

In simple terms, we’re copying the key/value data from one GPU memory location to another, `N` times (once per token) using the slot mapping to decide where each token’s KV should live physically.


#### Flash attention

The KV data is stored in the cache. Now, we need to actually compute the attention. Keep in mind that since the cache is stored in a paged fashion, the attention function needs to support that.

The distinction is actually very explicit in the attention block:

```py
    def forward(self, q: torch.Tensor, k: torch.Tensor, v: torch.Tensor):
        # ...
        if context.is_prefill:
            if context.block_tables is not None:    # prefix cache
                k, v = k_cache, v_cache
            o = flash_attn_varlen_func(q, k, v,
                                       max_seqlen_q=context.max_seqlen_q, cu_seqlens_q=context.cu_seqlens_q,
                                       max_seqlen_k=context.max_seqlen_k, cu_seqlens_k=context.cu_seqlens_k,
                                       softmax_scale=self.scale, causal=True, block_table=context.block_tables)
        else:    # decode
            o = flash_attn_with_kvcache(q.unsqueeze(1), k_cache, v_cache,
                                        cache_seqlens=context.context_lens, block_table=context.block_tables, 
                                        softmax_scale=self.scale, causal=True)
        return o
```

`flash_attn_varlen_func` and `flash_attn_with_kvcache` are two methods from the famous [Flash Attention library](https://github.com/dao-ailab/flash-attention). The TLDR of Flash Attention is that it’s a very clever and efficient way to compute attention: it avoids materializing the full attention matrix, so the *attention computation* uses memory that scales roughly linearly with sequence length instead of the naive quadratic scaling.

But even with Flash Attention, prefill (`varlen_func`) and decode (`with_kvcache`) behave differently and come with different assumptions. `flash_attn_varlen_func` is typically compute-heavy: in prefill we have a lot of query positions to compute attention for (and this same path is also used during training, which is out of scope for an inference engine). `flash_attn_with_kvcache` is called during decode: we usually have very few new queries (often just one token), and we care a lot about latency, at that point the bottleneck is often reading K/V efficiently from the cache.

Flash Attention is its own repository and has very optimized CUDA kernels. It supports paging: notice the `block_tables` provided in the context. When `block_tables` is provided, the Flash Attention functions know this is paged KV, and assume that the `k_cache` and `v_cache` args (notice that `varlen_func` is given `k` and `v` but it’s actually `k_cache` and `v_cache`) refer to the full paged cache memory, with `block_table` telling the kernel how to interpret it for each sequence.

Conceptually, if a sequence’s block table is `[4, 18, 3, ...]`, this means logical blocks `0, 1, 2, ...` map to physical cache blocks `4, 18, 3, ...`.

`prepare_blocks` that populates `block_tables` is called in both `prepare_prefill` and `prepare_decode`:

```py

    def prepare_block_tables(self, seqs: list[Sequence]):
        max_len = max(len(seq.block_table) for seq in seqs)
        block_tables = [seq.block_table + [-1] * (max_len - len(seq.block_table)) for seq in seqs]
        block_tables = torch.tensor(block_tables, dtype=torch.int32, pin_memory=True).cuda(non_blocking=True)
        return block_table
```

Just put the `seq.block_table`s together and make sure all sequences have the same number of blocks by padding the shorter ones with `-1` (i.e. “unused” entries, as expected by the paged-kernel interface).

The max for both `q` and `k` is needed because the CUDA kernel needs an upper bound (`max_seqlen_q` / `max_seqlen_k`) for specialization/bounds.

`varlen_func`, as the name implies, supports sequences with different lengths. That’s why it needs `cu_seqlens_q`, which has the cumulative lengths of the query sequences (remember that each input token we need to compute attention for contributes Q vectors). Cumulative means that if we have 3 sequences with lens 4, 8, 3, `cu_seqlens_q` will be `[0, 4, 12, 15]` (length `B + 1`). Same thing for `cu_seqlens_k`, which holds cumulative lengths for K/V (i.e. how many cached tokens are visible per sequence). In the simple case, the number of K/V positions matches the sequence/context length, but with prefix cache / truncation it’s best to think of `cu_seqlens_k` as “how much context is actually attended to”.

So we really just set up the metadata for Flash Attention, which handles paged attention in an efficient and effective way. Its internals would make for an interesting future blog post :D.



## Running the model across multiple GPUs


nano-vLLM supports splitting the model across multiple GPUs. That's essential if the model weights don't fit on a single GPU. The model from the example, the Qwen 0.6B, with FP16 (that's 2 bytes per weight), means for each 1B weights we need 2GB. This means for the 70B parameter model, we'd need 140GB of GPU VRAM, and that's without accounting for the KV cache, those are just the weights.

Quantization helps: instead of FP16 (2 bytes), the weights can be mapped to less precise, smaller representations—INT8 (1 byte), or even more extreme INT4, which means 4× less memory than full FP16.

All this to say, being able to leverage multiple GPUs is important. nano-vLLM supports multiple GPUs, but only within a single node. The full-fledged vLLM supports running across multiple nodes, each with multiple GPUs, but that introduces even more complexity and distributed systems challenges.




Anyway, how does nano-vLLM do single-node multi-GPU?

First, we need to split the model weights across the GPUs. That's called tensor parallelism. There are other types of parallelism. We could do data parallelism—that's when the model can fit on a single GPU, and we run multiple copies of it across different data. That's useful for speeding up training; that's what's done in nano-gpt, for instance.

Another type is layer parallelism: instead of splitting each layer, we distribute full layers themselves. There's also expert parallelism, in which each expert in an MoE is placed on its own GPU.

In our case, we're using tensor parallelism (TP). Within each layer, weights are split across multiple GPUs.





In PyTorch, we have the `world_size`, which is the number of GPUs/processes and there's a 1-to-1 mapping. Each process/GPU has a rank. So with 8 GPUs, the world size is 8, and each process has a rank from 0 to 7. Process 0 is usually the master and takes care of bookkeeping and coordination.

### Model Runner Parallilism 

We specify the number of processes/GPUs via the `tensor_parallel_size` config. The model weights are sharded across N GPUs managed by N processes. The first one, `id=0`, is the master, and only it has an instance of the LLM engine and can accept requests via `generate`. The others only have an instance of the `ModelRunner`. Furthermore, when we need to run inference, we trigger it from the master, which triggers a run on the other processes, each on its own portion of the model.

For certain layers, coordination is needed, and the models wait for each other and propagate and reduce their results. For example, after the sharded embedding layer, each GPU has 1/N of the embedding vectors. We run a `dist.all_reduce` to gather them (it's actually a sum, but each GPU has a zero vector for its out-of-bound indices, so a sum is equivalent to a gather). The same goes for `RowParallelLinear` layers, and so on.

However, the final sampling—once we have all the logits and turn them into probabilities for the next token—only takes place on the master.

The master pushes work to the other processes through two primitives in multiprocessing: `Event` and `SharedMemory`. We'll discuss the details shortly. But first, let's look at `LLMEngine`'s constructor:

```py
# wrapper around Python's built-in `multiprocessing`
import torch.multiprocessing as mp

class LLMEngine:

    def __init__(self, model, **kwargs):
        config_fields = {field.name for field in fields(Config)}
        config_kwargs = {k: v for k, v in kwargs.items() if k in config_fields}
        config = Config(model, **config_kwargs)
        self.ps = []
        self.events = []
        ctx = mp.get_context("spawn")
        for i in range(1, config.tensor_parallel_size):
            event = ctx.Event()
            process = ctx.Process(target=ModelRunner, args=(config, i, event))
            process.start()
            self.ps.append(process)
            self.events.append(event)
        self.model_runner = ModelRunner(config, 0, self.events)
```

For each GPU in `tensor_parallel_size`, we create an `Event` and a `Process` that will run `ModelRunner`. Each instance of the runner gets its `Event` instance and is passed `i`, which is its rank. That rank will be used to load its portion of the weights.

The loop starts from `range(1, config.tensor_parallel_size)`. We're short one—that's the master process, which is instantiated right after and is passed all the events.

A process can call `Event.wait()`, and it will block until another process calls `set()` on that event. Then the waiter/consumer can do some work and reset the event using `clear()`, this means it will block on the next call to `wait()`. 

So whenever there is work to be done (running inference on a portion of the model), the master can call `set()` on each of the other processes' events. They will be waiting for it, perform the work, and then go back to sleep waiting for a new piece of work.

We're missing one piece of the puzzle: how can each process know what kind of work it's supposed to do? That's where `SharedMemory` comes into the picture. `SharedMemory` is an OS primitive. As the name implies, it's a portion of memory that different processes can write to and read from. So the master can write the work specs to shared memory, and the others can read from it.

```py
class ModelRunner:

    def __init__(self, config: Config, rank: int, event: Event | list[Event]):
        self.config = config
        hf_config = config.hf_config
        self.block_size = config.kvcache_block_size
        self.enforce_eager = config.enforce_eager
        self.world_size = config.tensor_parallel_size
        self.rank = rank
        self.event = event

        dist.init_process_group("nccl", "tcp://localhost:2333", world_size=self.world_size, rank=rank)
        torch.cuda.set_device(rank)
        default_dtype = torch.get_default_dtype()
        torch.set_default_dtype(hf_config.torch_dtype)
        torch.set_default_device("cuda")
        self.model = Qwen3ForCausalLM(hf_config)
        load_model(self.model, config.model)
        self.sampler = Sampler()
        self.warmup_model()
        self.allocate_kv_cache()
        if not self.enforce_eager:
            self.capture_cudagraph()
        torch.set_default_device("cpu")
        torch.set_default_dtype(default_dtype)

        if self.world_size > 1:
            if rank == 0:
                self.shm = SharedMemory(name="nanovllm", create=True, size=2**20)
                dist.barrier()
            else:
                dist.barrier()
                self.shm = SharedMemory(name="nanovllm")
                self.loop()
```

```py
    def exit(self):
        if self.world_size > 1:
            self.shm.close()
            dist.barrier()
            if self.rank == 0:
                self.shm.unlink()
        if not self.enforce_eager:
            del self.graphs, self.graph_pool
        torch.cuda.synchronize()
        dist.destroy_process_group()
```

```py
    def loop(self):
        while True:
            method_name, args = self.read_shm()
            self.call(method_name, *args)
            if method_name == "exit":
                break
```

```py
    def read_shm(self):
        assert self.world_size > 1 and self.rank > 0
        self.event.wait()
        n = int.from_bytes(self.shm.buf[0:4], "little")
        method_name, *args = pickle.loads(self.shm.buf[4:n+4])
        self.event.clear()
        return method_name, args
```

```py
    def write_shm(self, method_name, *args):
        assert self.world_size > 1 and self.rank == 0
        data = pickle.dumps([method_name, *args])
        n = len(data)
        self.shm.buf[0:4] = n.to_bytes(4, "little")
        self.shm.buf[4:n+4] = data
        for event in self.event:
            event.set()
```

```py
    def call(self, method_name, *args):
        if self.world_size > 1 and self.rank == 0:
            self.write_shm(method_name, *args)
        method = getattr(self, method_name, None)
        return method(*args)
```

I don't know why, but this particular portion of the code feels very crisp and aesthetic to me. Each `ModelRunner` instance creates its own model copy and initializes its own KV cache. At the end of initialization, the master creates the `SharedMemory`, and the others simply open it and enter `self.loop`, which, as the name suggests, is an infinite loop (until a call to `exit` is made).

Inside the loop, we call `read_shm`, which is supposed to run only on non-master processes. It blocks on `self.event.wait()`, and if you search the codebase, the only occurrence of `event.set()` is inside `write_shm`, which is run by the master.

The way things work is that the master is triggered with a call to `call` (sure, why not!). Within `call`, if `world_size > 1` (i.e. there are other processes), we call `self.write_shm`, which writes the method name and args to shared memory and wakes the others with `set()`. That's it.

One interesting thing is that the Sequence class defines `__getstate__` and `__setstate__` because these are needed during Pickle serialization which is used when writing `run`'s args that consist of `Sequence` instances.

So, using `Event` and `SharedMemory`, we can call a method across all N processes and GPUs by writing the method name and args to a common area and having the other processes loop infinitely, waiting for that work.

The distributed methods need to be carefully written so coordination happens correctly, and this is done thanks to Pytorch's `dist`. Also, the methods need to work when we have only one GPU/process, which is why we check `world_size`. But it really is beautiful.



### Embedding Layer

In TP, for the embedding layer—which is basically a lookup table—each token id gets mapped to an embedding vector. So we have:

```py
self.embed_tokens = VocabParallelEmbedding(config.vocab_size, config.hidden_size)
```

We want each GPU to be responsible for one part of the vocab. So with 8 GPUs, each GPU will load `vocab_size / 8` embeddings. Then, during the forward pass, we make sure that each GPU only handles tokens within its interval. So GPU 0 handles `[0, vocab_size/8)`, while GPU 7 handles `[7*vocab_size/8, vocab_size)`.

```py
class VocabParallelEmbedding(nn.Module):
    def __init__(self, num_embeddings: int, embedding_dim: int,):
        super().__init__()
        self.tp_rank = dist.get_rank()
        self.tp_size = dist.get_world_size()
        assert num_embeddings % self.tp_size == 0
        self.num_embeddings = num_embeddings
        self.num_embeddings_per_partition = self.num_embeddings // self.tp_size
        self.vocab_start_idx = self.num_embeddings_per_partition * self.tp_rank
        self.vocab_end_idx = self.vocab_start_idx + self.num_embeddings_per_partition
        self.weight = nn.Parameter(torch.empty(self.num_embeddings_per_partition, embedding_dim))
        self.weight.weight_loader = self.weight_loader

    def weight_loader(self, param: nn.Parameter, loaded_weight: torch.Tensor):
        param_data = param.data
        shard_size = param_data.size(0)
        start_idx = self.tp_rank * shard_size
        loaded_weight = loaded_weight.narrow(0, start_idx, shard_size)
        param_data.copy_(loaded_weight)

    def forward(self, x: torch.Tensor):
        if self.tp_size > 1:
            mask = (x >= self.vocab_start_idx) & (x < self.vocab_end_idx)
            x = mask * (x - self.vocab_start_idx)
        y = F.embedding(x, self.weight)
        if self.tp_size > 1:
            y = mask.unsqueeze(1) * y
            dist.all_reduce(y)
        return y
```

Each GPU is responsible for `num_embeddings_per_partition` embeddings, and we load only that portion of the weights using the weight loader.

During the forward pass, we look up each vector from the embedding table and build a mask that is false (0) for tokens that fall outside the current GPU's range. `y = mask.unsqueeze(1) * y` ensures that `y` will be 0 for all out-of-range inputs.

Finally, we call `dist.all_reduce(y)`, which performs a sum across all GPUs (the default op is SUM). Vectors will be 0 on all GPUs except the one responsible for the interval to which the token belongs. So the end result contains all the embeddings, even though they were split across GPUs.


---

Notice that this module works perfectly when `tp_size = 1`. In that case, we're using a single GPU, and the whole embedding table is on that GPU. `num_embeddings_per_partition == num_embeddings`, and we load all the weights onto one GPU. In the forward pass, we skip the mask and the `all_reduce`.

That's key when it comes to parallelism: the base case of 1 must always work.



### MLP Tensor Parallislm with the Column Row Trick 
This is probably one of the most impressive tricks out there.

An MLP layer is just `Y = XW`, where `W` is the weight matrix. There are two natural ways to split it (since it has two dimensions): either split the **columns** across GPUs or split the **rows** across GPUs.

Let’s think through both.


#### Split columns

GPU0: `X * W0 = Y0`
GPU1: `X * W1 = Y1`
...

Here, every GPU needs the full input `X`. Each output entry is a dot product between a row of `X` and a column of `W`, so if we split columns, each GPU is responsible for a subset of output columns.

So we get `Y0, Y1, ..., YN`, where each one holds 1/N of the output columns.

But this output is the input to the next layer (usually another matmul), so we need an **all_gather** to stitch the full `Y` back together before moving on.

#### Split rows

Now instead, each GPU holds 1/N of the rows of `W`.

This changes things. Each output value depends on the *entire* column of `W`, so if we only have part of the rows, we’re only computing a **partial sum** of the dot product.

Example:

```
# X = [ 1  2  3  4 ]
# W = [ 1  2
#       3  4
#       5  6
#       7  8 ]

X * W = [ 50  60 ]
```

If we split by rows:

```
W_top:
[ 1  2 ]
[ 3  4 ]

W_bot:
[ 5  6 ]
[ 7  8 ]

x_left  = [ 1  2 ]
x_right = [ 3  4 ]
```

Top half:

```
x_left * W_top =
[ (1*1 + 2*3)   (1*2 + 2*4) ]
= [ 7  10 ]
```

Bottom half:

```
x_right * W_bot =
[ (3*5 + 4*7)   (3*6 + 4*8) ]
= [ 43  50 ]
```

Add them:

```
[ 7  10 ] + [ 43  50 ] = [ 50  60 ]
```

So when we split by rows, each GPU computes a **partial Y**: same shape as the final output, but each entry is only ~1/N of the full dot product. At the end, we need an **all_reduce (sum)** to get the correct result.


Comparing the two

* **Column parallelism**:

  * Each GPU produces **1/N of the output columns**, fully computed
  * Need **all_gather** to assemble full `Y`

* **Row parallelism**:

  * Each GPU produces the **full output shape**, but each value is partial
  * Need **all_reduce (sum)** to combine results


Here’s the really neat part:

* Row parallelism only needs the **corresponding slice of the input** (e.g. `x_left`, `x_right`)
* Column parallelism produces **exactly that kind of sliced output** (a subset of columns)

So the output of column parallelism is *already in the right form* to be the input for row parallelism.


If you arrange MLP layers as:

```
[ column-parallel ] -> [ row-parallel ]
```

then:

* The first layer produces sharded outputs (no need to gather)
* The second layer consumes them directly
* You only need to communicate once at the end (via all_reduce)

So you get two fully distributed layers, but only pay for one communication step.

That’s the trick.



Let’s take a look at the actual code.

```py
class LinearBase(nn.Module):
    def __init__(self, input_size: int, output_size: int, tp_dim: int)
        self.tp_dim = tp_dim
        self.tp_rank = dist.get_rank()
        self.tp_size = dist.get_world_size()
        self.weight = nn.Parameter(torch.empty(output_size, input_size))
        self.weight.weight_loader = self.weight_loader
```

`ColumnParallelLinear` is a linear layer with tensor parallelism applied. The critical column split happens inside `__init__`. Notice `super().__init__(input_size, divide(output_size, tp_size), bias, 0)`. The number of rows (i.e. input dimension) is the same across GPUs, but the number of columns (i.e. output dimension) is reduced to `divide(output_size, tp_size)`, where `tp_size` is the world size (number of GPUs).

Also note the last argument passed to the parent constructor: `tp_dim`, the tensor-parallel dimension. In the column-parallel case, it is `0`. Since the weight matrix has shape `(output_size, input_size)`, splitting along dimension `0` means we are partitioning the **rows** across GPUs. Taking 1/N columns from each row.

This shows up in `weight_loader`. When we load the weights, we “narrow” along `tp_dim`:
`loaded_weight.narrow(self.tp_dim, start_idx, shard_size)`.
This means that along dimension `0`, each GPU loads a contiguous shard of rows. So GPU 0 starts at index 0, GPU N starts at `shard_size * N`.

One last thing: in `forward`, we simply run the matrix multiplication and there’s no call to `dist` (no gather or reduce). This is because each GPU produces only a **partial output (a subset of output features)**. It is typically assumed that this layer will be followed by a `RowParallelLinear` (or another operation) that combines results appropriately.

Speaking of which, let’s move on to row parallelism.

```py
class RowParallelLinear(LinearBase):

    def __init__(self, input_size: int, output_size: int, bias: bool = False):
        tp_size = dist.get_world_size()
        super().__init__(divide(input_size, tp_size), output_size, bias, 1)

    def weight_loader(self, param: nn.Parameter, loaded_weight: torch.Tensor):
        param_data = param.data
        shard_size = param_data.size(self.tp_dim)
        start_idx = self.tp_rank * shard_size
        loaded_weight = loaded_weight.narrow(self.tp_dim, start_idx, shard_size)
        param_data.copy_(loaded_weight)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        y = F.linear(x, self.weight, self.bias if self.tp_rank == 0 else None)
        if self.tp_size > 1:
            dist.all_reduce(y)
        return y
```

This is very similar to `ColumnParallelLinear`, but there are three key differences:

1. We divide `input_size` by `tp_size`, unlike the column-parallel case where we divided `output_size`.
2. Notice the last argument passed to the parent constructor: `tp_dim = 1`. Since the weight matrix has shape `(output_size, input_size)`, splitting along dimension `1` means we are partitioning the **icolumns** across GPUs. Each GPU holds a 1/N rows each column.
3. After running `F.linear` in `forward`, we call `dist.all_reduce(y)`, which sums the partial outputs across all GPUs. Each GPU computes a **partial contribution** to the full output (because it only sees part of the input). After the sum, every GPU ends up with the full result.

We must perform this reduction because the next stage expects the complete output tensor, not a partial one.

### Splitting Attention Layer Across Multiple GPUS

The beating heart of our model is the attention layer:

```
Given input X:

Q = X @ W_Q
K = X @ W_K
V = X @ W_V

Attention scores (causal / with masking):
S = Q @ K^T

Scaled scores:
S_scaled = S / sqrt(d_k)

Apply softmax:
A = softmax(S_scaled)

Output:
O = A @ V
```

So technically, there are 3 sets of weights we care about: W_Q, W_K, and W_V.

The mathematical operations shown above are not exactly what happens. Each weight matrix is actually split into NH parts. Each part is called a head. All the operations for each head run separately, and it's only at the end, at `O = A @ V`, that we concatenate the output of each head to get the final result.

For the first 3 matmuls (projecting X onto W_Q, W_K, and W_V), it doesn't really make any difference whether we split or not—the result is the same. For the actual attention, softmax, and final output, these ops happen within each head. A query from head i pays attention only to keys and values from head i.

The first 3 matmuls and their weights can be seen as a regular linear layer. Furthermore, because the result of these first matmuls will be split into heads and each head's calculation is independent of the rest, this fits perfectly with column parallelism: the output of each ColumnParallel module is 1/N output columns. These columns are independent heads.

Now things get a little bit tricky. The weights we're loading from [HF](https://huggingface.co/Qwen/Qwen3-0.6B?show_file_info=model.safetensors) have separate layers for the attention weights:

```
model.layers.0.self_attn.k_proj.weight    [1 024, 1 024]    
model.layers.0.self_attn.o_proj.weight    [1 024, 2 048]    
model.layers.0.self_attn.q_proj.weight    [2 048, 1 024]    
model.layers.0.self_attn.v_proj.weight    [1 024, 1 024]
```

Notice that Qwen actually has 2 Q heads for each K and V head. That’s why `q_proj.weight` is `[2 048, 1 024]`.

But in vLLM, for efficiency purposes, QKV are bundled into the same layer:

```py
class QKVParallelLinear(ColumnParallelLinear):

    def __init__(
        self,
        hidden_size: int,
        head_size: int,
        total_num_heads: int,
        total_num_kv_heads: int | None = None,
        bias: bool = False,
    ):
        tp_size = dist.get_world_size()
        total_num_kv_heads = total_num_kv_heads or total_num_heads
        self.head_size = head_size
        self.num_heads = divide(total_num_heads, tp_size)
        self.num_kv_heads = divide(total_num_kv_heads, tp_size)
        output_size = (total_num_heads + 2 * total_num_kv_heads) * self.head_size
        super().__init__(hidden_size, output_size, bias)

    def weight_loader(self, param: nn.Parameter, loaded_weight: torch.Tensor, loaded_shard_id: str):
        param_data = param.data
        assert loaded_shard_id in ["q", "k", "v"]
        if loaded_shard_id == "q":
            shard_size = self.num_heads * self.head_size
            shard_offset = 0
        elif loaded_shard_id == "k":
            shard_size = self.num_kv_heads * self.head_size
            shard_offset = self.num_heads * self.head_size
        else:
            shard_size = self.num_kv_heads * self.head_size
            shard_offset = self.num_heads * self.head_size + self.num_kv_heads * self.head_size
        param_data = param_data.narrow(self.tp_dim, shard_offset, shard_size)
        loaded_weight = loaded_weight.chunk(self.tp_size, self.tp_dim)[self.tp_rank]
        param_data.copy_(loaded_weight)
```

In each QKV shard (1 shard per GPU), we have `num_heads` Q heads and `num_kv_heads` K and V heads. From the weight loader, we can see that the packing is as follows:

```
at offset "0": Q weights
at offset  "num_heads * head_size" : K  weights
at offset "num_heads * head_size + num_kv_heads * head_size": V  weights
```

So each QKV shard on each GPU holds 1/N of the Q, K, and V layers. The offsets are calculated in the weight loader, and the weights are packed accordingly.


We're just getting started with attention. That's the projection. We now need the softmax and output calculation and concatenation. This happens in:

```py
class Qwen3Attention(nn.Module):
    def __init__(self, hidden_size:int, num_heads:int, num_kv_heads:int, max_position:int=4096*32,...) -> None:
        # ...
        self.qkv_proj = QKVParallelLinear(hidden_size, self.head_dim, num_heads, num_kv_heads, bias=qkv_bias)
        self.o_proj = RowParallelLinear(num_heads*self.head_dim, hidden_size, bias=False)
        self.rotary_emb = get_rope(self.head_dim, rotary_dim=self.head_dim, max_position=max_position,
                                  base=rope_theta, rope_scaling=rope_scaling)
        self.attn = Attention(self.num_heads, self.head_dim, self.scaling, self.num_kv_heads)

    def forward(self, positions:torch.Tensor, hidden_states:torch.Tensor) -> torch.Tensor:
        q,k,v = self.qkv_proj(hidden_states).split([self.q_size,self.kv_size,self.kv_size], dim=-1)
        q,k,v = q.view(-1,self.num_heads,self.head_dim), k.view(-1,self.num_kv_heads,self.head_dim), v.view(-1,self.num_kv_heads,self.head_dim)
        if not self.qkv_bias: q,k = self.q_norm(q), self.k_norm(k)
        q,k = self.rotary_emb(positions, q, k)
        return self.o_proj(self.attn(q,k,v).flatten(1,-1))

```

This is trimmed and condensed, but the gist is there. We use `QKVParallelLinear` (column parallelism) to get the projections of q, k, and v in the same layer. 

We then split and reshape them to make sure the subsequent computations are isolated to each head. We run RoPE to take positions into account, and finally we’re ready to run the actual attention: softmax, scaling, and value weighting. `Attention`, or `self.attn`, is where the KV cache and flash attention are called, and where the real optimization and magic of paged attention take place. 


The final op is `self.o_proj`, which is a simple linear layer, but we’re using `RowParallelLinear` because that’s the distributed linear layer that performs a “reduce” at the end and provides a complete result (unlike ColumnParallel).

Let’s keep in mind that each GPU has its own process running an instance of `ModelRunner`, and KV cache management happens within `ModelRunner`. This makes sense because that cache is used to manage VRAM, and each GPU has its own. Furthermore, the KV cache is populated within the call to `self.attn(q, k, v)`, and as we said, each GPU handles its own portion with its own heads.

The whole Qwen model is simply a succession of these various layers (we omitted a few in our exploration):

```py
class Qwen3DecoderLayer(nn.Module):
    def __init__(self, config: Qwen3Config) -> None:
        super().__init__()
        self.self_attn = Qwen3Attention(
            config.hidden_size, config.num_attention_heads, config.num_key_value_heads,
            config.max_position_embeddings, config.rms_norm_eps,
            getattr(config, 'attention_bias', True),
            getattr(config, 'head_dim', None),
            getattr(config, "rope_theta", 1_000_000),
            getattr(config, "rope_scaling", None),
        )
        self.mlp = Qwen3MLP(config.hidden_size, config.intermediate_size, config.hidden_act)
        self.input_layernorm = RMSNorm(config.hidden_size, eps=config.rms_norm_eps)
        self.post_attention_layernorm = RMSNorm(config.hidden_size, eps=config.rms_norm_eps)

    def forward(self, positions, hidden_states, residual=None):
        hidden_states, residual = (
            (self.input_layernorm(hidden_states), hidden_states)
            if residual is None else self.input_layernorm(hidden_states, residual)
        )
        hidden_states = self.self_attn(positions, hidden_states)
        hidden_states, residual = self.post_attention_layernorm(hidden_states, residual)
        return self.mlp(hidden_states), residual


class Qwen3Model(nn.Module):

    def __init__(
        self,
        config: Qwen3Config,
    ) -> None:
        super().__init__()
        self.embed_tokens = VocabParallelEmbedding(config.vocab_size, config.hidden_size)
        self.layers = nn.ModuleList([Qwen3DecoderLayer(config) for _ in range(config.num_hidden_layers)])
        self.norm = RMSNorm(config.hidden_size, eps=config.rms_norm_eps)

    def forward(
        self,
        input_ids: torch.Tensor,
        positions: torch.Tensor,
    ) -> torch.Tensor:
        hidden_states = self.embed_tokens(input_ids)
        residual = None
        for layer in self.layers:
            hidden_states, residual = layer(positions, hidden_states, residual)
        hidden_states, _ = self.norm(hidden_states, residual)
        return hidden_states
```

So stitching everything together, we see attention, MLP, and RMSNorm with residuals put together in a decoder block, then stacked multiple times to form the core of our model.

