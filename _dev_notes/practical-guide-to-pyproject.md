---
layout: post  
name: From Scripts to Packages - A Practical Guide to pyproject.toml
birth: 2026-08-27
---

## Introduction

This article shows, from the perspective of a newcomer to Python, what problems can arise when trying to reuse code, and how that leads to the concept of packaging and installing your own code. To fully understand this, both the traditional method (`setup.py`) and the modern method (`pyproject.toml`) will be shown. Besides, the reasons for using editable pip install and the `src/` layout instead of a flat layout are also explained.

---

## The Problem with Reusing Code Across Projects

The simplest way to structure a project is two files sitting side by side:

```
my_project/
├── foo.py
└── main.py
```

`foo.py` just prints something:

```python
# my_project/foo.py
print("foo")
```

`main.py` imports it by module name and calls it:

```python
# my_project/main.py
import foo
```

This works fine as long as you run it from inside `my_project/`, since `''` (the current directory) is in `sys.path`:

```
$ cd my_project
$ python main.py
foo
```

**The problem** shows up once you try to reuse `my_project` elsewhere. Say you copy the whole folder into a new project, `my_app`, and want to call into it as a subpackage:

```
my_app/
├── main.py
└── my_project/
    ├── foo.py
    └── main.py
```

`my_app/main.py` tries to import `my_project` as a subpackage:

```python
# my_app/main.py
import my_project.main
```

Now the import breaks:

```
$ cd my_app
$ python main.py  
Traceback (most recent call last):
  File "C:\temp\my_app\main.py", line 1, in <module>
    import my_project.main
  File "C:\temp\my_app\my_project\main.py", line 1, in <module>
    import foo
ModuleNotFoundError: No module named 'foo'
```

`foo.py` was only importable because `my_project/` itself used to be the current working directory. Once `my_project` becomes a subfolder of `my_app`, that assumption no longer holds.

One fix is to rewrite the import inside `my_project` to be absolute, instead of relying on the working directory:

```python
# my_app/my_project/main.py
import my_project.foo
```

That works for this toy example, but it doesn't scale: if `my_project` were a large codebase, rewriting every import statement by hand — and keeping them correct wherever the code gets reused — isn't realistic. What's needed instead is a way to make a project reusable without touching its internals. The following sections show how.

## How Python Import Works

When Python starts, it scans a fixed set of paths in `sys.path`. Only modules found under these paths can be `import`ed. The exact entries depend on the OS and how Python was installed:

```
Windows:
[...,
 'C:\\Program Files\\Python312\\python312.zip',
 'C:\\Program Files\\Python312\\DLLs',
 'C:\\Program Files\\Python312\\Lib',
 'C:\\Program Files\\Python312',
 'C:\\Program Files\\Python312\\Lib\\site-packages']

Linux/Mac:
[...,
 '/usr/lib/python312.zip',
 '/usr/lib/python3.12',
 '/usr/lib/python3.12/lib-dynload',
 '/usr/local/lib/python3.12/site-packages']
```

