---
layout: post
name: Deriving the Effective Memory Length of an EMA
birth: 2026-09-27
---

## The Claim Being Derived

[A Roadmap to Gradient Descent Optimizers](roadmap-to-gradient-descent-optimizers) introduces the first moment and the second moment. Both are built from the same shape of recurrence — an exponential moving average (EMA):

$$\begin{equation} y^{t+1} = \beta\, y^{t} + (1-\beta)\, x^{t}, \qquad y^{0}=0 \end{equation}$$

This recurrence acts like a weighted average over roughly the last $1/(1-\beta)$ steps.

> For example, $\beta=0.9$ gives $1/(1-\beta)=10$, so we can say that $y^{t+1}$ is shaped mainly by the last 10 iterations. Every past value technically still appears in the sum, but older contributions have decayed enough to be practically negligible.

This note derives that claim from scratch.

---

## Step 1: Unroll the Recurrence

To see how much each past input $x$ contributes to $y^{t+1}$, substitute the recurrence into itself, one step at a time:

$$\begin{aligned}
y^{t+1} &= (1-\beta)\,x^{t} + \beta\,y^{t} \\
&= (1-\beta)\,x^{t} + \beta\big[(1-\beta)\,x^{t-1} + \beta\,y^{t-1}\big] \\
&= (1-\beta)\,x^{t} + \beta(1-\beta)\,x^{t-1} + \beta^2\,y^{t-1} \\
&= (1-\beta)\,x^{t} + \beta(1-\beta)\,x^{t-1} + \beta^2(1-\beta)\,x^{t-2} + \beta^3\,y^{t-2}
\end{aligned}$$

Each substitution pushes the leftover $y$ term one step further back and multiplies it by another $\beta$. Continuing all the way down to $y^0$:

$$\begin{equation} y^{t+1} = (1-\beta)\sum_{k=0}^{t}\beta^{k}\,x^{t-k} \;+\; \beta^{t+1}y^{0} \end{equation}$$

Since $y^0=0$, the last term vanishes, leaving a weighted sum of all past inputs:

$$\begin{equation} y^{t+1} = \sum_{k=0}^{t} w_k\, x^{t-k}, \qquad w_k = (1-\beta)\,\beta^{k} \end{equation}$$

Here $k$ counts how many steps back an input lies. Every past input is still in the sum, but each step further back costs one more factor of $\beta$.

---

## Step 2: Compare Each Weight to the Newest One

What matters for "memory" is how fast the weights shrink, not their absolute size. So compare each weight to the newest one, $w_0$:

$$\begin{equation} \frac{w_k}{w_0} = \frac{(1-\beta)\,\beta^{k}}{(1-\beta)\,\beta^{0}} = \beta^{k} \end{equation}$$

The common factor $(1-\beta)$ cancels, and $\beta^0=1$. An input $k$ steps back therefore carries $\beta^k$ times the weight of the newest input. With $\beta=0.9$:

| Steps back $k$ | Weight $w_k$ | Relative weight $\beta^k$ |
|---|---|---|
| 0 | 0.1 | 1 |
| 1 | 0.09 | 0.9 |
| 2 | 0.081 | 0.81 |
| 10 | ≈ 0.035 | ≈ 0.35 |

---

## Step 3: Ask the Right Question

The relative weight $\beta^k$ never reaches zero, so "how many steps does the EMA remember?" has no sharp answer. A precise version of the question is:

> How many steps back does it take for the relative weight to drop to $1/e$?

This number of steps is called the **time constant**, written $\tau$. That is its definition: the number of steps an exponential decay takes to fall to $1/e$ of its starting value.

### Why $1/e$?

Any fraction $p$ would work. Solving $\beta^k = p$ gives

$$\begin{equation} k = \frac{\ln(1/p)}{-\ln\beta} \end{equation}$$

so every choice of $p$ gives the same dependence on $\beta$, just scaled by the constant $\ln(1/p)$. Choosing $p=1/2$ gives the familiar *half-life*, with constant $\ln 2 \approx 0.693$. Choosing $p=1/e$ is the one choice that makes the constant exactly $1$, which is why it gives the cleanest formula and is the standard convention in physics, engineering, and signal processing.

---

## Step 4: Solve for $\tau$ Exactly

Set the relative weight equal to $1/e$ and take the logarithm of both sides:

$$\beta^{\tau} = \frac{1}{e} \quad\Longrightarrow\quad \tau\ln\beta = -1$$

$$\begin{equation} \tau = \frac{-1}{\ln\beta} \end{equation}$$

This is the exact time constant. Since $0<\beta<1$, $\ln\beta$ is negative and $\tau$ is positive.

Another way to see that this is *the* natural length of the decay: rewrite $\beta^k$ as a power of $e$. Because $\beta = e^{\ln\beta}$,

