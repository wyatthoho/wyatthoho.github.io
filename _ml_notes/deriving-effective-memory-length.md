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

What matters for memory is how fast the weights shrink, so compare each to the newest one, $w_0$:

$$\begin{equation} \frac{w_k}{w_0} = \frac{(1-\beta)\,\beta^{k}}{(1-\beta)\,\beta^{0}} = \beta^{k} \end{equation}$$

The common factor $(1-\beta)$ cancels, and $\beta^0=1$. An input $k$ steps back therefore carries $\beta^k$ times the weight of the newest input. With $\beta=0.9$:

| Steps back $k$ | Relative weight $\beta^k$ |
|:---:|---:|
| 0 | 1 |
| 1 | 0.9 |
| 2 | 0.81 |
| 10 | ≈ 0.35 |

---

## Step 3: Define the Time Constant

To find how many steps back it takes for the relative weight to drop to a specific threshold $p$, set $\beta^k = p$ and take the logarithm of both sides:

$$\begin{equation} k = \frac{\ln(1/p)}{-\ln\beta} \end{equation}$$

Choosing $p=1/e$ makes the numerator $\ln(1/p)$ exactly $1$, which leaves the cleanest formula. This is the standard convention in physics, engineering, and signal processing, where the resulting number of steps is called the **time constant**, written $\tau$:

$$\begin{equation} \tau = \frac{-1}{\ln\beta} \end{equation}$$

Since $0<\beta<1$, $\ln\beta$ is negative and $\tau$ is positive.

---

## Step 4: Approximate

In practice, the logarithm is replaced by a simpler approximation for convenience:

$$\begin{equation} \tau = \frac{-1}{\ln\beta} \approx \frac{1}{1-\beta} \end{equation}$$

To see where this comes from, note that $\beta$ is typically close to $1$ ($0.9$, $0.99$, $0.999$). Write

$$\beta = 1-\varepsilon, \qquad \varepsilon = 1-\beta$$

and expand the logarithm as a Taylor series:

$$\begin{equation} \ln\beta = \ln(1-\varepsilon) = -\varepsilon - \frac{\varepsilon^2}{2} - \frac{\varepsilon^3}{3} - \cdots \end{equation}$$

Since $\varepsilon$ is small, the $\varepsilon^2$ and higher terms are negligible, leaving $\ln\beta \approx -\varepsilon = -(1-\beta)$. Substituting into $\tau = -1/\ln\beta$:

$$\begin{equation} \tau \approx \frac{1}{1-\beta} \end{equation}$$

To check the approximation, the table below compares the exact time constant $\tau_{\text{exact}} = -1/\ln\beta$ with the approximate one $\tau_{\text{approx}} = 1/(1-\beta)$, along with the relative weight $\beta^{\tau_{\text{approx}}}$ at the approximate one. By definition, the relative weight at $\tau_{\text{exact}}$ is exactly $1/e \approx 0.3679$:

| $\beta$ | $\tau_{\text{exact}}$ | $\tau_{\text{approx}}$ | $\beta^{\tau_{\text{approx}}}$ |
|---|---|---|---|
| 0.9 | 9.49 | 10 | 0.3487 |
| 0.99 | 99.50 | 100 | 0.3660 |
| 0.999 | 999.50 | 1000 | 0.3677 |

---

## Summary

1. Unrolling the recurrence shows that an input $k$ steps back has weight $(1-\beta)\beta^k$.
2. Relative to the newest input, that weight is $\beta^k$.
3. The number of steps for $\beta^k$ to fall to a threshold $p$ is $\ln(1/p)/(-\ln\beta)$. Choosing $p=1/e$ makes the numerator $\ln(1/p)$ exactly $1$ and defines the time constant, with exact value $\tau = -1/\ln\beta$.
4. For $\beta$ close to $1$, $\ln\beta \approx -(1-\beta)$, so $\tau \approx 1/(1-\beta)$, accurate to about half a step.