The first entry differs by invocation method — see [Understanding Python's Module Search Path](/dev_notes/understanding-pythons-module-search-path.html) for that — but that's not what we want to discuss in this article. What matters here is the last entry, `site-packages`, where third-party packages live.

This is the directory we'll target to make our own scripts reusable from anywhere in the same environment. Getting our code in there is what **installing** means. Copying and pasting every script in by hand isn't a good way to do that — conveniently, there's a tool called `pip` that happens to take charge of exactly this.

We all have experience using `pip` to install third-party packages, like numpy, requests, or Django. For our own package to be installable through `pip` too, it first needs to be **packaged**.

---

## Traditional Packaging Method

Take `setuptools` for example — the traditional way to package a project is a `setup.py` file at the project root:

```python
# setup.py
from setuptools import setup, find_packages

setup(
    name="my_project",
    version="0.1.0",
    packages=find_packages(),
)
```

`name`, `version`, and `packages` are the same core metadata pip needs — `packages` refers to the project's own Python packages, folders containing `__init__.py`, so `my_project/` needs one added alongside `foo.py` and `main.py` for `find_packages()` to pick it up:

```
my_project/
├── __init__.py
├── foo.py
└── main.py
```

Running `python setup.py install`, or `pip install .`, executes this script and skips straight to copying the package into site-packages:

```
site-packages/
├── my_project/
│   ├── __init__.py
│   ├── foo.py
│   └── main.py
└── my_project-0.1.0.egg-info/
    ├── PKG-INFO
    └── SOURCES.txt
```

`my_project-0.1.0.egg-info/` is metadata pip uses to track what got installed — it's what makes `pip show my_project` or `pip uninstall my_project` work.

---

## Modern Packaging Method — `pyproject.toml`

`pyproject.toml` is the modern Python project config file. `pip install` recognizes it by default.

```toml
[build-system]
requires = ["setuptools>=64"]
build-backend = "setuptools.build_meta"

[project]
name = "my_project"
version = "0.1.0"
```

`[build-system]` declares what's needed to build the project: `requires` lists the build-time dependencies, and `build-backend` says which tool actually does the building — `setuptools.build_meta` here. This is the declarative version of what `setup.py` had to be executed to figure out.

`[project]` holds the same core metadata as `setup.py`'s `name` and `version`.

---

## Two Ways to Install

### `pip install .` (regular install)

1. pip reads `pyproject.toml`
2. setuptools finds `my_project/`
3. **Copies the code into site-packages**
4. The running code lives in site-packages

> Downside: after editing source files, you must re-run `pip install .` to see the changes.

### `pip install -e .` (editable install, for development)

1. pip reads `pyproject.toml`
2. setuptools finds `my_project/`
3. **Creates a pointer in site-packages** (a `.pth` file or link) pointing to your local `my_project/` directory
4. The running code lives in your local project directory

> Upside: edits to source files take effect immediately on the next import — no reinstall needed.

Files created in site-packages:

```
site-packages/
├── my_project.egg-link              # older setuptools
└── __editable__.my_project...pth    # newer setuptools
```

---

## Enter the `src/` Layout

If the package sits directly in the project root:

```
my_project/
├── my_project/   ← package here
└── tests/
```

When running tests from the root, Python finds `my_project/` via `''` in `sys.path` and **bypasses site-packages entirely**. You think you're testing the installed version, but you're not.

**The `src/` layout fixes this:**

```
my_project/
├── src/
│   └── my_project/   ← package is here
└── tests/
```

Now `''` can't resolve `my_project/` directly, forcing you to go through the install step. Tests always run against the installed version.

Moving the package into `src/` means setuptools also needs to be told where to look for it:

```toml
[tool.setuptools.packages.find]
where = ["src"]   # ← tells setuptools to look inside src/
```

Without this, setuptools looks in the project root by default and won't find `src/my_project/`.

Note that `[tool.setuptools.packages.find]` doesn't require an `__init__.py` the way `find_packages()` in `setup.py` did — it discovers `src/my_project/` even as a bare folder of `.py` files.

---

## More about pyproject.toml

After packaging, you can expose a CLI command as an entry point via `[project.scripts]`:

```toml
[project.scripts]
my-project-cli = "my_project:main_cli"
```

After `pip install .`, an executable `my-project-cli` is created in the venv's `Scripts/` (Windows) or `bin/` (Linux/Mac). It's essentially a wrapper for:

```python
import sys
from my_project import main_cli
sys.exit(main_cli())
```

Another important field is `dependencies`:

```toml
[project]
dependencies = [
    "numpy>=1.26",
    "requests>=2.31,<3",
    "Django==5.0.*",
]
```

Each entry in `dependencies` is a [PEP 508](https://peps.python.org/pep-0508/) requirement string: a package name plus an optional version specifier (`>=`, `<`, `==`, `~=`, ...). When you run `pip install .`, pip reads this list and automatically installs any of these packages that aren't already present — you don't need a separate `requirements.txt` or manual `pip install` step.

There are a few other commonly used fields you can specify:

```toml
[project]
description = "A small toolkit for ..."
authors = [{name = "Your Name", email = "you@example.com"}]
requires-python = ">=3.12"

readme = "README.md"
license = "MIT"
```

Finally, a full `pyproject.toml` may look like:

```toml
[build-system]
requires = ["setuptools>=64"]
build-backend = "setuptools.build_meta"

[project]
name = "my_project"
version = "0.1.0"
description = "A small toolkit for ..."
authors = [{name = "Your Name", email = "you@example.com"}]
dependencies = [
    "numpy>=1.26",
    "requests>=2.31,<3",
    "Django==5.0.*",
]
requires-python = ">=3.12"
readme = "README.md"
license = "MIT"

[project.scripts]
my-project-cli = "my_project:main_cli"

[tool.setuptools.packages.find]
where = ["src"]
```
