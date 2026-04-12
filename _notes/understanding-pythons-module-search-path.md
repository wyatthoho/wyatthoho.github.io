---
layout: post
name: Understanding Python's Module Search Path
birth: 2026-04-12
---

## Introduction

When Python imports a module, it searches a list of directories stored in `sys.path`.
Knowing how `sys.path` is constructed is essential for understanding module resolution.
According to the [official documentation][sys_path_init],
the first entry in this list depends on how Python is invoked:

| Invocation method     | First entry in `sys.path`     |
| --------------------- | ----------------------------- |
| `py *.py` (Script)    | The script's parent directory |
| `py -c cmd` (Command) | Current working directory     |
| `py -m mod` (Module)  | Current working directory     |

The remaining entries — standard library directories, DLL paths,
and `site-packages` — are consistent across all invocation methods.
Let's verify each case with concrete examples.

---

## 1. Running a script file

I created the script `check_path.py` at `C:\temp` with the following content:

```Python
import sys
import pprint

pprint.pp(sys.path)
```

If we execute this script while standing in a different directory (`C:\Users\wyatt`):

```PowerShell
PS C:\Users\wyatt> py C:\temp\check_path.py
```

Output:

```
['C:\\temp',
 'C:\\Program Files\\Python312\\python312.zip',
 'C:\\Program Files\\Python312\\DLLs',
 'C:\\Program Files\\Python312\\Lib',
 'C:\\Program Files\\Python312',
 'C:\\Program Files\\Python312\\Lib\\site-packages']
```

**Key Takeaway**: 
When running a `.py` file, Python prepends the parent directory of the script to `sys.path`. 
This ensures the script can always import neighboring files, regardless of where you call it from.

---

## 2. Running a command with `-c`

The `-c` flag allows you to execute Python code directly from the shell. 
Let's inspect the path in this mode:

```PowerShell
PS C:\Users\wyatt> py -c "
>> import sys
>> import pprint
>> pprint.pp(sys.path)
>> "
```

Output:

```
['',
 'C:\\Program Files\\Python312\\python312.zip',
 'C:\\Program Files\\Python312\\DLLs',
 'C:\\Program Files\\Python312\\Lib',
 'C:\\Program Files\\Python312',
 'C:\\Program Files\\Python312\\Lib\\site-packages']
```

The same result is obtained from the interactive shell (`py` with no arguments).
The first entry is an empty string `''`, which Python treats as the current working directory.
We can verify this by checking the absolute path of an empty string:

```
PS C:\Users\wyatt> py -c "
>> from pathlib import Path
>> print(Path('').absolute())
>> "
```

Output:

```
C:\Users\wyatt
```

**Key Takeaway**: 
When running Python with `-c` or via the interactive REPL, 
the current working directory becomes the primary search location, 
represented by an empty string at the start of the list.

---

## 3. Running a module with `-m`

The `-m` flag runs a library module as a script. 
The built-in `site` module provides a convenient way to report `sys.path` and related environment info:

```PowerShell
PS C:\Users\wyatt> py -m site
```

Output:

```
sys.path = [
    'C:\\Users\\wyatt',
    'C:\\Program Files\\Python312\\python312.zip',
    'C:\\Program Files\\Python312\\DLLs',
    'C:\\Program Files\\Python312\\Lib',
    'C:\\Program Files\\Python312',
    'C:\\Program Files\\Python312\\Lib\\site-packages',
]
USER_BASE: 'C:\\Users\\wyatt\\AppData\\Roaming\\Python' (doesn't exist)
USER_SITE: 'C:\\Users\\wyatt\\AppData\\Roaming\\Python\\Python312\\site-packages' (doesn't exist)
ENABLE_USER_SITE: True
```

**Key Takeaway**: 
When running a module with `-m`, the full absolute path of the current working directory is inserted as the first location searched. 
This behaves similarly to command mode but uses an explicit path instead of an empty string.

---

## Summary

When executing a file, Python prioritizes the directory where that file is stored. Conversely, 
when executing a command or a module, Python prioritizes the directory where you are currently standing.

Recognizing this distinction is vital for debugging import errors that occur when moving scripts between different environments or execution contexts.

[sys_path_init]: https://docs.python.org/3.12/library/sys_path_init.html