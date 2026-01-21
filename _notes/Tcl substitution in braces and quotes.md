---
layout: post
---

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

# Tcl Expressions

## Expression Syntax

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

## The `expr` Command

The `expr` command concatenates its arguments (ignoring white spaces between
them) as a Tcl expression, evaluates the result, and returns the value.

## Critical Point: Double Substitution

There is an extremely important behavior I didn't understand until I
encountered the following error:

```tcl
expr {100 in "100 200 300"};  # Output: 1
expr 100 in "100 200 300";    # Error: missing operator at _@_
                              # in expression "100 in 100 _@_200 300"
```

I couldn't figure out why the error occurred until I carefully read the
[official documentation][expr].

> ... expressions are substituted twice: once by the Tcl parser and once by
> the expr command.

**How this works:**

- In the **first line**, the Tcl parser substitutes
  `{100 in "100 200 300"}` as-is, then the `expr` command receives the
  argument `100 in "100 200 300"` and evaluates it correctly.

- In the **second line**, the Tcl parser substitutes `"100 200 300"` first
  (removing the quotes), so the `expr` command receives multiple arguments:
  `100`, `in`, `100`, `200`, `300`. This leads to a "missing operator" error.

## Best Practice: Always Use Braces

The [official documentation][expr] provides clear guidance on writing
conventions:

> ... the command parser may already have performed one round of substitution
> before the expression processor was called. ... it is usually best to
> enclose expressions in braces to prevent the command parser from performing
> substitutions on the contents.

Additionally:

> Enclose expressions in braces for the best speed and the smallest storage
> requirements. This allows the Tcl bytecode compiler to generate the best
> code.

## Implications of Double Substitution

The double substitution procedure in Tcl creates interesting possibilities
during calculation. Below is an example from the [official documentation][expr]:

```tcl
set a 3
set b {$a + 2}

expr $b*4;             # Output: 11
expr {$b * 4};         # Error: can't use non-numeric string as operand of "*"
expr {[expr $b] * 4};  # Output: 20
```

This example demonstrates how substitution timing affects expression evaluation:

**Line 1: `expr $b*4` → Output: 11**

Without braces, double substitution occurs:
1. **First substitution** (Tcl parser): `$b` is replaced with `$a + 2`,
   resulting in `expr $a + 2*4`
2. **Second substitution** (`expr` command): `$a` is replaced with `3`,
   resulting in `3 + 2*4`
3. Following operator precedence (`*` before `+`), the result is `3 + 8 = 11`

**Line 2: `expr {$b * 4}` → Error**

With braces, only one substitution occurs:
1. **First substitution** (Tcl parser): The braces prevent substitution,
   so `expr` receives the literal string `$b * 4`
2. **Second substitution** (`expr` command): `$b` is replaced with the
   string `$a + 2` (not evaluated), resulting in `$a + 2 * 4`
3. The `expr` command tries to multiply the string `$a + 2` by `4`, which
   fails because it's not a numeric value

**Line 3: `expr {[expr $b] * 4}` → Output: 20**

This uses command substitution to evaluate `$b` first:
1. **First substitution** (Tcl parser): `[expr $b]` is evaluated
   - `$b` becomes `$a + 2`
   - `expr $a + 2` evaluates to `5`
   - The expression becomes `expr {5 * 4}`
2. **Second substitution** (`expr` command): Evaluates `5 * 4 = 20`

This example illustrates why understanding substitution order is crucial when
working with expressions stored in variables.

[expr]: https://www.tcl-lang.org/man/tcl8.6/TclCmd/expr.htm
