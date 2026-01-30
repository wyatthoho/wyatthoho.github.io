---
layout: post
name: Tcl Gotchas - Commands, Substitutions, Expressions
---

Learn when to use `"..."`, `[...]`, and `{...}` in Tcl—and why it matters. 
We'll break down how the interpreter parses commands, demystify substitution 
rules, and explain the double evaluation in `expr`. By understanding these 
syntax gotchas, you'll avoid common pitfalls and write more reliable Tcl code.

## Script, Commands, and Words

A Tcl *script* is a string containing one or more *commands*.
Semicolons and newlines are command separators. Below is a 
Tcl script containing multiple commands:

```tcl
# Using semicolons to separate commands
set x 1; set y 2; set z 3

# Using newlines to separate commands
set a 10
set b 20
set c 30
```

A *command* is composed of *words*. The first word is used to 
locate a command procedure to carry out the command, then 
all of the words of the command are passed to the command 
procedure. If a hash character `#` appears as the first character 
of a command, then this command is treated as a *comment*.

```tcl
set msg "Hello, World!"; # Set the value of a variable
list Apple Banana Cherry; # Create a list with any number of arguments
```

A *word* is defined as:
1. continuous characters separated by whitespace
2. string enclosed by double quotes `"..."`
3. string enclosed by braces `{...}`

When a word is enclosed in double quotes or braces, special characters such as 
semicolons, newlines, and whitespace are treated as literal characters and 
included in the word.

```tcl
puts Hello, World!;  # can not find channel named "Hello,"
puts "Hello, World!";  # Hello, World!
puts {Hello, World!};  # Hello, World!
```

The primary difference between these two grouping mechanisms is how 
*substitutions* are processed:

| Substitution Type | Double Quotes `"..."` | Braces `{...}` |
| ----------------- | :-------------------- | :------------- |
| Command []        | Supported             | Ignored        |
| Variable $        | Supported             | Ignored        |
| Backslash \       | Supported             | Ignored*       |

*Note: Braces ignore all substitutions except for backslash-newline sequences.

The specific details of these substitutions are described below.

---

## Substitution

Before carrying out the command, the Tcl interpreter breaks the 
command into words and performs substitutions.

### Command substitution

If a word contains an section enclosed by open/close bracket `[...]` 
then Tcl performs command substitution. The Tcl interpreter process 
the characters as a Tcl *script*. The script may contain any number 
of commands. The result of the script is substituted into the word 
in place of the brackets and all of the characters between them. 

```tcl
set fruit [lindex "apple banana cherry" 0];  # fruit = "apple"
```

### Variable substitution

If a word contains a dollar sign `$`, then Tcl performs variable substitution. 
That is, the dollar sign and the following characters are replaced in the word 
by the value of a variable.

```tcl
set name wyatt 
puts $name
# Output: wyatt
```

However, if using the same approach to try to get the output `wyatthoho`, an 
error will occur:

```tcl
puts $namehoho
# can't read "namehoho": no such variable
```

There is another form of variable substitution to overcome this difficulty.
That is using `${name}`.

```tcl
puts ${name}hoho
# Output: wyatthoho
```

### Backslash Substitution

