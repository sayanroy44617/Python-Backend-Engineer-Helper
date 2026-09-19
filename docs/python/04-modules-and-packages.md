# Modules and Packages

**Level:** Foundation

## What

A **module** is a single `.py` file. A **package** is a directory containing
an `__init__.py` (regular package) or without one (namespace package,
Python 3.3+) that groups related modules. This section covers the import
system, package structure, and how Python resolves imports.

## Why

Every non-trivial backend service is organized into modules and packages
(`app/api/`, `app/services/`, `app/models/`). Understanding import
resolution avoids circular import bugs, and understanding package structure
is a prerequisite for packaging your project properly (see the
**Python Packaging** section for `pyproject.toml`, `uv`, editable installs).

## How

### Modules

```python
# math_utils.py
def add(a: int, b: int) -> int:
    return a + b
```

```python
# main.py
import math_utils
from math_utils import add
from math_utils import add as sum_two

print(math_utils.add(1, 2))
print(add(1, 2))
```

### Packages

```
app/
├── __init__.py
├── api/
│   ├── __init__.py
│   └── users.py
├── services/
│   ├── __init__.py
│   └── user_service.py
└── models/
    ├── __init__.py
    └── user.py
```

```python
# app/api/users.py
from app.services.user_service import get_user
from app.models.user import User
```

`__init__.py` runs when the package is first imported — commonly used to
define the public API of a package (`__all__`) or to re-export symbols for a
cleaner import path.

```python
# app/services/__init__.py
from app.services.user_service import get_user, create_user

__all__ = ["get_user", "create_user"]
```

```python
# elsewhere
from app.services import get_user  # instead of app.services.user_service
```

### Absolute vs relative imports

```python
# Absolute (preferred for application code)
from app.models.user import User

# Relative (common inside a package's own internal modules)
from .user import User
from ..models.user import User
```

Absolute imports are clearer and don't break when a file moves within the
package hierarchy relative to its siblings; relative imports are common in
library code shipped as a package.

### The import system and `sys.path`

Python searches `sys.path` (current directory / script location, then
`PYTHONPATH`, then installed site-packages) to resolve `import` statements.
Modules are cached in `sys.modules` after first import — re-importing
doesn't re-execute the module.

### Circular imports

```python
# a.py
from b import foo  # ImportError: cannot import name 'foo' from partially
                    # initialized module 'b'

# b.py
from a import bar
```

Common fixes: restructure to remove the cycle (extract shared code to a
third module), import inside the function body (deferred import), or import
the module itself rather than a name from it (`import a` then `a.bar`).

### `__name__ == "__main__"`

```python
# cli.py
def main() -> None:
    print("running as a script")

if __name__ == "__main__":
    main()
```

When a file is run directly (`python cli.py`), Python sets `__name__` to
`"__main__"`. When the same file is *imported* (`import cli`), `__name__` is
`"cli"` instead — so the guarded block only runs on direct execution, not on
import. This lets a module double as both an importable library and a
runnable script/CLI entry point without side effects firing on import.

## When to use

- Split code into modules along **responsibility boundaries** (routing,
  business logic, data access) rather than arbitrary size.
- Use `__init__.py` re-exports to give consumers a stable, short import
  path while internal file structure can still change.
- Use namespace packages only when you specifically need to split a single
  logical package across multiple distributions — otherwise prefer regular
  packages with `__init__.py` for clarity.

## When NOT to use

- Don't put unrelated logic into `__init__.py` "because it runs first" —
  it's for public API surface, not side effects like DB connections or
  network calls.
- Don't create deep package hierarchies for a small service — 2-3 levels is
  usually enough; excessive nesting hurts discoverability.
- Avoid wildcard imports (`from module import *`) in application code — they
  pollute the namespace and hide where a name comes from.

## Common mistakes

- Circular imports caused by two modules importing from each other at
  module load time.
- Confusing `import package.module` (must reference full path) with
  `from package import module`.
- Relying on relative imports in a script run directly (`python file.py`)
  instead of as part of a package — relative imports only work when the
  module is imported as part of a package, not run as `__main__`.
- Shadowing standard library module names with your own files (e.g. naming
  a file `json.py` in your project root breaks `import json` elsewhere).

## Interview questions

1. What's the difference between a module and a package?

   **Answer:** A module is one `.py` file; a package is a directory that groups
   related modules. In real services, `users.py` is a module and `app/services/`
   is a package.

2. What causes a circular import, and how do you resolve one?

   **Answer:** A circular import happens when two modules need each other at
   import time before either has finished loading. The real fix is usually to
   move shared logic to a third module or untangle the dependency direction.

3. What's the purpose of `__init__.py`? What changed with namespace
   packages?

   **Answer:** `__init__.py` marks a regular package and can expose a clean
   public API or lightweight package setup. Namespace packages made
   `__init__.py` optional when one logical package is split across locations.

4. Why would you prefer absolute imports over relative imports in
   application code?

   **Answer:** Absolute imports are easier to read and keep working when files
   move around inside the package tree. They also make architectural boundaries
   more obvious in larger codebases.

   ```python
   from app.services.user_service import get_user
   ```

5. What is `sys.modules`, and why does re-importing a module not re-run its
   top-level code?

   **Answer:** `sys.modules` is Python's module cache keyed by module name.
   Once a module is imported, later imports reuse that same module object
   instead of executing the file again.

   ```python
   import sys
   import json
   print("json" in sys.modules)  # True
   ```

6. What does `__all__` control, and when does it matter (`from module
   import *`)?

   **Answer:** `__all__` defines which names are exported by `from module import *`.
   It mostly matters for package surfaces and re-export convenience; explicit
   imports are usually clearer in application code.

   ```python
   __all__: list[str] = ["get_user", "create_user"]
   ```

7. What does `if __name__ == "__main__":` guard against, and why does it
   matter for a file that's both imported and run directly?

   **Answer:** It prevents script-only code from running when the file is
   imported as a module. That matters because imports should be cheap and free
   of surprise side effects like starting jobs or printing output.

   ```python
   def main() -> None:
       print("run once")

   if __name__ == "__main__":
       main()
   ```

## Senior-level considerations

- Package structure encodes architectural boundaries: a `services/` package
  that imports from `api/` (instead of the reverse) is usually a sign of an
  inverted dependency and a maintainability smell. Example: `api/users.py`
  depending on `services/user_service.py` is normal; `services/user_service.py`
  depending on `api/users.py` usually means HTTP concerns leaked downward.
- Lazy/deferred imports (importing inside a function) can hide circular
  dependency problems rather than fix the underlying design issue — use
  sparingly and prefer restructuring. Example: moving a shared DTO into
  `app/common/types.py` is usually better than scattering `from ... import ...`
  inside function bodies just to dodge import errors.
- Import time matters for cold-start latency in serverless/containerized
  deployments — avoid heavy work (DB connections, large data loads) at
  module import time; defer to explicit startup hooks (e.g. FastAPI
  `lifespan`). Example: don't create a database engine by opening a connection
  at import time in `settings.py`; initialize it during app startup instead.
