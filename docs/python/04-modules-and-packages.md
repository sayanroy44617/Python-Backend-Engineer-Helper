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
2. What causes a circular import, and how do you resolve one?
3. What's the purpose of `__init__.py`? What changed with namespace
   packages?
4. Why would you prefer absolute imports over relative imports in
   application code?
5. What is `sys.modules`, and why does re-importing a module not re-run its
   top-level code?
6. What does `__all__` control, and when does it matter (`from module
   import *`)?

## Senior-level considerations

- Package structure encodes architectural boundaries: a `services/` package
  that imports from `api/` (instead of the reverse) is usually a sign of an
  inverted dependency and a maintainability smell.
- Lazy/deferred imports (importing inside a function) can hide circular
  dependency problems rather than fix the underlying design issue — use
  sparingly and prefer restructuring.
- Import time matters for cold-start latency in serverless/containerized
  deployments — avoid heavy work (DB connections, large data loads) at
  module import time; defer to explicit startup hooks (e.g. FastAPI
  `lifespan`).
