+++
title = "Transformer Architecture: Building GPT-2 from Scratch"
description = "A first-principles implementation of the Transformer decoder architecture, featuring causal self-attention, multi-head attention, and professional scaling techniques."
date = 2024-03-25
math = true
+++

The Transformer architecture is the engine behind modern generative AI. Unlike previous sequential models (RNNs) or
hierarchical models (WaveNets) that rely on fixed patterns, the Transformer uses **Self-Attention** to dynamically
weight the importance of every token in a sequence relative to every other token.

I built a complete Transformer decoder—the same architecture used in the GPT family—from the ground up. This project
involved implementing the core attention mechanism, modular multi-head blocks, and the engineering required to stabilize
deep residual networks.

---

## The Problem: The Receptive Field Bottleneck

Previous architectures I explored, like WaveNet, used a fixed "context window" (e.g., 8 tokens) to predict the next
word. While effective for short patterns, these models struggle with long-range dependencies—for example, remembering
the subject of a sentence that started 50 words ago.

To handle natural language at scale, a model needs:

1. **Global Context**: The ability for any token to "look at" any other token, regardless of distance.
2. **Parallelism**: The ability to process entire sequences at once on a GPU, rather than one step at a time.
3. **Dynamic Weighting**: A way to decide which parts of the past are relevant *right now*.

---

## Architecture: Causal Self-Attention

The core innovation of the Transformer is the **Query, Key, and Value (KQV)** mechanism. For every token in the input,
we compute three distinct vectors:

- **Query ($Q$)**: "What am I looking for?" (e.g., a verb looking for its subject).
- **Key ($K$)**: "What information do I contain?" (e.g., a subject offering its attributes).
- **Value ($V$)**: "The actual information to be passed along if I'm relevant."

The "attention" is calculated by comparing $Q$ against all $K$ vectors via a dot product, resulting in an affinity
matrix.

```python
class Head(nn.Module):
    """ One head of self-attention """
    def forward(self, x):
        B, T, C = x.shape
        k = self.key(x)   # (B, T, head_size)
        q = self.query(x) # (B, T, head_size)
        
        # Scaled dot-product attention: q @ k^T / sqrt(d_k)
        # Scaling by sqrt(head_size) preserves unit variance for the Softmax
        wei = q @ k.transpose(-2, -1) * k.shape[-1]**-0.5
        
        # Causal mask: ensure tokens can only look at the past
        # Future tokens are set to -inf so they receive 0 weight after Softmax
        wei = wei.masked_fill(self.tril[:T, :T] == 0, float('-inf'))
        wei = F.softmax(wei, dim=-1)
        
        # Aggregated context: weighted sum of the Values
        v = self.value(x)
        out = wei @ v 
        return out
```

---

## Scaling: Multi-Head Attention & Residual Blocks

A single attention head might only focus on one type of relationship (e.g., grammar). To capture multiple relationships
simultaneously (e.g., grammar, tone, and logic), I implemented **Multi-Head Attention (MHA)**, running several heads in
parallel and concatenating their results.

These heads are organized into **Transformer Blocks**, which alternate between "communication" (self-attention) and "
computation" (a feedforward neural network).

### Stability Patterns

To train deep Transformers (e.g., 6 or 12 layers) without gradients vanishing, I used two professional patterns:

1. **Residual (Skip) Connections**: Adding the input of a block back to its output ($x + f(x)$). This creates a "
   gradient highway" that allows signals to flow through the network unimpeded.
2. **Pre-LayerNorm**: Applying Layer Normalization *before* the attention and feedforward layers (as in GPT-2). This
   keeps activations stable and prevents the "exploding gradient" problem seen in the original "Post-LayerNorm"
   Transformer.

```python
class Block(nn.Module):
    """ Transformer block: communication followed by computation """
    def forward(self, x):
        # Communication (Self-Attention) with Residual Connection
        x = x + self.sa(self.ln1(x))
        # Computation (Feed-Forward) with Residual Connection
        x = x + self.ffwd(self.ln2(x))
        return x
```

---

## Results: Shakespearean Generation

I trained this architecture on the complete works of Shakespeare. Using a character-level tokenizer, the model learned
to:

- Form coherent words and names (e.g., "ROMEO", "JULIET").
- Capture the structure of a play (Dialogue, Stage Directions).
- Mimic the poetic meter and archaic vocabulary of the source text.

While simple compared to GPT-4, this model proved that the Transformer architecture can emerge complex patterns from raw
data using only a set of attention heads and residual connections.

---

## Skills

- **Neural Architectures**: Transformer Decoder, Causal Self-Attention, Multi-Head Attention (MHA), Residual Networks, Feed-Forward Networks.
- **AI Engineering**: Queries, Keys, and Values (KQV), Scaled Dot-Product Attention, Causal Masking, Layer Normalization (Pre-LayerNorm).
- **Mathematical Foundations**: Matrix Calculus, $O(T^2)$ Complexity Analysis, Softmax Stability, Variance Scaling.
- **Software Engineering**: PyTorch Module Design, Modular Architecture Patterns, GPU Memory Optimization, Sequence Modelling.
