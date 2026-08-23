---
layout: post
name: Tcl Gotchas - Commands, Substitutions, Expressions
birth: 2026-01-22
---

In Tcl, the distinction between `"..."`, `[...]`, and `{...}` initially felt 
like a guessing game to me. For a long time, I found myself memorizing syntax 
case-by-case and struggling with unexpected `expr` errors without truly 
understanding the underlying logic. This article documents my journey to 
demystify the Tcl interpreter by breaking down substitution rules and the 
*double evaluation* trap, transforming that initial confusion into a clearer 
mental model.

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

*Note: Inside braces, backslashes are generally ignored, with one major 
exception: a backslash followed by a newline is replaced by a single space to 
allow for multi-line formatting without breaking the string.

There is another special syntax with braces called *argument expansion*. If a 
word begins with `{*}`, the rest of the word is substituted as usual and then 
parsed as a list. Each element of that list is then added to a command as a 
separate argument. Example:

```tcl
set profile {name Alice age 30}
dict create {*}$profile
```

The specific details of substitutions are described below.

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

Applied to our case, the Tcl parser performs substitution on `"100 200 300"` 
first (stripping the quotes). Consequently, the `expr` command receives five 
distinct arguments: `100`, `in`, `100`, `200`, and `300`. Because `expr` 
expects a single expression or a valid operator between these values, this 
results in a "missing operator" error.

So, how do we write the expression correctly?

```tcl
expr {100 in "100 200 300"}
# Output: 1
```

In this instance, the Tcl parser treats the braced string 
`{100 in "100 200 300"}` as a single literal word, as braces prevent all 
internal substitutions. Consequently, the `expr` command receives the entire 
string as one argument. It then performs its own internal evaluation, correctly 
identifying the three distinct components-the value, the operator, and the 
list-and processes the expression without error.

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

Without braces, a double substitution occurs. First, the Tcl parser replaces 
`$b` with the literal string `$a + 2`, passing the expression `$a + 2*4` to the 
`expr` command. Subsequently, `expr` performs its own internal substitution, 
replacing `$a` with `3`. This results in the calculation `3 + 2*4`, yielding a 
final result of 11.

```tcl
set a 3
set b {$a + 2}
expr {$b * 4}
# Error!!
```

With braces, only one level of substitution occurs. The Tcl parser passes the 
literal string `$b * 4` directly to the `expr` command. During evaluation, 
`expr` performs a single substitution, replacing `$b` with its value: the 
string `$a + 2`. This results in the expression `$a + 2 * 4`. However, because 
`expr` has already completed its substitution phase, it does not re-scan the 
result for further variables. Consequently, it attempts to multiply the literal 
string `$a + 2` by `4`, which triggers a numeric syntax error.

```tcl
set a 3
set b {$a + 2}
expr {[expr $b] * 4}
# Output: 20
```

When using braces, the Tcl parser passes the literal string `[expr $b] * 4` 
directly to the `expr` command. During evaluation, the `expr` engine identifies 
the command substitution `[...]`, which pauses the current calculation to 
trigger a nested command execution. According to [tcl-lang][tcl] on command 
substitution:

> If command substitution occurs then the nested command is processed entirely 
> by the recursive call to the Tcl interpreter; no substitutions are performed 
> before making the recursive call and no additional substitutions are 
> performed on the result of the nested script.

Thus, `expr $b` is executed as an independent sub-command, returning the 
value 5. The outer `expr` then substitutes this result into the original string, 
simplifying the expression to `5 * 4`. This produces the expected result of 20.

Expressions within `if` and `for` statements behave the same way as `expr` 
arguments; however, the specifics of that evaluation are outside the scope of 
this article.

## Summary

After systematically organizing these syntax details, the myths that once 
confused me are now clear. Although the rules are intricate, I believe there 
are a few basic principles to keep in mind. 

- `"..."` is a word that allows substitution.
- `{...}` is a word that prevents substitution (except for backslash-newline).
- `[...]` invokes the Tcl interpreter *recursively* to process characters as a Tcl script.
- Expressions are substituted once by the Tcl parser and once by the `expr` command.

These principles form the basis of the more complex mechanics in Tcl.

[expr]: https://www.tcl-lang.org/man/tcl8.6/TclCmd/expr.htm
[tcl]: https://www.tcl-lang.org/man/tcl8.6/TclCmd/Tcl.htm
