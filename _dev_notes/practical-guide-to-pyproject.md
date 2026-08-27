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
├── utils.py
└── main.py
```

`utils.py` defines a function that prints something:

```python
# utils.py
def say_hi():
    print("Hi")
```

`main.py` imports it by module name and calls it:

```python
# main.py
import utils

def main():
    utils.say_hi()

if __name__ == "__main__":
    main()
```

This works fine as long as you run it from inside `my_project/`:

```
$ cd my_project
$ python main.py
Hi
```

**The problem** shows up once you try to reuse `my_project` elsewhere. Say you copy the whole folder into a new project, `my_app`, and want to call into it as a subpackage:

```
my_app/
├── my_project/
|   ├── utils.py
|   └── main.py
└── main.py
```

The new `main.py` tries to import `my_project.main`:

```python
# main.py
import my_project.main
```

Now the import breaks:

```
$ cd my_app
$ python main.py  
Traceback (most recent call last):
  File "C:\my_app\main.py", line 1, in <module>
    import my_project.main
  File "C:\my_app\my_project\main.py", line 1, in <module>
    import utils
ModuleNotFoundError: No module named 'utils'
```

`utils.py` was only importable because `my_project/` itself used to be the current working directory. Once `my_project` becomes a subfolder of `my_app`, that assumption no longer holds.

One fix is to rewrite the import inside `my_project` to be **absolute**, instead of relying on the working directory:

```python
# my_project/main.py
import my_project.utils

def main():
    my_project.utils.say_hi()

if __name__ == "__main__":
    main()
```

That works for this toy example. However, if `my_project` were a large codebase, rewriting every import statement by hand isn't realistic. 

What's needed instead is a way to make a project reusable without touching its internals.

The solution is to write absolute imports from the start. For absolute imports to actually resolve in the dev environment, the file structure needs an extra level of nesting:

```
my_project/
├── my_project/        ← the actual package
|   ├── utils.py
|   └── main.py
└── main.py            ← entry point for local testing
```

This way, the dev environment and the app environment line up exactly — the same code runs whether it sits in `my_project/` during development or gets dropped into `my_app/` as a dependency.

But this creates a new problem: once an app depends on many packages at once, nesting each one directly under the app folder makes the project bloated:

```
my_app/
├── main.py
├── my_project_1/
│   └── ...
├── ...
└── my_project_100/
    └── ...
```

So in practice, all these packages get **installed** into `site-packages` and run from there instead. The following sections show how.

## What is `site-packages`?

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

The first entry (shown here as `...`) differs by invocation method — that's covered in [Understanding Python's Module Search Path](/dev_notes/understanding-pythons-module-search-path.html), not what we want to discuss in this article. What matters here is the last entry, `site-packages`, where third-party packages live.

This is the directory we'll target to make our own scripts reusable from anywhere in the same environment, which could be the global environment or a virtual environment.

Getting our code in there is what **installing** means. Copying and pasting every script in by hand isn't a good way to do that — conveniently, there's a tool called `pip` that happens to take charge of exactly this.

We all have experience using `pip` to install third-party packages, like numpy, requests, or Django. For our own package to be installable through `pip` too, it first needs to be **packaged**.

---

## Traditional Packaging Method

Take `setuptools` for example — the traditional way to package a project. Place a `setup.py` file at the project root, along with a `__init__.py` to mark where the package is. In the following example, the project root and the package share the same location:

```
my_project/
├── my_project\
│   ├── __init__.py
│   ├── utils.py
│   └── main.py
├── main.py
└── setup.py
```

where `setup.py` is:

```python
# setup.py
from setuptools import find_packages, setup

setup(
    name="my_project",
    version="0.1.0",
    packages=find_packages(),
)
```

`name`, `version`, and `packages` are the same core metadata pip needs. `packages` refers to the project's own Python packages. `find_packages()` detects folders containing `__init__.py`.

Running `python setup.py install`, or `pip install .`, executes this script and skips straight to copying the package into site-packages:

```
site-packages/
└── my_project/
    ├── __init__.py
    ├── utils.py
    └── main.py
