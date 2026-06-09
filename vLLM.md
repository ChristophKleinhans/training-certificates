From deeplearning.ai [LINK](https://learn.deeplearning.ai/courses/fast-and-efficient-llm-inference-with-vllm/) the *Fast & Efficient LLM Inference with vLLM* training

# Fast & Efficient LLM Inference with vLLM

## Memory challenge

### General

One a LLM generate text, it creates one token at a time and also taking into account all previous tokens. Each token's generation uses GPU memory for:

1. Model weights
2. KV cache (representing the context from these previous tokens)

While the model weights are loaded once and stay fixed, the KV cache is dynamic. The KV cache grows with every token. We need to leave room for the KV cache, like 2.5 GB for 8k tokens or 10 GB for 32k tokens.

### Optimizing Memory

- Weights: Quantization can be used to compress the weights.
- KV Cache: Previous methods have been reserving one big block for the KV cache even we dont know how big the KV cache will be at the end. Now with *PagedAttention* we split the KV cache in fixed size block which they can sit everywhere in memory.

## Efficien LLM deplyoment

### Measure LLM reliability and performance

1. Accuracy SLOs:
   - Hallucinations
   - Off-brand responses
1. Inference Performance SLOs
   - Latency must not get in the way. **Time to first token (TTFT)** for the output. **Inter token latency (ITL)**, the average time between generating consectuive tokens in the output, excluding the first token. **Request latency**, the total time end to end. **Troughput**, the average number of output tokens generated per second across all requests.
1.       

### What are the hardware requirements for running a LLM

1. GPU memory:
   - Model weights have always fixed space independent of serving 1 or 100 users
   - KV cache grows with every token in every active request
     -   One long-context request (32k tokens) needs 10GB KV Cache, that means with 180GB I could serve 18 long-context users in parallel (32,768 tokens are about the half of the book "The Great Gatsby" when assuming that a token is about 3/4 of an English word, or 50 single-spaced pages)

Unfortunately we have always tradeoffs, the tradeoff triangle for LLM deployments is between **COST - ACCURACY - PERFORMANCE**. We can pick two of them, not all three.

---


## Fundamentals

### Inference

The serve one or many User with an Inference API where the user ask using a prompt and get some response, we need:
1. The model
2. The inference server (f.i. vLLM)
3. Hardware accelerator (f.i. nvidia GPU)

### Text generation

1.1. Lets pass "The quick brown" tokens to the model
1.2. The model processes the input by a *forward pass* and predicts the next token "fox" which gets appended to the **input**
2.1. Next run with "The quick brown fox" forward pass
2.2. Model predicts "jumps"
X.1. ...
X.2. ...
X.3. Last token is an *End of sequence token* that signals that it is done

That means, that when the model is generating a 500 tokens long response, each token needs to propagate through the model, that means the model runs 500 times.

> [!NOTE]
> Not the token itself propates through the model, the vector representation of the token does.

### Forward pass through the model

The transformer layer has two main parts:
1. *Self-Attention* is where token exchange information with each other
1. *Feed-Forward Network* processes each tokens representation further

At the end output goes into the LM Head which returns a score for the models internal representation for the possible next token. The highest scoring token is the prediction. Then the prediction gets appended to the input and the whole stack runs again for the next forward pass.

<img width="1250" height="560" alt="image" src="https://github.com/user-attachments/assets/e5d8c7a1-630e-4b6c-a5d2-cf338a6d1453" />

### Linear Layers

How do the *q_proj*, *k_proj*, *v_proj*, *o_proj* layers work:

<img width="824" height="151" alt="image" src="https://github.com/user-attachments/assets/eef9a2c2-4924-40c9-8b01-7010a25c9d2b" />

**Q Projection**

Using the query *Q4* (The four means token 4, so the token 4 is the query) and compare it agains every token so far including itself using a dot product. The higher dot product means *the token is relevant for token four*, a lower dot product means is less relevant.

<img width="525" height="226" alt="image" src="https://github.com/user-attachments/assets/edabe575-e37c-4a65-87a0-fef40d4b885c" />

**V Projection**

Takes the weighted sum of all the value vectors using those weigts. The result is a single vector which is enriched with the context from the rest of the sequence. That vector passes through the *o_proj*

<img width="472" height="264" alt="image" src="https://github.com/user-attachments/assets/9007c28b-cf2c-4752-92d8-ab37848666ca" />

**The KV cache size calculation**
As you can see in the diagrams above, we need the complete history of the Keys and Values. Because the K and V is only computed once it can be cached, only the lagtest KV is calculated.

Lets take LLama 3 70B as example and use BF16 precision:

<img width="726" height="564" alt="image" src="https://github.com/user-attachments/assets/aef03bee-c1a2-4eda-bf0e-7f1d681c80ae" />

Managing the KV cache is the singlest biggest job of a production inference server

> [!NOTE]
> The Q,K,V, the KV cache and the model weigts are all tensors

### GPU Memory Hierarchy

HBM = High Bandwidth Memory
SRAM = Static RAM

<img width="804" height="558" alt="image" src="https://github.com/user-attachments/assets/50af2e29-86aa-4785-b334-8fb635250774" />

The Models weigts and the KV cache live in VRAM. Every forward pass is passed to the SRAM (discarded when computations are done), means chunks of weights, chunks of KV cache and intermediate results live in SRAM.

---

## Compression

### Quantization

- LLMs are typically released in BF16 (Brain Floating Point 16 bit (2 Byte) which is more efficient than FP16)
- Quantization often supports FP8, INT8, INT4 (FP=Floating-Point, INT=Integer)

<img width="498" height="350" alt="image" src="https://github.com/user-attachments/assets/14c98439-2770-4182-9b66-8d51f10b77d9" />

Quantization focuses on the linear layers of the self-attention blocks and feed-forward networks and ALSO on the input actionations. NOT the embedding layer at the start and the LM Head at the end.

The *input activations* are the tensor that gets multiplied by the weights in every linear layer. For example, the tensor representation of a token which is the input for q,k and v. And/Or the weighted sum output which flows into o_proj.

### Sparsification

Removes unnecessary weights
- Typically 2 of 4, where 2 out of every 4 values within a model weight tensor are set to 0 -> Reducing memory and computation cost


### Advantages of Weight and Activation Quantization

- Lower latency for data movement. Reduce data from HBM to SRAM, so not so much data needs to be moved
- The tensor cores can do more operations per second when numbers are in lower-precision format

### Quantization Schemes

- **Weight only quantization (ex. W8A16)**: The weights are INT8 and during inference the activation remain BF16. Less data to pull between VRAM and SRAM but NO tensor core speedup, because the matrix multiplication needs to be done with the same precision BF16xBF16.
- **Weight & Activation Quantization (ex. W8A8)**: Reduces data from HBM to SRAM and ALSO use tensor cores with more FLOPS (Floating point operations per second)

### Quantization methods

**GPTQ** (Generative Pre-trained Transformer Quantization) is a post-training weight quantization method that compresses model weights to low precision, typically 4-bit. It works layer by layer, quantizing weights one column at a time while using second-order (Hessian) information to adjust the remaining unquantized weights so they compensate for the error introduced. This greedy error-correction approach lets GPTQ push down to 3–4 bits with relatively little accuracy loss, and because it only needs a small calibration dataset and a single pass, it is fast and reproducible. (A calibration dataset is a small set of sample text run through the model during quantization so the algorithm can observe the real distribution of weights and activations and pick quantization parameters that minimize error on data resembling actual use.) The result is a model whose per-token weight reads from VRAM shrink roughly fourfold versus FP16, directly easing the memory-bandwidth bottleneck that dominates decode.

**AWQ** (Activation-aware Weight Quantization) takes a different angle: instead of correcting errors after the fact, it identifies which weight channels matter most by observing the magnitude of the *activations* that flow through them. A small fraction of "salient" channels disproportionately affect output quality, so AWQ scales those channels before quantization to protect them, leaving the rest to be quantized normally. This activation-aware scaling avoids the need for expensive Hessian computation and tends to generalize better across domains, often preserving accuracy slightly better than GPTQ at the same bit width. Both methods land in the same place practically — 4-bit weights, ~4× less bandwidth per token — but AWQ leans on activation statistics while GPTQ leans on weight-error correction.

**Round-to-nearest** Rounds each weight independently and moves on. Fast, needs no data, but the errors accumulate across millions of weights and degrade accuracy — especially below 8-bit.

**Sparse-GPT** Is a one-shot pruning method from the same research line as GPTQ (Frantar and Alistarh), and it shares GPTQ's core machinery — the difference is that instead of rounding weights to a low-precision grid, it removes weights entirely by setting them to zero. It can prune a large language model to around 50% sparsity in a single pass, with no retraining, while keeping accuracy loss small.

---

## Coding Example

- Using `llm-compressor` library in python to apply GPTQ to produce W4A16 quantized model. The lib is a toolkit from the vLLM project.
- The core API is `oneshot` in `llm-compressor`
- 


