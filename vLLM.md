From deeplearning.ai (https://learn.deeplearning.ai/courses/fast-and-efficient-llm-inference-with-vllm/) the *Fast & Efficient LLM Inference with vLLM* training

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
     
## Fundamentals

### 