```

But now a new problem surfaces. Run:

```
$ cd my_project
python .\main.py
```

Because the `my_project/` package folder sits right next to the `main.py` you're running, there's no way to tell whether the import resolved to your local source files or to the copy installed in site-packages.

---

## Enter the `src/` Layout

We need to add one more layer, `src/`:

```
my_project/
├── src\
|   └── my_project\
|       ├── __init__.py
|       ├── utils.py
|       └── main.py
├── main.py
└── setup.py
```

`setup.py` needs to change too:

```python
# setup.py
from setuptools import find_packages, setup

setup(
    name="my_project",
    version="0.1.0",
    packages=find_packages("src/"),
    package_dir={"": "src"},
)
```

Run:

```
python main.py
```

Since `my_project/` no longer sits directly under the project root, `main.py`'s `import my_project.utils` can't resolve it locally anymore — this confirms you're running the version installed in site-packages.

---

## Modern Packaging Method — `pyproject.toml`

`pyproject.toml` is the modern Python project config file. `pip install` recognizes it by default. You need to place it at the root — no `__init__.py` needed.

```
my_project/
├── src\
|   └── my_project\
|       ├── utils.py
|       └── main.py
└── pyproject.toml
```

where `pyproject.toml` is:

```toml
[build-system]
requires = ["setuptools>=64"]
build-backend = "setuptools.build_meta"

[project]
name = "my_project"
version = "0.1.0"

[tool.setuptools.packages.find]
where = ["src"]
```

`[build-system]` declares what's needed to build the project: `requires` lists the build-time dependencies, and `build-backend` says which tool actually does the building — `setuptools.build_meta` here. This is the declarative version of what `setup.py` had to be executed to figure out.

`[project]` holds the same core metadata as `setup.py`'s `name` and `version`.

`[tool.setuptools.packages.find]` tells setuptools to look for packages inside `src/` instead of the project root — without it, setuptools looks in the root by default and won't find `src/my_project/`.

To install the package, you can run `pip install .`:

1. pip reads `pyproject.toml`
2. setuptools finds `my_project/`
3. **Copies the code into site-packages**
4. The running code lives in site-packages

However, there is a downside. After editing source files, you must re-run `pip install .` to see the changes.

To prevent the inconvenience, install in editable mode for development by running `pip install -e .`:

1. pip reads `pyproject.toml`
2. setuptools finds `my_project/`
3. **Creates a pointer in site-packages** (a `.pth` file or link) pointing to your local `my_project/` directory
4. The running code lives in your local project directory

The upside is that edits to source files take effect immediately on the next import — no reinstall needed.

---

## More about pyproject.toml

After packaging, you can expose a CLI command as an entry point via `[project.scripts]`:

```toml
[project.scripts]
my-project-cli = "my_project.main:main"
```

After `pip install .`, an executable `my-project-cli` is created in the venv's `Scripts/` (Windows) or `bin/` (Linux/Mac). It's essentially a wrapper for:

```python
import sys
from my_project.main import main
sys.exit(main())
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

---

## Conclusion

- **Absolute imports** make a module's location-dependent import (`import utils`) work no matter where the project is placed, which is what lets it be reused elsewhere at all.

- **`site-packages`** is already on Python's `sys.path`, so installing packages there lets any project in the same environment import them without nesting dependencies directly inside the app folder.

- **The `src/` layout** keeps the package out of the default import path, so a successful `import my_project` proves you're running the installed copy in `site-packages`, not a local folder that happens to shadow it.

- **`pip install -e .`** points `site-packages` back at your local `src/` directory instead of copying files, so source edits take effect immediately without reinstalling.

- **`pyproject.toml`** is the modern, declarative alternative to `setup.py`: one standardized file for build backend, metadata, dependencies, and entry points, read directly by `pip` without executing a script.
