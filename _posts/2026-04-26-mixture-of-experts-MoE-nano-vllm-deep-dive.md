---

title: "Exploring Mixture of Experts: From Concept to Inference Engine"
author: cef
date: 2026-04-26
categories: [Technical Writing, Open Source]
tags: [AI, Open Source, Opencode, LLM, vLLM]
render_with_liquid: false
description: "In this post, we dabble in Mixture of Experts (MoE) models through a concrete nano-vLLM implementation, exploring Triton kernels, expert parallelism, and other fun things."

---

<iframe
  class="embed-video"
  loading="lazy"
  src="https://www.youtube.com/embed/Ek-3buKUuvk"
  title="YouTube video player"
  frameborder="0"
  allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
  allowfullscreen
></iframe>


In my last post, I did a [deep dive into LLM inference engines using nano-vllm](https://cefboud.com/posts/inside-llm-inference-engine-nano-vllm-explanation/). It was a blast.

But the thing is, nano-vllm only supported the dense Qwen version (Qwen3 0.6B) as its main example. It didn't have support for the MoE version. Mixture of Experts.

As a fun learning exercise, I (using Codex and Opencode's 'Big Pickle') set out to add MoE support to nano-vllm, using vllm as the reference. For a newbie, it was even harder than imagined. In this post, I'll do a brief overview of MoE and discuss the added implementation.


Full implentation can be found [here](https://github.com/CefBoud/nano-vllm/commit/e9c144e159e43962f4c9c02c5bed3f61d70eb649).

## Mixture of Experts

Dense MLP layers take a lot of space in memory. The key idea behind MoE is that you can train a huge model but only rely on a subset of the weights at inference time.

Think of it like a consulting firm staffing a project. The firm has 128 consultants on the bench, each with a different specialty. For any given project (token), a managing partner (the router) picks the 8 most relevant people, and those 8 each contribute their piece to the deliverable. The firm has enormous total capacity, but any single project only uses a small, targeted team. Different projects get different teams.

MoE works the same way. We split a dense MLP layer into N chunks (experts), then add a small layer that routes each token to the right chunks. The router is learned, ofc.

The basic MLP layer looks like this:

```
FFN(x) = down_proj(SiLU(gate_proj(x)) * up_proj(x))
```

Each token's vector `x` interacts with all the weights. To improve the model, we scale the weights up, but computation scales linearly with that.

Now here's the MoE twist:

We replace the single FFN with N parallel FFNs (experts), and only activate K of them per token.

```
MoE-FFN(x) = sum of  g_i(x) * Expert_i(x)     for i in top_K_experts
```

`g_i(x)` is the gating weight of expert i. Out of N experts, we pick the top K (say K=2), each with its own weight.

How do we get the `g_i` values? We have a learned gating weight matrix `W_g`:

```
g = softmax(W_g * x)
```

Then we use topk to grab the K highest weights (those map to the experts we activate. After picking the top-K experts, we often renormalize by dividing each weight by the sum of the chosen weights. This ensures they sum to 1, which keeps the output stable) without it, the magnitude could shrink unpredictably depending on how the gate distributes probability mass.



```
softmax(x * W_g) = [0.2, 0.3, 0.1, 0.4]
                        ^         ^
topk(2)           => experts 1 and 3 (0-indexed), with weights 0.3 and 0.4
```

Each of those experts has its own weight matrices: We0, We1, We2, We3. We take the top experts, matmul their weights with x, then combine the results using the gating weights. The cool thing: we can scale the model by adding more experts and their weights, but only use the relevant knowledge based on the router (the softmaxed gate). The model's quality improves, but the compute doesn't scale linearly with total parameter count.

```python
# Pseudocode
logits = W_gate @ x                    # shape: [N_experts]
top_k_indices = topk(logits, K)        # pick top K experts
top_k_weights = softmax(logits[top_k_indices])  # renormalize

output = sum(
    weight_i * expert_i(x)
    for i, weight_i in zip(top_k_indices, top_k_weights)
)
```


For a model like [Qwen3-30B-A3B-Base](https://huggingface.co/Qwen/Qwen3-30B-A3B-Base), this means the model has 30B parameters total, but only 3B are activated per token because of MoE.

Calculating that is pretty straightforward. We go [here](https://huggingface.co/Qwen/Qwen3-30B-A3B-Base?show_file_info=model.safetensors.index.json), grab all the layer dimensions, and multiply them out. Each layer of our Qwen model has 128 experts, but each token only activates 8 of them per layer. The activated expert indices change every layer, because we run the router (gating softmax with topk) at each one.

```python
def total_params(num_expert):
    return (
        # --- Embedding ---
        (151_936 * 2_048)  # token embeddings

        # --- Transformer layers (48 total) ---
        + 48 * (

            # --- Attention ---
            (4_096 * 2_048)   # q_proj
            + (512 * 2_048)   # k_proj
            + (512 * 2_048)   # v_proj
            + (2_048 * 4_096) # o_proj

            # --- Attention norms ---
            + 128             # q_norm
            + 128             # k_norm

            # --- LayerNorms ---
            + 2_048           # input_layernorm
            + 2_048           # post_attention_layernorm

            # --- MoE gating ---
            + (128 * 2_048)   # router / gate

            # --- MoE experts (128 experts per layer, only 8 active) ---
            + num_expert * (
                (2_048 * 768)   # down_proj
                + (768 * 2_048) # gate_proj
                + (768 * 2_048) # up_proj
            )
        )

        # --- Final norm ---
        + 2_048

        # --- LM head ---
        + (151_936 * 2_048)
    )

print("active", total_params(8) / 10**9, "Billions")
print("total", total_params(128) / 10**9, "Billions")

# active 3.353032704  Billions
# total 30.532122624 Billions
```

And there we have it: 30B-A3B!


## Inference

Now that we understand what MoE is and how to count its parameters, let's talk about what happens when we actually run this thing.

MoE is tricky because there's unpredictability in the calculation. At each layer, we don't know which experts (weights) will be used by the tokens.

This becomes a problem when we're running multiple GPUs. Say our model doesn't fit in a single device. We need to split it. For the MoE layers, one way of doing this is through EP (Expert Parallelism). From our Qwen30B example with 128 experts and 4 GPUs, each GPU will shelter 32 experts from each layer. GPU 0 gets experts 0-31, GPU 1 gets 32-63, and so on.

Now let's ask the following question: we have a batch of 100 tokens. At each layer, each one of these will need 8 experts. These experts could live on any of the 4 GPUs. At the next layer, the chosen 8 experts could be entirely different. That's tricky.

How do we solve it with our simple EP (there are other more complex variants)? The gate layer (softmax and topk) is replicated across all GPUs, so at each layer within each GPU, we know where every token should go. Each GPU then runs the MoE layer for all token/expert pairs (i.e. token i routed to expert j). If expert j doesn't reside on this GPU, we just set the result to 0.

So each GPU only calculates the output of its own experts and zeros out the rest. Let's remember the MoE formula:

```
MoE-FFN(x) = sum of  g_i(x) * Expert_i(x)     for i in top_K_experts
```

Within each GPU we have the result `g_i(x) * Expert_i(x)` for the local experts only. After that step, we perform an [`all_reduce`](https://docs.pytorch.org/docs/stable/distributed.html#torch.distributed.all_reduce) to sum these partial results across all GPUs. Each GPU now has the full MoE output.

This pattern (split the work, compute partial results, all\_reduce to combine) shows up a lot in distributed inference. It's exactly how RowParallel linear layers work, and the analogy is worth understanding because it maps directly to MoE.

### RowParallel: A Concrete Example

Say we have a weight matrix `W` of shape `(4, 4)` and an input vector `x` of shape `(1, 4)`. We want `y = x @ W`. On 2 GPUs, we split `W` by columns (each GPU gets 2 columns worth of rows in the transposed view):

```
Full computation:  y = x @ W

x = [x0, x1, x2, x3]

W = [[w00, w01, w02, w03],    GPU 0 gets rows 0-1:  W0 = [[w00, w01, w02, w03],
     [w10, w11, w12, w13],                                  [w10, w11, w12, w13]]
     [w20, w21, w22, w23],    GPU 1 gets rows 2-3:  W1 = [[w20, w21, w22, w23],
     [w30, w31, w32, w33]]                                  [w30, w31, w32, w33]]
```

Each output element `y[j]` is `x[0]*W[0][j] + x[1]*W[1][j] + x[2]*W[2][j] + x[3]*W[3][j]`. When we split W by rows, each GPU computes a partial dot product, GPU 0 computes `x[0]*W[0][j] + x[1]*W[1][j]` and GPU 1 computes `x[2]*W[2][j] + x[3]*W[3][j]`. Both produce a full-shaped output, but each only has a partial sum. The all\_reduce adds them together to give the correct result on every GPU.

Now apply this reasoning to MoE. Each GPU computes the weighted expert outputs for its local experts only, producing a full-shaped result where non-local experts contribute zero. The all\_reduce sums these across GPUs, same pattern, same result. If a token's top-K experts all happen to live on one GPU, the all\_reduce just sums that GPU's full result with zeros from the others and propagates it everywhere. This is crucial: after each MoE layer, every GPU needs the full result to keep going.

### The Locality Problem

Another issue rears its ugly head. GPUs love locality and contiguous computation. Imagine GPU 1 has run the router on 100 tokens. Let's say 30 of those tokens are mapped to local experts, so we need to run the calculation on them. GPU 1 has experts 32 to 63 (in our 4-GPU scenario), but those 30 tokens are probably not contiguous in the input matrix X. Even worse, neighboring tokens aren't necessarily mapped to the same local expert. It would be very inefficient to load the expert weights for each token in an ad hoc fashion. Ideally, we group all tokens routed to the same expert close to each other, so when we run the matmul with that expert's weights, we benefit from locality and GPU cache.

We need to be very mindful about this. Preparing the input for the expert calculation on the CPU (in pure Python, not PyTorch, Triton, or CUDA) would be a huge bottleneck. So we try to minimize what takes place on the CPU as much as possible. For our purposes, we perform this preparation in PyTorch, but we still end up needing the CPU at one step for an allocation. That's far from ideal, the original vLLM does this in pure CUDA. This CPU step also breaks CUDA graphs, which require everything to run on the GPU with predictable sizes.


## The Model Code

With the theory and the distributed strategy covered, let's look at the [actual code](https://github.com/CefBoud/nano-vllm/commit/e9c144e159e43962f4c9c02c5bed3f61d70eb649).  I'm showing trimmed snippets here focused on the key ideas.

`Qwen3MoeModel` is a list of `Qwen3MoeDecoderLayer`, which is Qwen3Attention followed by our `Qwen3MoeSparseMoeBlock` (that's the main addition). After calling `self_attn` then `post_attention_layernorm`, instead of calling a dense MLP, we call our MoE layer.

```py
# Qwen3MoeSparseMoeBlock (trimmed)
class Qwen3MoeSparseMoeBlock(nn.Module):
    def __init__(self, ...):
        self.gate = ReplicatedLinear(hidden_size, num_experts, bias=False)
        self.experts = FusedMoE(num_experts=num_experts, top_k=num_experts_per_tok, ...)

    def forward(self, hidden_states):
        router_logits = self.gate(hidden_states)   # replicated across all GPUs
        return self.experts(hidden_states, router_logits)
```

Two components. The `gate` is a `ReplicatedLinear`, meaning its weights are duplicated across all GPUs, because every GPU needs to know which experts each token is assigned to. Then there's `self.experts`, an instance of `FusedMoE`. That's where all the MoE logic resides. It's called FusedMoE because it fuses all the MoE steps into a single module: softmax, aligning tokens to the appropriate expert, each expert's MLP calculation, and summing across the weighted expert results.

Now let's look inside `FusedMoE`. First, the expert parallelism setup:

```py
# FusedMoE.__init__ (trimmed)
self.local_num_experts = num_experts // self.tp_size  # divide experts across GPUs

expert_map = torch.full((num_experts,), -1, dtype=torch.int32)
start = self.tp_rank * self.local_num_experts
end = start + self.local_num_experts
expert_map[start:end] = torch.arange(self.local_num_experts, dtype=torch.int32)

# each GPU only holds weights for its local experts
self.w13 = nn.Parameter(torch.empty(self.local_num_experts, 2 * intermediate_size, hidden_size))
self.w2 = nn.Parameter(torch.empty(self.local_num_experts, hidden_size, intermediate_size))
```

We divide the number of experts by `world_size` (number of GPUs) to get how many experts each GPU holds. Using the rank (the ID of each specific GPU), we determine which experts it gets. The `expert_map` is crucial: it maps a global expert ID to a local one. On a 4-GPU setup, global expert 32 maps to local expert 0 on GPU 1, and global expert 127 maps to local expert 31 on GPU 3. This map is used on each GPU to decide which weights to load and which token-expert pairs to compute.

And the forward pass:

```py
# FusedMoE.forward (trimmed)
def forward(self, hidden_states, router_logits):
    out = fused_moe(hidden_states, router_logits, self.w13, self.w2, ...)
    if self.tp_size > 1:
        dist.all_reduce(out)   # sum partial results across GPUs
    return out
```

Run MoE on local experts, then `all_reduce` to sum the partial results. Exactly the pattern we covered in the RowParallel section above.

### The w13 Weight Fusion

Compared to the dense Qwen models such as Qwen 0.6B, MoE is simply adding this layer and loading its weights. If we inspect the [safetensors JSON file](https://huggingface.co/Qwen/Qwen3-30B-A3B-Base?show_file_info=model.safetensors.index.json) on HuggingFace, for each expert we'll see:

```
model.layers.0.mlp.experts.0.down_proj.weight    [2048, 768]
model.layers.0.mlp.experts.0.gate_proj.weight    [768, 2048]
model.layers.0.mlp.experts.0.up_proj.weight      [768, 2048]
```

Each expert's equation again:

```
FFN(x) = down_proj(SiLU(gate_proj(x)) * up_proj(x))
```

So we need `dot(X, W_gate_proj)`, apply SiLU, element-wise multiply with `dot(X, W_up_proj)`, and finally dot with `W_down_proj`. That's 3 dot products.

Instead of running them separately, we can combine the first two (gate\_proj and up\_proj), that's what `w13` is for. Its size is `(2 * intermediate_size, hidden_size)`. When we load the weights, `gate_proj` goes in the first half and `up_proj` in the second half. We run a single dot product, split the result in two, apply SiLU to the gate half, and element-wise multiply with the up half.


## The fused_moe Function

The gate that produces the router weights runs before `FusedMoE`. The output of `ReplicatedLinear` (the `gate`) is passed in as `router_logits`. Here's the function, trimmed to its essential flow:

```py
def fused_moe(hidden_states, router_logits, w13, w2, top_k, renormalize=True, expert_map=None):
    # 1. Route: softmax → topk → renormalize
    routing_weights = torch.softmax(router_logits.float(), dim=-1)
    topk_weights, topk_ids = torch.topk(routing_weights, top_k, dim=-1)
    if renormalize:
        topk_weights = topk_weights / topk_weights.sum(dim=-1, keepdim=True)

    # 2. Align tokens by expert for GPU-friendly processing
    sorted_token_ids, expert_ids, num_tokens_post_padded = moe_align_block_size(
        topk_ids, block_size=64, num_experts=num_experts, expert_map=expert_map
    )

    # 3. First matmul: hidden_states @ w13 → [gate_proj | up_proj] combined
    invoke_fused_moe_kernel(A=hidden_states, B=w13, C=intermediate, ..., mul_routed_weight=False)

    # 4. Activation: SiLU on gate half, element-wise multiply with up half
    gate, up = intermediate.chunk(2, dim=-1)
    intermediate = F.silu(gate) * up

    # 5. Second matmul: intermediate @ w2 → output, weighted by router
    invoke_fused_moe_kernel(A=intermediate, B=w2, C=output, ..., mul_routed_weight=True)

    # 6. Sum across the top_k expert outputs per token
    return output.view(num_tokens, top_k, hidden_size).sum(dim=1)
```

Steps 1 through 6 map directly to everything we've discussed. The two most interesting and challenging pieces are `moe_align_block_size` (step 2) and `invoke_fused_moe_kernel` (steps 3 and 5).


## Token Alignment: moe_align_block_size

`moe_align_block_size` takes `topk_ids` of shape `(num_tokens, top_k)`, where each row contains the chosen experts for that token. It reorganizes things so tokens routed to the same expert are grouped together. It also packs them into fixed-size blocks, padding when the number of token/expert elements doesn't fill a block evenly.

Each block (size 64) contains the positions of token/expert elements in the flattened `topk_ids`. Every block cares about exactly one expert, so our GPU kernel can process it efficiently. Padded slots (ones without an actual token/expert pair) use the sentinel value `num_tokens * top_k` (= `num_valid_tokens`), so we can skip them during computation.

We also pass the `expert_map` because the expert values in `topk_ids` are global and we need to map them to local ones. That happens at the end: `expert_ids = expert_map[expert_ids.long()]`.

A concrete example makes this click:

```
Example with 4 tokens, top_k=2, 4 experts, block_size=4:

topk_ids = [[2,3], [0,2], [1,0], [3,1]]

Flattened: [2, 3, 0, 2, 1, 0, 3, 1]
Pair indices:  0  1  2  3  4  5  6  7

Group by expert:
  Expert 0: pair indices [2, 5]   (from tokens 1 and 2)
  Expert 1: pair indices [4, 7]   (from tokens 2 and 3)
  Expert 2: pair indices [0, 3]   (from tokens 0 and 1)
  Expert 3: pair indices [1, 6]   (from tokens 0 and 3)

After padding to block_size=4:
  Expert 0: [2, 5, PAD, PAD]    Expert 1: [4, 7, PAD, PAD]
  Expert 2: [0, 3, PAD, PAD]    Expert 3: [1, 6, PAD, PAD]

sorted_token_ids = [2,5,8,8, 4,7,8,8, 0,3,8,8, 1,6,8,8]
                                                (8 = num_valid_tokens = padding sentinel)
expert_ids = [0, 1, 2, 3]  (one per block of 4)
```

The implementation is: count tokens per expert, compute block-aligned offsets, argsort by expert, write token indices into their block slots. Here are the key steps:

```py
def moe_align_block_size(topk_ids, block_size, num_experts, expert_map=None):
    flat_ids = topk_ids.flatten()

    # count how many tokens each expert got, pad to block_size
    tokens_per_expert = torch.bincount(flat_ids, minlength=num_experts)
    tokens_per_expert_padded = (tokens_per_expert + block_size - 1) // block_size * block_size

    # pre-fill with sentinel value (num_valid_tokens) so padded slots are skippable
    sorted_token_ids = torch.full((num_tokens_post_padded,), num_valid_tokens, ...)

    # one expert_id per block. this tells the kernel which expert weights to load
    blocks_per_expert = tokens_per_expert_padded // block_size
    expert_ids = torch.repeat_interleave(torch.arange(num_experts, ...), blocks_per_expert)

    # sort token indices by expert, write them into block-aligned positions
    order = flat_ids.argsort(stable=True)
    # ... compute write positions from expert offsets + within-expert index ...
    sorted_token_ids[write_positions] = order

    # map global expert IDs to local ones for this GPU
    if expert_map is not None:
        expert_ids = expert_map[expert_ids.long()]

    return sorted_token_ids, expert_ids, num_tokens_post_padded
```


## The Triton Kernel

Now let's get to the fun part. `invoke_fused_moe_kernel` launches a Triton kernel that runs the actual expert matmul for all token-expert pairs in a single GPU kernel launch. 

### It's Just a GEMM With Indirection

The kernel is basically a standard tiled matrix multiply: `C = A @ B`. The twist? It doesn't iterate over rows of A sequentially. Instead, it reads from `sorted_token_ids` to figure out *which* row of A to pluck, and from `expert_ids` to figure out *which* expert's weight matrix B to use.

```py
# which tokens does this block process? look up from sorted order
offs_token = tl.load(sorted_token_ids_ptr + pid_m * BLOCK_SIZE_M + offs_m)

# which expert's weights? same for the entire block (by construction from the align step)
off_expert = tl.load(expert_ids_ptr + pid_m)

# A row: pair index // top_k recovers the original token index
a_ptrs = a_ptr + (offs_token[:, None] // top_k) * stride_am + offs_k[None, :] * stride_ak

# B slice: this expert's weight matrix
b_ptrs = b_ptr + off_expert * stride_be + offs_n[None, :] * stride_bn + offs_k[:, None] * stride_bk
```

One extremely interesting thing is happening here. `offs_n[None, :] * stride_bn + offs_k[:, None] * stride_bk` is what I'd call a beautiful broadcast. We have N columns in `offs_n` and K rows in `offs_k`:

```
offs_n[None, :] = [[0, 1, 2]]        # shape (1, N)
offs_k[:, None] = [[0],
                    [1],
                    [2],
                    [3]]              # shape (K, 1)

ptrs = base + offs_k[:, None] * stride_bk + offs_n[None, :] * stride_bn

     = [[1200+0+0,   1200+0+10,  1200+0+20],
        [1200+1+0,   1200+1+10,  1200+1+20],
        [1200+2+0,   1200+2+10,  1200+2+20],
        [1200+3+0,   1200+3+10,  1200+3+20]]
```

Basically, for each row offset in `offs_k`, we add every column offset in `offs_n`. And we end up with a full K x N tile of pointers into memory. One broadcast, one add  and we have all the addresses we need.

The kernel itself follows the well-established tiled GEMM pattern:

```
for each tile (m_block, n_block):
    accumulator = 0
    for k_block in range(K // BLOCK_SIZE_K):
        accumulator += A_tile @ B_tile
    C_tile = accumulator
```

Why tiles? GPUs are fast at math but slow at memory access. By loading a tile of A and a tile of B into fast on-chip memory (registers/SRAM), we can reuse those values across many multiply-accumulate operations before going back to slow global memory for the next tile. The ratio of compute to memory access is what makes or breaks GPU performance and tiling maximizes it.

Now, this kernel gets called twice in `fused_moe`. The first call computes `hidden_states @ w13`, that gives us the combined gate\_proj and up\_proj output for each token-expert pair. We pass `top_k=top_k` so the kernel knows each token appears `top_k` times in the sorted list (one per expert it was routed to). `mul_routed_weight=False` here, we don't apply the router weights yet.

Between the two calls, back in Python, we split the result, apply SiLU, and element-wise multiply (the activation step we discussed earlier).

The second call computes `intermediate @ w2`. This time we pass `top_k=1` because each row of the intermediate already corresponds to a single token-expert pair. And `mul_routed_weight=True`, after the GEMM, the kernel multiplies each row by the router's weight for that pair. We only multiply once at the end because `w * (A @ B) = (A @ B) * w`, so it's mathematically equivalent but saves a broadcast multiply on the first call.

One subtle thing: the accumulator stays in float32 until after the router weight multiply, then gets cast to the output dtype. This matters for bf16 where the 8-bit mantissa can introduce noticeable rounding errors if you multiply in low precision.

### Grouped Ordering for L2 Cache Reuse

The kernel uses a `GROUP_SIZE_M` trick. The grid is 1D: one program per `(m_block, n_block)` tile. Instead of mapping `pid` to `(pid_m, pid_n)` in row-major order (left to right, top to bottom), we group `GROUP_SIZE_M` adjacent M-blocks together before advancing to the next N-block.

Why does this matter? Remember, `moe_align_block_size` sorted tokens by expert. So adjacent M-blocks often belong to the same expert. Same expert = same B weights. By processing these M-blocks on nearby thread blocks, those B tiles stay hot in L2 cache instead of getting evicted and reloaded. Pretty neat.

```py
# grouped pid -> (pid_m, pid_n) mapping
num_pid_in_group = GROUP_SIZE_M * num_pid_n
group_id = pid // num_pid_in_group
first_pid_m = group_id * GROUP_SIZE_M
group_size_m = min(num_pid_m - first_pid_m, GROUP_SIZE_M)
pid_m = first_pid_m + ((pid % num_pid_in_group) % group_size_m)
pid_n = (pid % num_pid_in_group) // group_size_m
```

### Zero-Write for Non-Local Experts

Remember, with EP each GPU only has weights for its local experts. The `expert_map` remapped non-local experts to `-1`. In the kernel, when we hit a `-1`, we don't skip the block, we write zeros:

```py
if off_expert == -1:
    tl.store(c_ptrs, tl.zeros((BLOCK_SIZE_M, BLOCK_SIZE_N), dtype=compute_type), mask=c_mask)
    return
```

We can't just skip the write because the output buffer might have stale garbage in it. We need clean zeros so the `all_reduce` sum across GPUs gives the correct result.

### The Padding Sentinel

The padding slots from `moe_align_block_size` have `offs_token = num_valid_tokens`. The kernel uses this as a natural mask:

```py
token_mask = offs_token < num_valid_tokens
```

Any load or store with `mask=token_mask` skips padded slots. No extra bookkeeping, the sentinel value works as both "skip this slot" and "won't accidentally alias a real token."

### Router Weights: Apply Once, Not Twice

The kernel is called twice: once for `hidden_states @ w13` (without router weights) and once for `intermediate @ w2` (with router weights). The `MUL_ROUTED_WEIGHT` flag controls this. When true, after the GEMM loop:

```py
if MUL_ROUTED_WEIGHT:
    moe_weight = tl.load(topk_weights_ptr + offs_token, mask=token_mask, other=0.0)
    accumulator *= moe_weight[:, None]
```

This multiplies each row by the router's weight for that token-expert pair. We only do this on the second matmul, applying it once at the end is mathematically equivalent to applying it on both (since `w * (A @ B) = (A @ B) * w`), but saves a broadcast multiply.

One observation thing: the accumulator stays in float32 until after this multiply, then gets cast to the output dtype. This matters for bf16 where the 8-bit mantissa can introduce noticeable rounding errors if you multiply in low precision.
