---
title: "optimizer state"
date: "2026-06-20"
tags: ["deeplearning", "optimizers"]
---

### The Memory Bank of Deep Learning: Demanding Better from Neural Network Optimizers

Training a neural network is an exercise in navigation. You are dropped into a massive, foggy landscape with millions of valleys and ridges, and your only goal is to find the lowest point the absolute minimum loss. How you choose to take your steps matters just as much as where you are trying to go.

To appreciate why modern networks train successfully, we have to look closely at the mathematical memory that drives them: the optimizer state.

---

### 1. Why Vanilla SGD Fails

In the early days of deep learning, models relied on Vanilla Stochastic Gradient Descent (SGD). It is a purely reactive algorithm. It looks at the slope right under its feet, takes a step down that slope, and immediately forgets everything else.

The formula is entirely memoryless:

$$
\theta_{t+1} = \theta_t - \eta \cdot \nabla_t
$$

Where $\theta$ represents the model weights, $\eta$ is the learning rate, and $\nabla_t$ is the gradient (the current slope).

### Failure Example: The Ravine Problem

Imagine a landscape that looks like a long, narrow canyon. The floor of the canyon slopes very gently toward the exit (the true minimum), but the walls of the canyon are incredibly steep.

If you drop a Vanilla SGD optimizer into this canyon, here is what happens:

1. It calculates the gradient. The slope pointing down the steep canyon walls is massive compared to the gentle slope leading toward the exit.
2. SGD takes a massive step based on that steep slope. It violently hurtles across the canyon floor and smashes into the opposite wall.
3. On the next step, it sees a massive slope pointing back the other way. It bounds across the floor again.

Because SGD has no memory of its past trajectory, it spends all its energy oscillating wildly back and forth between the canyon walls. It makes almost zero progress down the length of the canyon toward the exit. In complex modern AI models, this exact behavior causes training to stall out completely.

---

### 2. Enter AdamW: The Optimizer with History

To solve this lack of awareness, modern training relies on AdamW (Adam with decoupled Weight decay). AdamW succeeds because it keeps records. For every single parameter in your model, the optimizer state tracks historical statistics.

Instead of treating every weight the same, it creates a custom, adaptive update for every individual parameter using two historical metrics: the 1st momentum and the 2nd momentum.

---

### 3. The Mechanics of the 1st and 2nd Momentum

The core of the optimizer state consists of two vectors that act as filters for the noisy gradients generated during training.

### The 1st Momentum ($m_t$): The Direction Manager

The 1st momentum tracks the directional trend of the gradients over time. It functions exactly like physical momentum.

$$
m_t = \beta_1 \cdot m_{t-1} + (1 - \beta_1) \cdot \nabla_t
$$

- **The Work:** It acts as an exponential moving average. If the gradient points in the same direction for several steps, the optimizer speeds up. If the gradient starts oscillating back and forth (like bouncing off canyon walls), the opposing signs cancel each other out in the running average.
- **The Result:** It flattens out noisy, erratic bounces and keeps the model pushing steadily along the true path.

### The 2nd Momentum ($v_t$): The Scale Manager

The 2nd momentum tracks the volatility, or the sheer variance, of the gradients for each parameter.

$$
v_t = \beta_2 \cdot v_{t-1} + (1 - \beta_2) \cdot (\nabla_t)^2
$$

- **The Work:** By squaring the gradient ($\nabla_t^2$), the optimizer strips away the directional sign and focuses purely on magnitude.
- **The Result:** It acts as an automatic speed regulator. In the final update equation, the optimizer divides the step by $\sqrt{v_t}$. If a parameter is wildly volatile with massive gradients, dividing by a large $\sqrt{v_t}$ shrinks its step size to keep it safe. If a parameter has tiny, rare gradients, dividing by a tiny $\sqrt{v_t}$ boosts its step size so it does not get left behind. This is what makes the learning rate "adaptive."

---

### 4. The Complete AdamW Update Equation

When we combine the direction manager (1st momentum) and the scale manager (2nd momentum), we get the unified mathematical step that updates the weights. But first, the raw momentum estimates must be corrected for initialization bias.

### Bias Correction

Because $m_t$ and $v_t$ are initialized at zero, the early estimates are artificially pulled toward zero. Bias correction compensates for this:

$$
\hat{m}_t = \frac{m_t}{1 - \beta_1^t}, \quad \hat{v}_t = \frac{v_t}{1 - \beta_2^t}
$$

As $t$ grows large, $\beta_1^t \to 0$ and $\beta_2^t \to 0$, so the corrections fade and the estimates converge to their true values. In the early steps, however, they provide an essential rescaling that prevents the optimizer from taking negligibly small steps at the start of training.

### The Full Update Rule

$$
\theta_{t+1} = \theta_t - \eta \cdot \frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} - \eta \lambda \theta_t
$$

- $\eta$ is your global learning rate.
- $\frac{\hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon}$ is the bias-corrected adaptive step: the smoothed direction divided by the smoothed volatility scale. The tiny constant $\epsilon$ prevents division by zero.
- $\eta \lambda \theta_t$ is the **decoupled** weight decay term the "W" in AdamW. Crucially, it is applied as a **separate subtraction**, not folded inside the adaptive fraction. This is the central distinction between Adam and AdamW: in vanilla Adam, L2 regularization is mixed into the gradient before the adaptive scaling, which causes the effective decay rate to vary per parameter. AdamW decouples it entirely, so every weight decays at a uniform, predictable rate regardless of its gradient history.

---

### 5. FP32 Master Weights in Mixed-Precision Training

When scaling up large language models, storing everything in 32-bit floating-point numbers (FP32) consumes too much GPU memory. To save space, we use mixed-precision training, where the active forward and backward passes run using 16-bit brain floats (BF16) or half-precision floats (FP16).

This introduces a massive problem called **underflow**.

Because 16-bit floats have limited precision, they cannot represent incredibly small numbers. During an update, the step size ( $\eta \times \text{step}$ ) is often a tiny fraction. When you try to subtract a tiny 16-bit fraction from a 16-bit weight, the precision limit causes that tiny fraction to round down to absolute zero. The weight never changes, and the model stops learning.

### The Solution: The FP32 Master Weight Copy

To bypass underflow, the optimizer state holds a hidden, full-precision FP32 copy of all the model weights.

1. The model does its fast calculations in 16-bit to save memory.
2. The gradients are computed and passed to the optimizer.
3. The optimizer calculates the 1st and 2nd moments.
4. The resulting tiny update step is applied directly to the **FP32 master weights**. Because it is full 32-bit precision, the tiny numbers are preserved without rounding to zero.
5. The freshly updated FP32 weights are then cast back down to 16-bit for the next round of training.

Without this FP32 master weight copy inside the optimizer state, training modern architectures in low precision would be mathematically impossible.