If a backslash `\` appears within a word, then backslash 
substitution occurs. The backslash is used to escape special 
characters, such as:

- `\\`: Literal backslash `\`
- `\$`: Literal dollar sign `$`
- `\[`: Literal open bracket `[`
- `\]`: Literal close bracket `]`
- `\"`: Literal double quote `"`

The backslash is also used to insert whitespace characters, 
such as:

- `\n`: Newline character
- `\t`: Tab character
- `\<newline>whiteSpace`: A single space character

---

## Tcl Expressions

### Expression Syntax

A Tcl expression consists of a combination of **operands**, **operators**,
parentheses, and commas. White space may be used between operands, operators,
parentheses, and commas.

Operands may be specified in any of the following ways:

- Numeric values
- Boolean values
- Variable substitution: `$...`
- Command substitution: `[...]`
- Strings enclosed in double-quotes: `"..."`
- Strings enclosed in braces: `{...}`

Common operators include `+`, `-`, `*`, `/`, `**`, `==`, `!=`, `eq`, `ne`,
`in`, `ni`, and others. The specific meaning of these operators is not the
focus here, so no further explanation is provided.

### The expr Command

The `expr` command concatenates its arguments (ignoring white spaces between
them) as a Tcl expression, evaluates the result, and returns the value. Here's
an example:

```tcl
expr 1 + 1 + 1
# Output: 3
```

However, let's try this:

```tcl
expr 100 in "100 200 300"
# Error!!
```

Why did it fail? This example illustrates an extremely important behavior of `expr`, 
according to [tcl-lang][expr]:

> ... expressions are substituted twice: once by the Tcl parser and once by
> the expr command.

Applied to our case, the Tcl parser substitutes `"100 200 300"` first
(removing the quotes), so the `expr` command receives multiple arguments:
`100`, `in`, `100`, `200`, `300`. This leads to a "missing operator" error.

So, how do we write the expression correctly?

```tcl
expr {100 in "100 200 300"}
# Output: 1
```

In this line, the Tcl parser leaves a **single word** `{100 in "100 200 300"}` 
as-is, because as mentioned earlier, braces prevent all substitutions. Then the 
`expr` command receives the argument `100 in "100 200 300"` representing 3 distinct 
words and evaluates it correctly.

The [tcl-lang][expr] provides clear guidance on writing conventions:

> ... the command parser may already have performed one round of substitution
> before the expression processor was called. ... it is usually best to
> enclose expressions in braces to prevent the command parser from performing
> substitutions on the contents.

Additionally:

> Enclose expressions in braces for the best speed and the smallest storage
> requirements. This allows the Tcl bytecode compiler to generate the best
> code.

Unfortunately, simply placing the calculation you need within braces will not
solve all problems. Substitution timing must be considered carefully.

```tcl
set a 3
set b {$a + 2}
expr $b*4
# Output: 11
```

Without braces after expr, double substitution occurs. First, the Tcl parser
replaces `$b` with `$a + 2`, resulting in `expr $a + 2*4`. Then, within `expr`,
`$a` is replaced with `3`, giving `3 + 2*4`. Following operator precedence
(`*` before `+`), the result is `3 + 8 = 11`.

```tcl
set a 3
set b {$a + 2}
expr {$b * 4}
# Error!!
```

With braces, only one substitution occurs, so expr receives the literal
string `$b * 4`. During evaluation, `$b` is replaced with the string `$a + 2`  
(which is not further evaluated), resulting in `$a + 2 * 4`. Since substitution
inside expr has already been performed, `$a` is not replaced again. The
command then attempts to multiply the string `$a + 2` by `4`, which fails
because it is not a numeric value.

```tcl
set a 3
set b {$a + 2}
expr {[expr $b] * 4}
# Output: 20
```

With braces, only one substitution occurs. The Tcl parser passes the expression
`[expr $b] * 4` to the outer expr command. During evaluation, `expr` detects
the command substitution `[...]`, which triggers a nested command execution.
According to [tcl-lang][tcl] on command substitution:

> If command substitution occurs then the nested command is processed entirely 
> by the recursive call to the Tcl interpreter; no substitutions are performed 
> before making the recursive call and no additional substitutions are 
> performed on the result of the nested script.

Thus, `expr $b` is executed independently and returns `5`. The outer expression
then becomes `5 * 4`, producing the expected result `20`.

This example illustrates why understanding substitution order is crucial when
working with expressions stored in variables.

[expr]: https://www.tcl-lang.org/man/tcl8.6/TclCmd/expr.htm
[tcl]: https://www.tcl-lang.org/man/tcl8.6/TclCmd/Tcl.htm
