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

## Nutrition Information of McDonald's

The nutrition information of the McDonald's meal is shown below:

| Item         | Calories(Cal) | Total Fat(g) | Total Carbs(g) | Protein(g) |
| :----------- | ------------: | -----------: | -------------: | ---------: |
| Big Mac      |           550 |           30 |             45 |         25 |
| Cheeseburger |           300 |           13 |             32 |         15 |
| McChicken    |           400 |           21 |             39 |         14 |
| Filet-O-Fish |           380 |           18 |             39 |         16 |
| McNuggets    |           310 |           18 |             21 |         14 |

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
