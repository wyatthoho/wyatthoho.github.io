---
layout: post
name: A Roadmap to Gradient Descent Optimizers
birth: 2026-09-19
---
## The Basic Form of Gradient Descent
 
Here, $h^{t,(i)}$ is the model's predicted value for the $i$-th example at iteration $t$, and $J$ is the cost function that measures how far those predictions are from the true values $y^{(i)}$. The index $i$ runs over the $m$ training examples, while $j$ runs over the $n+1$ parameters ($\theta_0, \theta_1, \dots, \theta_n$).
 
By convention, $x_0^{(i)}=1$ for every example, so that $\theta_0$ is the intercept.
 
$$\begin{equation} h^{t,(i)} = \theta_0^{t} x_0^{(i)} + \theta_1^{t} x_1^{(i)} + \dots + \theta_n^{t} x_n^{(i)} \end{equation}$$
 
$$\begin{equation} J\left(\theta^{t}\right) = \frac{1}{2m}\sum_{i=1}^{m}{\left(h^{t,(i)}-y^{(i)}\right)^2} \end{equation}$$
 
This update rule computes $\theta_j^{t+1}$, the new value of the $j$-th parameter after this step, using all $m$ examples:
 
$$\begin{equation} \theta_j^{t+1} = \theta_j^{t} - \alpha \frac{1}{m}\sum_{i=1}^{m}{\left(h^{t,(i)}-y^{(i)}\right)x_j^{(i)}} \end{equation}$$
 
The constant $\alpha$ is the learning rate, a hyperparameter that controls how large each update step is; it is typically set somewhere between $0.001$ and $0.01$. Repeat this update until convergence to obtain the trained parameters $\theta_0, \theta_1, \dots, \theta_n$ that define the fitted hypothesis.
 
---
 
## Stochastic Gradient Descent, SGD
 
SGD updates the parameters using **one** randomly chosen training example at a time, rather than the full dataset — this randomness is where the name "stochastic" comes from, and it avoids the cost of computing the gradient over all $m$ examples on every step as Batch GD does.
 
$$\begin{equation} J^{(i)}\left(\theta^{t}\right) = \frac{1}{2}\left(h^{t,(i)}-y^{(i)}\right)^2 \end{equation}$$
 
This update rule uses only the $i$-th example, and $t$ increments after every single example rather than after every epoch:
 
$$\begin{equation} \theta_j^{t+1} = \theta_j^{t} - \alpha\left(h^{t,(i)}-y^{(i)}\right)x_j^{(i)} \end{equation}$$
 
Although each step only looks at one example, the gradient computed from it is still an unbiased estimator of the true gradient over the full dataset — so while individual steps are noisy, they average out to roughly the right direction over many updates.
 
---
 
## Mini-batch Gradient Descent
 
Rather than a single example as in SGD, Mini-batch GD updates the parameters using a small batch at a time, trading some of SGD's speed for a less noisy gradient estimate. Let $b$ be the batch size ($1<b<m$), and let $B_t$ be the set of the $b$ example indices sampled at iteration $t$:
 
$$\begin{equation} J_{B_t}\left(\theta^{t}\right) = \frac{1}{2b}\sum_{i\in B_t}{\left(h^{t,(i)}-y^{(i)}\right)^2} \end{equation}$$
 
$$\begin{equation} \theta_j^{t+1} = \theta_j^{t} - \alpha \frac{1}{b}\sum_{i\in B_t}{\left(h^{t,(i)}-y^{(i)}\right)x_j^{(i)}} \end{equation}$$
 
Setting $b=m$ recovers Batch GD; setting $b=1$ recovers SGD.
 
> The sections above (Batch GD, SGD, Mini-batch GD) are all about one question: **how many examples should be used** to compute the gradient at each step. The sections below (Momentum, AdaGrad, RMSProp, Adam) answer a different question: once a gradient has been computed, however many examples it came from, how should it be used to decide the direction and size of the update. These two questions are independent of each other, so the gradient term $\dfrac{\partial}{\partial\theta_j^{t}}J\left(\theta^{t}\right)$ used below could in principle come from Batch GD, SGD, or Mini-batch GD — in practice, it is almost always paired with Mini-batch GD.
 
---
 
## Smoothing the Update Direction - Momentum
 
The term $s_j^{t}$ denotes the first moment (an exponential moving average of the gradient); it is written as $s$ rather than the more common $m$ to avoid colliding with $m$, the number of training examples.
 
$$\begin{equation} s_j^{t+1} = \beta_1\, s_j^{t} + (1-\beta_1)\,\frac{\partial}{\partial\theta_j^{t}}J\left(\theta^{t}\right) \end{equation}$$
 
$$\begin{equation} s_j^{0}=0 \end{equation}$$
 
$$\begin{equation} \theta_j^{t+1} = \theta_j^{t} - \alpha\, s_j^{t+1} \end{equation}$$
 
The raw gradient at any single step can point in a noisy or oscillating direction, especially when different mini-batches or examples disagree with each other.
 
Instead of moving purely in the direction of the current negative gradient, the update rule above combines the current gradient, weighted by $(1-\beta_1)$, with the previous $s_j^{t}$, weighted by $\beta_1$.
 
Since $s_j^{t}$ was built the same way from $s_j^{t-1}$, each past gradient keeps getting carried forward, shrinking by another factor of $\beta_1$ with every step — so $s_j^{t+1}$ is really a weighted average over recent gradients, not just the current one.
 
This is the same idea as a ball rolling downhill carrying momentum from its previous motion, rather than reacting only to the slope right beneath it — hence the method's name.
 
