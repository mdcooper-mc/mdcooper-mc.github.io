+++
title = "Reinforcement Learning & Autonomous Agents"
description = "Building goal-oriented autonomous systems using Deep Q-Networks (DQN) and Physics-Informed Neural Networks (PINNs)."
date = 2024-03-27
math = true
+++

Reinforcement Learning (RL) represents a fundamental shift in how machines learn. Unlike supervised learning, where a
model is provided with a "ground truth" label for every input, an RL agent must discover which actions yield the highest
reward by interacting with a dynamic environment.

I developed a set of autonomous agents capable of navigating complex, stochastic environments. This project explores the
transition from traditional Q-learning to Deep Q-Networks (DQN) and the integration of physical laws into the learning
process through Physics-Informed Reinforcement Learning (PIRL).

---

## The Problem: The Sparse Reward & State Explosion

In a simple grid world, we can store the "value" of every possible action in every possible state in a lookup table (
Q-Table). However, as the environment grows—or becomes continuous—this table becomes impossibly large. Furthermore,
rewards in the real world are often **sparse**: an agent might take 100 actions before receiving a single "success"
signal.

To build a robust agent, the system must solve:

1. **The Credit Assignment Problem**: Which of the 100 actions actually led to the success?
2. **Exploration vs. Exploitation**: When should the agent try something new vs. doing what it knows works?
3. **Generalization**: How can an agent handle a state it has never seen before?

---

## Architecture: Deep Q-Networks (DQN)

To handle high-dimensional state spaces, I implemented a **Deep Q-Network**. Instead of a table, a neural network acts
as a function approximator, mapping a state vector to the expected "Q-values" for all possible actions.

```python
class DQN(nn.Module):
    def __init__(self, in_states, out_actions):
        super(DQN, self).__init__()
        # Multi-layer architecture to capture spatial relationships
        self.fc1 = nn.Linear(in_states, 64)
        self.fc2 = nn.Linear(64, 128)
        self.out = nn.Linear(128, out_actions)

    def forward(self, x):
        x = F.relu(self.fc1(x))
        x = F.relu(self.fc2(x))
        return self.out(x)
```

### The Bellman Equation

The network is trained using the **Bellman Equation**, which recursively defines the value of a state-action pair based
on the immediate reward and the discounted value of the next state.

$$
Q(s, a) \approx r + \gamma \max_{a'} Q(s', a')
$$

---

## Stability Engineering: Experience Replay

Training a neural network on sequential data is notoriously unstable because consecutive samples are highly correlated (
the state at time $T+1$ is very similar to the state at time $T$). This violates the i.i.d. assumption required for
stable gradient descent.

I implemented **Experience Replay** to break these correlations. The agent's experiences—$(s, a, r, s', done)$—are
stored in a circular buffer, and the network is trained on random mini-batches sampled from this history.

```python
class ReplayMemory:
    def __init__(self, capacity):
        self.memory = deque(maxlen=capacity)

    def push(self, transition):
        self.memory.append(transition)

    def sample(self, batch_size):
        return random.sample(self.memory, batch_size)
```

---

## Physics-Informed Reinforcement Learning (PIRL)

While standard RL agents are "black boxes," many real-world applications (robotics, drones) are governed by the laws of
physics. I explored **Physics-Informed Reinforcement Learning** to ensure agents act not only optimally but also
physically accurately.

### The Physics Residual Loss

Using **Physics-Informed Neural Networks (PINNs)**, I augmented the standard RL loss with a "physics residual." By
calculating the temporal derivative of the agent's state using automatic differentiation, I can penalize actions that
violate kinematic or conservation laws.

$$
L_{total} = L_{RL} + \lambda \cdot \|\frac{\partial \hat{s}}{\partial t} - f(s, u)\|^2
$$

This ensures the agent "imagines" future trajectories that are physically plausible, leading to safer deployment and
higher data efficiency.

---

## Impact

Using the **Gymnasium** framework, I successfully trained agents to navigate the stochastic "8x8 FrozenLake"
environment. By combining DQN with Experience Replay and Temperature-based Softmax exploration, the agents developed
robust navigation policies that outperformed simple heuristic methods. The integration of PINNs further demonstrated how
domain knowledge can be used to constrain and accelerate the learning process.

---

## Skills

**Autonomous Agents**  
Deep Q-Networks (DQN) | Policy Gradient | Exploration-Exploitation | Temperature Annealing | Experience Replay

**Environments & Physics**  
Gymnasium (OpenAI Gym) | Physics-Informed Neural Networks (PINNs) | Kinematic Constraints | Reward Shaping

**Optimisation & Theory**  
Bellman Optimality | Temporal Difference (TD) Learning | Markov Decision Processes (MDP) | Discount Factors

**Mathematical Foundations**  
Stochastic Environments | Automatic Differentiation | Partial Differential Equations | Vectorised State Handling
