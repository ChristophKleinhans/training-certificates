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

> [!INFO]
> The Q,K,V, the KV cache and the model weigts are all tensors

### GPU Memory Hierarchy

HBM = High Bandwidth Memory
SRAM = Static RAM

<img width="804" height="558" alt="image" src="https://github.com/user-attachments/assets/50af2e29-86aa-4785-b334-8fb635250774" />