$$\begin{equation} \beta^{k} = e^{k\ln\beta} = e^{-k/\tau} \end{equation}$$

which is the standard form of exponential decay, with $\tau$ appearing as its only parameter. Plugging in $k=\tau$ gives $e^{-1}$, consistent with the definition.

---

## Step 5: Approximate

The exact result $\tau = -1/\ln\beta$ is not convenient:

- It is hard to evaluate mentally. What is $-1/\ln 0.99$?
- It hides how $\tau$ scales with $\beta$.

In practice $\beta$ is close to $1$ ($0.9$, $0.99$, $0.999$), which invites an approximation. Write

$$\beta = 1-\varepsilon, \qquad \varepsilon = 1-\beta \text{ small}$$

and expand the logarithm as a Taylor series:

$$\begin{equation} \ln(1-\varepsilon) = -\varepsilon - \frac{\varepsilon^2}{2} - \frac{\varepsilon^3}{3} - \cdots \end{equation}$$

Keeping only the leading term, $\ln\beta \approx -(1-\beta)$. Substituting into $\tau = -1/\ln\beta$:

$$\begin{equation} \tau \approx \frac{1}{1-\beta} \end{equation}$$

This is the claim. It also makes the scaling obvious: $\tau$ is inversely proportional to how far $\beta$ is from $1$. Shrink $1-\beta$ by a factor of 10, and the memory grows by a factor of 10.

### How good is the approximation?

Keeping one more term of the expansion shows the size of the error:

$$\tau = \frac{1}{\varepsilon + \frac{\varepsilon^2}{2} + \cdots} = \frac{1}{\varepsilon}\cdot\frac{1}{1+\frac{\varepsilon}{2}+\cdots} \approx \frac{1}{\varepsilon}\left(1-\frac{\varepsilon}{2}\right)$$

$$\begin{equation} \tau \approx \frac{1}{1-\beta} - \frac{1}{2} \end{equation}$$

So $1/(1-\beta)$ overestimates the exact time constant by only about half a step:

| $\beta$ | Exact $-1/\ln\beta$ | Approximation $1/(1-\beta)$ | $\beta^{1/(1-\beta)}$ (vs. $1/e\approx 0.368$) |
|---|---|---|---|
| 0.9 | 9.49 | 10 | 0.349 |
| 0.99 | 99.50 | 100 | 0.366 |
| 0.999 | 999.50 | 1000 | 0.3677 |

The closer $\beta$ is to $1$, the smaller the relative error. For Adam's default values, this gives memory lengths of about $10$ steps for the first moment ($\beta_1=0.9$) and about $1000$ steps for the second moment ($\beta_2=0.999$).

## What the Claim Does and Doesn't Mean

The claim's wording deserves a closer look, because $1/(1-\beta)$ is a *time scale*, not a cutoff.

**Inputs at $\tau$ steps back are not yet negligible.** By definition, they still carry about $37\%$ of the newest input's weight. The last $\tau$ inputs together account for only about $1-\beta^{\tau} \approx 1-1/e \approx 63\%$ of the total weight. Older inputs do become negligible, but further back: the most recent $n$ inputs hold $1-\beta^n$ of the weight, so holding $95\%$ takes

$$n = \frac{\ln 0.05}{\ln\beta} \approx 3\tau$$

steps, which is about 28 steps when $\beta=0.9$.

**It is not the same as a plain average of the last $\tau$ inputs.** A simple moving average (SMA) over $N$ inputs gives each of them weight $1/N$ and ignores everything older. Matching an EMA to an SMA by either lag (the center of mass of the weights) or noise reduction (the sum of squared weights) gives the same answer:

$$\begin{equation} N = \frac{2}{1-\beta} - 1 \end{equation}$$

For $\beta=0.9$, that is $N=19$, not $10$. The EMA lags the input by $\beta/(1-\beta) = 9$ steps on average, while a 10-step SMA lags by only $4.5$.

So the accurate reading of the claim is: **the EMA's memory has a time scale of about $1/(1-\beta)$ steps**, meaning inputs that far back have decayed to about $1/e$ of the newest input's weight. It is a quick, reliable way to judge orders of magnitude ($\beta = 0.9, 0.99, 0.999 \to$ roughly $10, 100, 1000$ steps), not a statement that the EMA averages exactly that many inputs.

---

## Summary

1. Unrolling the recurrence shows that an input $k$ steps back has weight $(1-\beta)\beta^k$.
2. Relative to the newest input, that weight is $\beta^k$.
3. The time constant $\tau$ is defined as the number of steps for $\beta^k$ to fall to $1/e$.
4. Solving $\beta^\tau = 1/e$ gives the exact result $\tau = -1/\ln\beta$.
5. For $\beta$ close to $1$, $\ln\beta \approx -(1-\beta)$, so $\tau \approx 1/(1-\beta)$, accurate to about half a step.