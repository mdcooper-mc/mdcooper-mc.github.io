+++
title = "Language Modelling"
description = "A technical progression from simple statistical character-level models to deep hierarchical neural networks for language generation."
date = 2024-03-21
math = true
+++

To predict the next character or word in a sequence, a model must "remember" what came before. I explored this
progression from first principles, starting with basic statistical counting and evolving into deep, hierarchical neural
architectures that capture complex linguistic structures.

This project covers the transition from frequentist models to neural-based optimization, the introduction of continuous
embeddings, and the engineering required to stabilize deep networks.

---

## The Problem: The Curse of Dimensionality

In a simple **Bigram Model**, we predict the next character based solely on the current one. If we have 27 possible
characters (a-z plus space), we can store all possible transitions in a $27 \times 27$ lookup table.

However, language is not a bigram process. To predict the next character accurately, we often need to look back 5, 10,
or 20 characters. If we try to extend the lookup table approach to a context of 10 characters, we would need a table
with $27^{10}$ entries—approximately $205$ trillion cells. This is the **Curse of Dimensionality**: as the context
window grows, the state space explodes exponentially, making pure statistical counting impossible.

---

## Phase 1: Statistical vs. Neural Bigrams

I first implemented a bigram model using a simple counting matrix. By normalizing the rows of this matrix, we get a
probability distribution: $P(\text{next} | \text{current})$.

The transition to AI occurs when we frame this "counting" problem as an optimization task. Instead of counting, we
initialize a weight matrix $W$ and use **Gradient Descent** to minimize the **Negative Log-Likelihood (NLL)** of the
training data.

```python
# The Neural Bigram approach
x_encoded = F.one_hot(x, num_classes=27).float()
logits = x_encoded @ W  # (B, 27) @ (27, 27) -> (B, 27)
counts = logits.exp()
probs = counts / counts.sum(1, keepdims=True)
loss = -probs[torch.arange(num_examples), y].log().mean()
```

This equivalence is profound: a single-layer neural network with a softmax output is mathematically identical to a
bigram lookup table, but it provides a differentiable framework that can be extended to much more complex architectures.

---

## Phase 2: Continuous Embeddings & MLPs

To solve the Curse of Dimensionality, I moved from one-hot encoding to **Continuous Embeddings**. Instead of treating
each character as an isolated category, we map them into a low-dimensional vector space (e.g., 2 dimensions or 30
dimensions).

In this space, the model can learn that "a", "e", "i", "o", and "u" are similar because they appear in similar contexts.
This allows for **generalization**: the model can make intelligent guesses about character combinations it has never
seen before by leveraging the geometric relationships in the embedding space.

I implemented a Multi-Layer Perceptron (MLP) that takes a context of $N$ characters, lookups their embeddings,
concatenates them, and passes them through a hidden layer with a `tanh` non-linearity.

```python
# Embedding lookup and flattening
emb = C[X]  # (B, context_size, n_embd)
# Concatenate the context: (B, context_size * n_embd)
h = torch.tanh(emb.view(-1, n_inputs) @ W1 + b1)
logits = h @ W2 + b2
```

---

## Phase 3: Stabilizing Deep Dynamics

As the MLP grew deeper, training became unstable. I addressed this through two critical engineering patterns:

### 1. Kaiming Initialisation

If weights are initialized too large, the `tanh` activations saturate at -1 or 1, causing "dead gradients" where the
model stops learning. If they are too small, the signal vanishes. I implemented **Kaiming Initialisation**, scaling
weights by $\text{gain} / \sqrt{\text{fan\_in}}$ to ensure the variance of activations remains constant across layers.

### 2. Batch Normalisation

To further stabilize the "internal covariate shift," I implemented **Batch Normalisation**. By forcing the activations
in each mini-batch to have zero mean and unit variance, the loss landscape becomes much smoother, allowing for
significantly higher learning rates and faster convergence.

```python
# BatchNorm implementation logic
bn_mean = h_preact.mean(0, keepdim=True)
bn_var = h_preact.var(0, keepdim=True)
h_normalized = (h_preact - bn_mean) / torch.sqrt(bn_var + 1e-5)
h_out = gamma * h_normalized + beta  # scale and shift
```

---

## Phase 4: Hierarchical WaveNet Architectures

To efficiently process even longer context windows (e.g., 8+ characters), a flat MLP becomes too "top-heavy." I
implemented a **Hierarchical Architecture** inspired by WaveNet.

Instead of squashing the entire context into one hidden layer, the model processes tokens in pairs across multiple
layers. The first layer processes 8 tokens into 4 representations; the second layer processes those 4 into 2; and the
final layer produces the prediction. This tree-like structure preserves temporal locality and allows the model to learn
hierarchical patterns (letters $\rightarrow$ syllables $\rightarrow$ words) with far fewer parameters than a flat
network.

```python
# Modular Hierarchical API
model = Sequential([
    Embedding(vocab_size, n_embd),
    FlattenConsecutive(2), Linear(n_embd * 2, n_hidden), BatchNorm(n_hidden), Tanh(),
    FlattenConsecutive(2), Linear(n_hidden * 2, n_hidden), BatchNorm(n_hidden), Tanh(),
    FlattenConsecutive(2), Linear(n_hidden * 2, n_hidden), BatchNorm(n_hidden), Tanh(),
    Linear(n_hidden, vocab_size)
])
```

---

## Impact & Results

By evolving the architecture from a simple bigram to a hierarchical WaveNet, I reduced the model's validation loss
significantly while maintaining a context window 8x larger than the original MLP. More importantly, I built a modular,
object-oriented framework for deep learning that mirrors the internal design of professional libraries like PyTorch.

---

## Skills and Technologies

- **Neural Architectures**: Bigram Models, Multi-Layer Perceptrons (MLP), Hierarchical WaveNet, Residual Connections.
- **Optimization & Training**: Negative Log-Likelihood (NLL), Stochastic Gradient Descent (SGD), Kaiming Initialisation, Batch Normalisation.
- **Deep Learning Engineering**: Continuous Embeddings, One-hot Encoding, Modular API Design, Learning Rate Scheduling.
- **Diagnostics & Health**: Activation Histograms, Gradient Flow Analysis, Diagnostic Visualisations.
- **Mathematical Foundations**: Information Theory, Multivariable Calculus, Vector Space Geometry.