In practice, $\beta_1$ is typically $0.9$, and because each step's contribution shrinks by a factor of $\beta_1$, the effective memory length works out to about $1/(1-\beta_1)=10$ steps — so the update direction reflects roughly the last 10 steps' worth of gradients, rather than being dominated by the noise of any single one.
 
Momentum still applies the same learning rate $\alpha$ to every parameter $\theta_j$, though — it smooths out the direction of the update, but has no way to give different parameters different step sizes.
 
---
 
## Giving Each Parameter Its Own Learning Rate - AdaGrad
 
AdaGrad gives each parameter, $\theta_j^t$, its own effective learning rate, based on how large its past gradients have been — parameters with a history of large gradients get smaller steps, and parameters with a history of small gradients get larger ones.
 
The term $r_j^{t}$ denotes the second moment — the accumulated squared gradient for parameter $j$.
 
$$\begin{equation} r_j^{t+1} = r_j^{t} + \left(\frac{\partial}{\partial\theta_j^{t}}J\left(\theta^{t}\right)\right)^2 \end{equation}$$
 
$$\begin{equation} r_j^{0}=0 \end{equation}$$
 
$$\begin{equation} \theta_j^{t+1} = \theta_j^{t} - \frac{\alpha}{\sqrt{r_j^{t+1}+\epsilon}}\,\frac{\partial}{\partial\theta_j^{t}}J\left(\theta^{t}\right) \end{equation}$$
 
Notably, $r_j^{t}$ accumulates **without decay, indefinitely**, so as $t$ grows the effective learning rate $\dfrac{\alpha}{\sqrt{r_j^{t+1}+\epsilon}}$ monotonically shrinks toward 0, and learning effectively stalls late in training.
 
---
 
## Stopping the Learning Rate From Shrinking to Zero - RMSProp
 
RMSProp builds on AdaGrad's idea, but only accumulates the squared gradient over recent steps rather than over all of history:
 
$$\begin{equation} r_j^{t+1} = \beta_2\, r_j^{t} + (1-\beta_2)\left(\frac{\partial}{\partial\theta_j^{t}}J\left(\theta^{t}\right)\right)^2 \end{equation}$$
 
$$\begin{equation} r_j^{0}=0 \end{equation}$$
 
$$\begin{equation} \theta_j^{t+1} = \theta_j^{t} - \frac{\alpha}{\sqrt{r_j^{t+1}+\epsilon}}\,\frac{\partial}{\partial\theta_j^{t}}J\left(\theta^{t}\right) \end{equation}$$
 
In practice, $\beta_2$ is typically $0.9$ or $0.999$, so that $r_j^{t}$ only reflects the gradient magnitude over a recent window, and the effective learning rate no longer decays monotonically to 0. The small constant $\epsilon$ in the denominator, typically $10^{-8}$, exists purely to prevent division by zero when $r_j^{t+1}$ is very close to 0.
 
---
 
## Combining Momentum and RMSProp - Adam
 
Adam combines the two ideas from the previous two sections: it keeps Momentum's smoothed direction and RMSProp's per-parameter learning rate, computing both a first moment and a second moment at every step.
 
The first moment is computed the same way as in Momentum:
 
$$\begin{equation} s_j^{t+1} = \beta_1\, s_j^{t} + (1-\beta_1)\,\frac{\partial}{\partial\theta_j^{t}}J\left(\theta^{t}\right) \end{equation}$$
 
The second moment is computed the same way as in RMSProp:
 
$$\begin{equation} r_j^{t+1} = \beta_2\, r_j^{t} + (1-\beta_2)\left(\frac{\partial}{\partial\theta_j^{t}}J\left(\theta^{t}\right)\right)^2 \end{equation}$$
 
Since $s_j^{0}=r_j^{0}=0$, the early estimates are biased toward 0, so a bias correction is applied:
 
$$\begin{equation} \hat{s}_j^{t+1} = \frac{s_j^{t+1}}{1-\beta_1^{\,t+1}}, \qquad \hat{r}_j^{t+1} = \frac{r_j^{t+1}}{1-\beta_2^{\,t+1}} \end{equation}$$
 
Note that $\beta_1^{\,t+1}$ and $\beta_2^{\,t+1}$ here mean $\beta_1$ and $\beta_2$ raised to the power $t+1$ — this is different from the superscript $t$ used elsewhere in this note to mark the iteration number.
 
With $\beta_1=0.9$ and $\beta_2=0.999$ typically, both terms start close to 1 when $t$ is small (making the correction large) and decay toward 0 as $t$ grows (making the correction negligible), which is exactly the behavior needed to offset the bias from $s_j^{0}=r_j^{0}=0$.
 
Putting the corrected first and second moments together, the final update rule is:
 
$$\begin{equation} \theta_j^{t+1} = \theta_j^{t} - \alpha\, \frac{\hat{s}_j^{t+1}}{\sqrt{\hat{r}_j^{t+1}}+\epsilon} \end{equation}$$
 
The numerator $\hat s_j^{t+1}$ comes from Momentum (deciding direction); the denominator $\sqrt{\hat r_j^{t+1}}$ comes from RMSProp (deciding step size); dividing one by the other gives Adam.
 
---
 
## Summary Table
 
| Section | Method | Problem addressed |
|:---|:---|:---|
| The Basic Form | GD | — |
| Stochastic Gradient Descent | SGD | GD is too slow on large datasets |
| Mini-batch Gradient Descent | Mini-batch GD | SGD is too unstable |
| Smoothing the Update Direction | Momentum | Oscillating update direction |
| Giving Each Parameter Its Own Learning Rate | AdaGrad | Different parameters need different learning rates |
| Stopping the Learning Rate From Shrinking to Zero | RMSProp | AdaGrad's learning rate decays to 0 |
| Combining Momentum and RMSProp | Adam | Needs both smooth direction and adaptive step size |
 