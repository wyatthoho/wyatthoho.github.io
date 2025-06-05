---
layout: default
title: Home
---

# Welcome

{{ "Hello World!" | downcase }}

This page includes examples of $LaTeX$, tables, and 
script language syntax highlighting.

## Fourier Series Representation

Below is the Fourier series expansion of a periodic function $p(t)$:

$$
\begin{align*}
p(t) &= \sum_{j = -\infty}^{\infty} P_j e^{ij\omega_0t} \\\\
P_j &= \frac{1}{T_0} \int_0^{T_0} p(t) e^{-ij\omega_0t} \, dt
\end{align*}
$$

## Constitutive Law of Rubber

The parameters of the rubber material constitutive model 
are shown:

| Index |   $\mu_i$ | $\alpha_i$ |      $D_i$ |
| :---- | --------: | ---------: | ---------: |
| 1     |    0.3723 |      1.545 |  9.898E-03 |
| 2     | 6.562E-04 |      5.846 | -1.285E-02 |
| 3     | 1.704E-02 |     -1.835 |  4.779E-04 |


## Operators in Tcl

Some operators work on numbers in general (`+`, `-`, 
`*`, `/`, `**`, `%`):

```Tcl
% set height 174
174
% set weight 70
70
% expr $weight / ($height * 0.01)**2
23.120623596247853
% expr 5 % 3;  # only for integers
2
```

## CSV Module in Python

Read a csv file using [Python](https://www.python.org/).

```Python
import csv

with open('input.csv', 'r') as f:
    data = csv.reader(f, delimiter=',', skipinitialspace=True)
    for rowdata in data:
        print(rowdata)
    
    print(data.line_num)
```
