---
layout: default
title: Home
---

# Welcome

{{ "Hello World!" | downcase }}

This page includes an example of inline $\LaTeX$ as well as a block of mathematical equations:

## Fourier Series Representation

Below is the Fourier series expansion of a periodic function $p(t)$:

$$
\begin{align*}
p(t) &= \sum_{j = -\infty}^{\infty} P_j e^{ij\omega_0t} \\\\
P_j &= \frac{1}{T_0} \int_0^{T_0} p(t) e^{-ij\omega_0t} \, dt
\end{align*}
$$
