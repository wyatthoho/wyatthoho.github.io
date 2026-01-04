---
layout: default
title: Home
---

# Welcome

Hello, World! I am wyatthoho. I built this website to document 
my learning notes on various technical topics, including *Tcl/Tk*, 
*Python*, *LaTeX*, *Markdown*, *Git*, and *Jekyll*. You will also find 
content related to mathematics and mechanics theories here.

To give you a quick preview, I have included some simple examples 
of these tools below.

---

## Exploring Tcl/Tk

Tcl is a scripting language that might be less known to the general 
public, but it is highly prevalent in specialized fields like electronic 
design automation (EDA), computer-aided design (CAD), and computer-aided 
engineering (CAE) software.

Here is an example of numerical operations in Tcl:

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

Tk is a graphical user interface (GUI) toolkit. Originally developed 
as an extension for Tcl, it has been integrated into many other 
languages like Python, Ruby, and Perl.

Below is a simple dialog created using Tk:

![alt text](assets/images/tcl-hello-world.png)

The corresponding Tcl code is:

```Tcl
package require Tk

wm title . "hello world"
grid [ttk::frame .f] -column 0 -row 0 -sticky nwes
grid [ttk::label .f.l -textvariable msg] -column 0 -row 0 -sticky nwes
set ::msg "Hello, World!"
vwait forever
```

---

## Python at a Glance

Python is celebrated for its readability and versatility. It 
is so popular that it arguably needs no introduction.

My journey with Python began with automating CAE workflows in 
Abaqus. Since then, I have expanded its use to data science, 
machine learning, and countless other applications. 

To give a quick preview, here is a simple script to read a CSV 
file using Python:

```Python
import csv

with open('input.csv', 'r') as f:
    data = csv.reader(f, delimiter=',', skipinitialspace=True)
    for rowdata in data:
        print(rowdata)
    
    print(data.line_num)
```

---

## Academic Typesetting with LaTeX

$\LaTeX$ is a high-quality typesetting system widely used in 
academia, thanks to its superior handling of complex mathematical 
notation. 

Below is the Fourier series expansion of a periodic function $p(t)$:

$$
\begin{align*}
p(t) &= \sum_{j = -\infty}^{\infty} P_j e^{ij\omega_0t} \\\\
P_j &= \frac{1}{T_0} \int_0^{T_0} p(t) e^{-ij\omega_0t} \, dt
\end{align*}
$$

The formula above is rendered using the following $\LaTeX$ code:

```LateX
\begin{align*}
    p(t) &= \sum_{j = -\infty}^{\infty} P_j e^{ij\omega_0t} \\
    P_j &= \frac{1}{T_0} \int_0^{T_0} p(t) e^{-ij\omega_0t} \, dt
\end{align*}
```

---

## Documenting with Markdown

Markdown is a lightweight markup language used for creating 
formatted text with a plain-text editor. It is widely adopted 
for its simplicity. 

Below is a quick look at the nutritional information for some 
McDonald's items using Markdown's table syntax:

| Item         | Calories(Cal) | Total Fat(g) | Total Carbs(g) | Protein(g) |
| :----------- | ------------: | -----------: | -------------: | ---------: |
| Big Mac      |           550 |           30 |             45 |         25 |
| Cheeseburger |           300 |           13 |             32 |         15 |
| McChicken    |           400 |           21 |             39 |         14 |
| Filet-O-Fish |           380 |           18 |             39 |         16 |
| McNuggets    |           310 |           18 |             21 |         14 |

The Markdown source for this table is:

```Markdown
| Item         | Calories(Cal) | Total Fat(g) | Total Carbs(g) | Protein(g) |
| :----------- | ------------: | -----------: | -------------: | ---------: |
| Big Mac      |           550 |           30 |             45 |         25 |
| Cheeseburger |           300 |           13 |             32 |         15 |
| McChicken    |           400 |           21 |             39 |         14 |
| Filet-O-Fish |           380 |           18 |             39 |         16 |
| McNuggets    |           310 |           18 |             21 |         14 |
```
