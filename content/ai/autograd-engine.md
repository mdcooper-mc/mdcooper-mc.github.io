+++
title = "Building a Scalar-Valued Autograd Engine from First Principles"
description = "A first-principles implementation of a DAG-based automatic differentiation engine in Python, supporting backpropagation and training simple neural networks."
date = 2024-03-20
math = true
+++

Modern deep learning frameworks like PyTorch and TensorFlow abstract away the complex calculus involved in model
training. While efficient, this abstraction can lead to a "black box" understanding of how models actually learn. To
bridge this gap, I built a fully functional, scalar-valued automatic differentiation engine from scratch.

This engine, which I've termed my "Value Engine," implements backpropagation over a Directed Acyclic Graph (DAG) and
supports the training of multi-layer perceptrons (MLP) using only basic Python and scalar math.

---

## The Problem

Understanding backpropagation is one thing; implementing it correctly in a way that handles arbitrary computational
graphs is another. Standard floating-point numbers in Python have no "memory" of where they came from or how they were
calculated. To perform automatic differentiation, we need a data structure that not only stores a value but also tracks
its "ancestry" (the operations and inputs that created it) and its "influence" (its gradient with respect to a final
output).

The challenge was to create a system that could:

1. Construct a computational graph in real-time as operations are performed.
2. Automatically apply the chain rule of calculus to propagate gradients.
3. Handle complex branching paths where a single variable influences multiple downstream outputs without over-counting
   or under-counting its total influence.

---

## Architecture: The Value Engine

At the core of the project is the `Value` class. Each `Value` object acts as a node in a computational graph, storing:

- `data`: The actual numerical value (scalar).
- `grad`: The derivative of the final output (typically the Loss) with respect to this value.
- `_backward`: A function that defines how to propagate the gradient from this node to its immediate children.
- `_prev`: A set of parent nodes in the graph.

### Implementing Atomic Operations

Every mathematical operation (addition, multiplication, exponentiation, etc.) is overloaded to produce a new `Value`
object while simultaneously defining its own gradient logic.

```python
class Value:
    def __init__(self, data, _children=(), _op='', label=''):
        self.data = data
        self.grad = 0.0
        self._backward = lambda: None
        self._prev = set(_children)
        self._op = _op

    def __add__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data + other.data, (self, other), '+')

        def _backward():
            # The derivative of a sum (a + b) with respect to each input is 1.0
            # We use += to accumulate gradients from multiple paths (Multivariate Chain Rule)
            self.grad += 1.0 * out.grad
            other.grad += 1.0 * out.grad

        out._backward = _backward
        return out

    def __mul__(self, other):
        other = other if isinstance(other, Value) else Value(other)
        out = Value(self.data * other.data, (self, other), '*')

        def _backward():
            # Product rule: d(a*b)/da = b and d(a*b)/db = a
            self.grad += other.data * out.grad
            other.grad += self.data * out.grad

        out._backward = _backward
        return out
```

By using `+=` for gradient updates, the engine correctly handles the **Multivariate Chain Rule**. If a variable is used
twice in a calculation, its total gradient is the sum of the gradients from both paths.

---

## Gradient Propagation: Topological Sorting

Backpropagation must be performed in a specific order: from the output back to the inputs. In a graph, this requires a *
*Topological Sort**. The engine performs a depth-first search starting from the final output (the Loss) to ensure that
every node is processed only after all the nodes it influences have had their gradients calculated.

```python
    def backward(self):
        # Build topological order
        topo = []
        visited = set()
        def build_topo(v):
            if v not in visited:
                visited.add(v)
                for child in v._prev:
                    build_topo(child)
                topo.append(v)
        build_topo(self)

        # Base case: dLoss/dLoss = 1.0
        self.grad = 1.0
        for node in reversed(topo):
            node._backward()
```

---

## Building a Neural Network

With the scalar engine complete, I constructed a modular neural network library on top of it, consisting of:

- **Neuron**: Manages weights and a bias, performs a weighted sum, and applies a non-linear activation (e.g., `tanh` or
  `ReLU`).
- **Layer**: A collection of neurons operating in parallel.
- **MLP**: A sequence of layers forming a deep network.

Because every parameter (weight and bias) in the network is a `Value` object, the entire network is "differentiable by
design."

```python
# Training loop snippet
for k in range(20):
    # Forward pass: predict and calculate loss
    ypred = [model(x) for x in xs]
    loss = sum((yout - ygt)**2 for ygt, yout in zip(ys, ypred))
    
    # Backward pass: zero gradients and propagate
    for p in model.parameters():
        p.grad = 0.0
    loss.backward()
    
    # Update weights (SGD)
    for p in model.parameters():
        p.data += -0.01 * p.grad
```

---

## Impact & Insights

This first-principles approach provided several key insights that are often obscured by high-level frameworks:

1. **The Criticality of Initialisation**: I observed how poor initial weight values could lead to "exploding" or "
   vanishing" gradients even in tiny networks.
2. **Computational Efficiency**: While scalar-level differentiation is conceptually clear, it is computationally
   expensive in Python. This underscored the necessity of the vectorized (Tensor-based) operations used in production
   frameworks.
3. **Debugging the Gradient**: By visualizing the computational graph using Graphviz, I could see exactly how the loss
   signal flowed back through individual neurons, making the learning process entirely transparent.

---

## Skills

**Mathematics & Theory**  
Multivariable Calculus | Chain Rule | Gradient Descent | Topological Sort | Directed Acyclic Graphs (DAG)

**AI Engineering**  
Automatic Differentiation | Backpropagation | Multi-Layer Perceptron (MLP) | Loss Functions (MSE) | Non-Linear
Activations (Tanh/ReLU)

**Software Engineering**  
Object-Oriented Design | Custom Computational Engines | Modular API Design | Recursive Function Calls | Graph
Visualisation
