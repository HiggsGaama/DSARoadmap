# Day 29 — The Import System & Packaging: Finders, Loaders, Cycles, and Wheels

[← Week 5](README.md) · **Track:** Python Systems · **Time box:** 55 min (concept 20 · gotcha 10 · code 20 · review 5)

> **Why this matters for platform work:** Cold-start latency in serverless and autoscaled pods is often dominated by imports. "Works on my machine" packaging bugs (missing data files, an editable install masking a broken wheel, unpinned transitive deps) are a top cause of failed deploys. And a circular import is the classic refactor that passes code review and then crashes at startup.

---

## 1. Core Concept

### 1.1 `import x` is a cache lookup first

```text
import a.b.c
  1. for each of "a", "a.b", "a.b.c" (parents first):
       if name in sys.modules: use it                       # the cache: a plain dict
       else: spec = first finder in sys.meta_path whose find_spec(name, parent.__path__) is not None
             module = spec.loader.create_module(spec) or a fresh ModuleType
             sys.modules[name] = module                     # inserted BEFORE executing
             spec.loader.exec_module(module)                # runs the top-level code, once
             (on exception: remove from sys.modules, re-raise)
  2. bind the name "a" in the importing namespace
```

Key facts:

- **A module's top-level code runs once per process.** Every later `import` is a dict lookup in `sys.modules`. That's why module-level objects act as singletons, and why `importlib.reload()` is the only way to re-execute a module (old references held elsewhere still point at the old objects).
- **`sys.meta_path`** by default holds `BuiltinImporter` (C modules compiled into the interpreter), `FrozenImporter` (frozen stdlib bootstrap modules), and `PathFinder`. `PathFinder` walks `sys.path`, or the parent package's `__path__` for submodules, asking a *path entry finder* (`FileFinder` for directories, `zipimporter` for zips) from `sys.path_hooks`. Results are cached in `sys.path_importer_cache`.
- **A `ModuleSpec`** records `name`, `loader`, `origin` (the file), `submodule_search_locations` (non-`None` means it's a package) and `cached` (the `.pyc` path). `importlib.util.find_spec(name)` gives you the spec without executing anything.
- **Bytecode cache:** `SourceFileLoader` compiles to `__pycache__/mod.cpython-312.pyc` and revalidates it using the source's mtime and size (or a hash, for hash-based pycs used in reproducible builds). In read-only containers, precompile at build time (`python -m compileall`), or every cold start pays the compile cost.
- **Inserted before executing:** this one line explains circular imports (§2). While `a` runs its top-level code, `sys.modules["a"]` already exists but is **partially initialized**.

### 1.2 `sys.path[0]` and the shadowing trap

`sys.path[0]` is the **directory of the script you ran**. For `-c`, `-m` or the REPL it's the current directory. A file named `json.py`, `logging.py`, `asyncio.py`, `types.py` or `test.py` next to your script **shadows the stdlib module** for every import in the process, including imports made inside third-party libraries. `python -P` (3.11+) or `-I` (isolated mode) drops that entry.

### 1.3 `python -m pkg.mod` vs `python pkg/mod.py`

Run a module as a script and it executes under the name `__main__`. If anything later does `import pkg.mod`, the module **executes a second time** under its real name. You now have two copies of every global, including registries, caches, singletons and `logging` handlers attached twice (doubled log lines). Run entry points with `python -m pkg.mod` (correct `sys.path` and package context), and keep `if __name__ == "__main__":` blocks thin.

### 1.4 Import-time side effects

Anything at module top level runs at import: opening DB connections, reading env vars or config files, starting threads, calling remote APIs, loading models. Consequences:

- **Slow startup.** Diagnose it with `python -X importtime -c "import myapp" 2> imports.log`. The log is cumulative microseconds per module. Heavy offenders are usually `pandas`, `torch`, `boto3` service models and cloud SDKs.
- **Fork hazards.** With gunicorn `--preload` (or any fork after import), a socket opened at import is **shared by every worker**, and responses get interleaved.
- **Untestable code.** Importing the module for a unit test connects to production.

**Rule:** module top level contains only definitions and cheap constants. Do I/O in an explicit `startup()`, an ASGI lifespan handler (Day 31), or a lazily-initialized `functools.cache` factory.

**Lazy imports:** move a heavy import inside the function that needs it (it's a cheap `sys.modules` hit after the first call), or use `importlib.util.LazyLoader`, which defers `exec_module` until the first attribute access (shown in the code below). PEP 810 adds explicit `lazy import` syntax targeted at Python 3.15. Check its status for the version you deploy.

### 1.5 Packaging in one screen

| Concept | What it is | Production note |
|---|---|---|
| `pyproject.toml` | `[build-system]` (which backend builds you: hatchling, setuptools, flit-core, uv_build) + `[project]` (name, version, `dependencies`, `requires-python`, `[project.scripts]`) | One file. `setup.py` is legacy; it executes arbitrary code at build time |
| **sdist** | A source tarball; installing it runs the build backend | Slow, needs compilers for C extensions, non-deterministic |
| **wheel** (`.whl`) | A zip with a pre-built layout and tags `{python}-{abi}-{platform}`, e.g. `cp312-cp312-manylinux_2_28_x86_64` or `cp312-abi3-…` | Installing a wheel **unpacks files, executes no code**. Ship wheels |
| **Entry points** | `[project.scripts] mytool = "mypkg.cli:main"` → the installer generates a `mytool` executable | Never rely on `python path/to/script.py` in containers |
| **Editable install** (PEP 660) | `pip install -e .`: a `.pth` file or import hook points at your source tree | Tests pass against the *source tree*, even when the built wheel is missing files |
| **Ranges vs locks** | Libraries declare *ranges* (`httpx>=0.27,<1`). Applications deploy a **lock** with exact versions + hashes | `uv lock` / `uv.lock`, `pip-compile --generate-hashes`; the standard format is `pylock.toml` (PEP 751) |
| **src layout** | `src/mypkg/...`, so the package is *not* importable from the repo root | Forces tests to import the **installed** package, so missing modules and data files show up in CI |

---

## 2. Production Gotcha: circular imports that fail only at startup, and only sometimes

```python
# app/models.py
from app.pricing import apply_discount       # models -> pricing

class Order:
    def __init__(self, total: float) -> None:
        self.total = apply_discount(total)

# app/pricing.py
from app.models import Order                 # pricing -> models  (added for a type hint!)

def apply_discount(total: float) -> float: ...
def describe(order: Order) -> str: ...
```

`import app.models`:

1. `sys.modules["app.models"]` = empty module, then models' top-level code starts running.
2. Line 1 imports `app.pricing`, whose top-level code starts running.
3. `from app.models import Order`: `app.models` **is** in `sys.modules`, but it's halfway through its own line 1, so `Order` doesn't exist yet.
4. 💥 `ImportError: cannot import name 'Order' from partially initialized module 'app.models' (most likely due to a circular import)`.

**Why "only sometimes":** if some other module happens to `import app.pricing` *first*, the order reverses. Now pricing starts, imports models, models does `from app.pricing import apply_discount` from a partially-initialized pricing module, and it fails differently, or not at all, depending on which names are already defined. **Import order depends on which entry point, test file or worker imported what first.** Tests pass and production crashes.

**Fixes, in order of preference:**

1. **Type-only import:** `from __future__ import annotations` + `if TYPE_CHECKING: from app.models import Order`. Annotations become strings that are never evaluated at runtime, so the runtime cycle disappears. This covers the most common case, where the cycle exists only for type hints.
2. **Restructure:** move the shared piece (`apply_discount`, or a `Protocol`/dataclass both sides need) into a third module that depends on neither. Cycles usually signal a missing layer.
3. **Import the module, not the name:** `import app.models` and use `app.models.Order` at *call time*. Attribute lookup is deferred until the function runs, by which point both modules are fully initialized.
4. **Function-local import** as a last resort. It works, but it hides the dependency.

Enforce the layering in CI with `import-linter` contracts, or `ruff`'s banned-import rules. Don't rely on review to catch it.

---

## 3. Runnable Code Example

It builds two throwaway packages in a temp dir, then shows specs, the once-only execution of module bodies, a logging meta-path finder, `LazyLoader`, the circular-import crash, and the `TYPE_CHECKING` fix. Save as `day29_imports.py`.

```python
"""
Day 29 — The import system, hands-on: specs, sys.modules caching, a logging
meta-path finder, LazyLoader, and a circular import (broken, then fixed).

Run:  python day29_imports.py        (Python 3.12+)
"""
from __future__ import annotations

import importlib
import importlib.abc
import importlib.machinery
import importlib.util
import os
import sys
import tempfile
import textwrap
from collections.abc import Sequence
from pathlib import Path
from types import ModuleType


class ImportAuditor(importlib.abc.MetaPathFinder):
    """Sees every import that MISSES sys.modules. Returning None means 'not mine, keep searching'."""

    def __init__(self, prefix: str) -> None:
        self.prefix = prefix
        self.seen: list[str] = []

    def find_spec(
        self,
        fullname: str,
        path: Sequence[str] | None,
        target: ModuleType | None = None,
    ) -> importlib.machinery.ModuleSpec | None:
        if fullname.startswith(self.prefix):
            self.seen.append(fullname)
        return None


BROKEN: dict[str, str] = {
    "__init__.py": "",
    "models.py": """
        from shop_broken.pricing import apply_discount      # models -> pricing

        class Order:
            def __init__(self, total: float) -> None:
                self.total = apply_discount(total)
    """,
    "pricing.py": """
        from shop_broken.models import Order                # pricing -> models: CYCLE

        def apply_discount(total: float) -> float:
            return round(total * 0.9, 2)

        def describe(order: Order) -> str:
            return f"order total={order.total}"
    """,
}

FIXED: dict[str, str] = {
    "__init__.py": "",
    "models.py": """
        print("  [shop_fixed.models] module body executing")
        from shop_fixed.pricing import apply_discount

        class Order:
            def __init__(self, total: float) -> None:
                self.total = apply_discount(total)
    """,
    "pricing.py": """
        from __future__ import annotations
        from typing import TYPE_CHECKING

        if TYPE_CHECKING:                                   # type checkers see it; runtime never runs it
            from shop_fixed.models import Order

        def apply_discount(total: float) -> float:
            return round(total * 0.9, 2)

        def describe(order: Order) -> str:
            return f"order total={order.total}"
    """,
    "heavy.py": """
        print("  [shop_fixed.heavy] expensive module body executing (imagine torch/pandas)")
        ANSWER = 42
    """,
}


def write_pkg(root: Path, name: str, files: dict[str, str]) -> None:
    pkg = root / name
    pkg.mkdir()
    for filename, source in files.items():
        (pkg / filename).write_text(textwrap.dedent(source), encoding="utf-8")


def lazy_import(name: str) -> ModuleType:
    """importlib recipe: create the module now, run its body on first attribute access."""
    spec = importlib.util.find_spec(name)
    if spec is None or spec.loader is None:
        raise ModuleNotFoundError(name)
    loader = importlib.util.LazyLoader(spec.loader)
    spec.loader = loader
    module = importlib.util.module_from_spec(spec)
    sys.modules[name] = module
    loader.exec_module(module)
    return module


def main() -> None:
    with tempfile.TemporaryDirectory() as tmp:
        root = Path(tmp)
        write_pkg(root, "shop_broken", BROKEN)
        write_pkg(root, "shop_fixed", FIXED)
        sys.path.insert(0, str(root))
        auditor = ImportAuditor("shop_")
        sys.meta_path.insert(0, auditor)
        try:
            print("-- 1. find_spec: locate without executing")
            spec = importlib.util.find_spec("shop_fixed.models")
            assert spec is not None and spec.origin is not None
            print(f"  name={spec.name} origin=.../{os.path.basename(spec.origin)} "
                  f"loader={type(spec.loader).__name__} parent={spec.parent}")
            print("  executed yet?", "shop_fixed.models" in sys.modules)

            print("\n-- 2. module bodies run once; later imports are dict lookups")
            first = importlib.import_module("shop_fixed.models")
            second = importlib.import_module("shop_fixed.models")    # prints nothing
            print("  same object:", first is second is sys.modules["shop_fixed.models"])
            print("  Order(100).total =", first.Order(100).total)
            importlib.reload(first)                                  # re-executes the body
            print("  (reload re-ran the body; old Order class objects still exist elsewhere)")

            print("\n-- 3. LazyLoader: body deferred until first attribute access")
            heavy = lazy_import("shop_fixed.heavy")
            print("  lazy module created, type:", type(heavy).__name__)
            print("  heavy.ANSWER =", heavy.ANSWER)

            print("\n-- 4. circular import")
            try:
                importlib.import_module("shop_broken.models")
            except ImportError as exc:
                print("  ImportError:", exc)
            print("  failed modules removed from sys.modules:",
                  "shop_broken.models" not in sys.modules)

            print("\n-- 5. imports that missed the cache (seen by our meta-path finder):")
            print("  ", auditor.seen)
        finally:
            sys.meta_path.remove(auditor)
            sys.path.remove(str(root))
            for name in [m for m in sys.modules if m.startswith("shop_")]:
                del sys.modules[name]


if __name__ == "__main__":
    main()
```

**Expected output:**

```text
-- 1. find_spec: locate without executing
  name=shop_fixed.models origin=.../models.py loader=SourceFileLoader parent=shop_fixed
  executed yet? False

-- 2. module bodies run once; later imports are dict lookups
  [shop_fixed.models] module body executing
  same object: True
  Order(100).total = 90.0
  [shop_fixed.models] module body executing
  (reload re-ran the body; old Order class objects still exist elsewhere)

-- 3. LazyLoader: body deferred until first attribute access
  lazy module created, type: _LazyModule
  [shop_fixed.heavy] expensive module body executing (imagine torch/pandas)
  heavy.ANSWER = 42

-- 4. circular import
  ImportError: cannot import name 'Order' from partially initialized module 'shop_broken.models' (most likely due to a circular import) (<tmpdir>/shop_broken/models.py)
  failed modules removed from sys.modules: True

-- 5. imports that missed the cache (seen by our meta-path finder):
   ['shop_fixed', 'shop_fixed.models', 'shop_fixed.models', 'shop_fixed.pricing', 'shop_fixed.models', 'shop_fixed.heavy', 'shop_broken', 'shop_broken.models', 'shop_broken.pricing']
```

Read step 5 carefully. `shop_fixed.models` appears **three times**: once for `find_spec` in step 1 (which also imported the parent package `shop_fixed`, but not the submodule itself), once for the real import, and once for `reload`, which re-runs the finder search. The second `import_module` call doesn't appear at all, because it was a `sys.modules` hit.

**Exercises (10 min):**

1. Change `FIXED["models.py"]` to `import shop_fixed.pricing` and call `shop_fixed.pricing.apply_discount(total)` inside `__init__`. Then reintroduce a *runtime* `import shop_fixed.models` at the top of pricing. Confirm it still works (fix #3) and explain why.
2. Run `python -X importtime -c "import asyncio" 2>&1 | sort -t'|' -k2 -n | tail -5` (or open the output in a text editor on Windows) to find the slowest transitive imports.
3. Create `json.py` next to a script that does `import json; print(json.__file__)`. Run it normally, then with `python -P`.

---

## 4. Curated Video Resources

| Priority | Search on YouTube | Channel / Speaker | Why |
|---|---|---|---|
| ⭐ Primary | `David Beazley Modules and Packages Live and Let Die PyCon 2015` | **PyCon US** (David Beazley) | A three-hour deep dive into everything `import` does. Watch the first hour today |
| Deep dive | `Brett Cannon how import works PyCon` | **PyCon US** (Brett Cannon, author of importlib) | The finder/loader/spec protocol from the person who designed it |
| Practical | `anthonywritescode pyproject.toml packaging` / `ArjanCodes uv python` | **anthonywritescode**, **ArjanCodes** | Modern packaging, entry points, and `uv` lockfile workflows |

---

[← Day 28 — Consolidation](../week-04/day-28-consolidation.md) · [Day 30 — Topological Sort →](day-30-topological-sort.md)
