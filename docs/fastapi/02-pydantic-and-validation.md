# Pydantic and Validation

## What

Pydantic models define the shape of data (request bodies, response
payloads, settings) using type hints, and validate/parse incoming data
against that shape at runtime — the piece that actually enforces type
hints, which the interpreter itself does not (see
[Type Hints](../python/05-type-hints.md)). **Response models** tell
FastAPI what shape to serialize a handler's return value into.

## Why

Type hints alone don't validate untrusted input — an API accepting
external requests must validate at the boundary. Pydantic does this
declaratively, generates clear validation errors automatically, and doubles
as the source of truth for FastAPI's OpenAPI schema, removing the need to
maintain validation logic and API docs separately.

## How

### Defining a model

```python
from pydantic import BaseModel, EmailStr, Field

class UserCreate(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(ge=0, le=150, default=18)
```

`Field(...)` attaches constraints (length, numeric bounds, defaults) beyond
a bare type hint. `EmailStr` is one of many Pydantic-provided constrained
types for common formats.

### Validation in action

```python
@app.post("/users")
def create_user(payload: UserCreate):
    return payload
```

```bash
curl -X POST /users -d '{"name": "", "email": "not-an-email", "age": 200}'
# 422 Unprocessable Entity, with a JSON body listing each field's error
```

FastAPI runs Pydantic validation before your handler executes — invalid
requests never reach your business logic, and the error response format is
consistent across the whole API without writing per-endpoint checks.

### Response models

```python
class UserOut(BaseModel):
    id: int
    name: str
    # note: no `email` -- deliberately excluded from the response

@app.post("/users", response_model=UserOut)
def create_user(payload: UserCreate) -> UserOut:
    user = save_user(payload)  # returns some internal object with more fields
    return user
```

`response_model` (or a return type annotation FastAPI can use similarly)
filters and validates the **outgoing** data — even if the handler returns
an object with extra fields (e.g. a hashed password, internal flags), only
the fields declared on `UserOut` are serialized. This is a deliberate
security/API-contract boundary, not just documentation.

### Nested models and lists

```python
class Address(BaseModel):
    city: str
    zip_code: str

class UserOut(BaseModel):
    id: int
    name: str
    addresses: list[Address] = []
```

Pydantic validates nested structures recursively — a malformed nested
`Address` produces a field-path-qualified error (e.g.
`addresses -> 0 -> zip_code`).

### Custom validators

```python
from pydantic import field_validator, model_validator

class UserCreate(BaseModel):
    password: str
    password_confirm: str

    @field_validator("password")
    @classmethod
    def password_strength(cls, value: str) -> str:
        if len(value) < 8:
            raise ValueError("password must be at least 8 characters")
        return value

    @model_validator(mode="after")
    def passwords_match(self) -> "UserCreate":
        if self.password != self.password_confirm:
            raise ValueError("passwords do not match")
        return self
```

`field_validator` validates a single field; `model_validator` validates
across multiple fields once the whole model is assembled — needed for
cross-field rules like password confirmation.

### Settings management

```python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    database_url: str | None = None
    debug: bool = False

    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

settings = Settings()
```

`pydantic-settings` (a separate package since Pydantic v2) applies the same
validation model to environment-variable-driven configuration — useful for
catching missing/malformed config at startup rather than at first use.

### Pydantic v1 vs v2

Pydantic v2 (used by current FastAPI versions) has a different validator
API (`field_validator`/`model_validator` vs v1's `validator`/`root_validator`)
and a Rust-based core for significantly better performance. This is
version-dependent — check which major version a given codebase uses before
applying examples from documentation or tutorials.

## When to use

- Use a Pydantic model for every request body and every non-trivial
  response — never accept/return raw untyped `dict`s at the API boundary.
- Use a **separate** response model from your request/create model,
  especially to exclude sensitive fields (passwords, internal flags) from
  serialized output.
- Use custom validators for rules that can't be expressed as simple type/
  field constraints (cross-field checks, business-rule-level validation).

## When NOT to use

- Don't use Pydantic models for internal-only data structures that never
  cross a validation boundary (e.g. a purely internal computation
  result) — a dataclass may be simpler and faster (see
  [Dataclasses, Properties, and Dunder Methods](../python/07-dataclasses-and-dunder-methods.md)).
- Don't put deep business logic inside Pydantic validators — keep them
  focused on structural/format validation; complex domain rules belong in
  the service layer.
- Don't reuse the same model for both request input and response output
  when the fields genuinely differ (e.g. a password field that should
  never be echoed back).

## Common mistakes

- Returning an ORM object directly without a `response_model`, leaking
  internal fields (e.g. `hashed_password`) into the API response.
- Confusing validation errors (`422`, malformed input) with application
  errors (`404` not found, `409` conflict) — these are different concerns;
  see
  [Middleware and Exception Handling](04-middleware-and-exception-handling.md).
- Writing Pydantic v1-style validators (`@validator`) in a v2 codebase, or
  vice versa — the decorators and semantics differ between versions.
- Forgetting `EmailStr`/similar constrained types require an optional
  dependency (`email-validator`) to be installed.

## Interview questions

- Why does FastAPI need Pydantic even though Python already has type
    hints?

    **Answer:** Type hints describe intent, but they don't stop bad input at
    runtime. Pydantic is the layer that actually parses external data,
    validates it, and gives FastAPI a consistent error shape.

    ```python
    from pydantic import BaseModel
    
    class UserCreate(BaseModel):
       age: int
    ```

- What's the difference between using the same model for requests and
    responses vs separate `UserCreate`/`UserOut` models?

    **Answer:** One shared model is fine only when the fields are truly the
    same both ways. In real services, separate models are safer because input
    and output usually have different concerns, especially around sensitive
    or server-generated fields.

    ```python
    from pydantic import BaseModel

    class UserCreate(BaseModel):
       name: str

    class UserOut(BaseModel):
       id: int
       name: str
    ```

- How would you validate that two fields (e.g. password and confirmation)
    match each other?

    **Answer:** Use a model-level validator because the rule depends on more
    than one field. That's the right place for cross-field checks like
    password confirmation or date range validation.

    ```python
    from pydantic import BaseModel, model_validator
    
    class PasswordReset(BaseModel):
       password: str
       confirm_password: str

       @model_validator(mode="after")
       def passwords_match(self) -> "PasswordReset":
           if self.password != self.confirm_password:
               raise ValueError("passwords do not match")
           return self
    ```

- What does `response_model` actually do at runtime, beyond documentation?

    **Answer:** It validates and filters the outgoing data before FastAPI
    sends the response. That means extra fields from ORM objects or internal
    models do not automatically leak into the public API.

    ```python
    from fastapi import FastAPI
    from pydantic import BaseModel

    app = FastAPI()

    class UserOut(BaseModel):
       id: int
       name: str

    @app.get("/users/{user_id}", response_model=UserOut)
    def get_user(user_id: int) -> UserOut:
       return {"id": user_id, "name": "Roy", "hashed_password": "secret"}
    ```

- What changed between Pydantic v1 and v2 that you should be aware of
    when reading older FastAPI code?

    **Answer:** The validator APIs changed (`@validator` became
    `@field_validator`, `@root_validator` became `@model_validator`), and v2
    has a much faster core. If you copy snippets across versions without
    checking, you'll often get broken validation code.

## Senior-level considerations

- Separating request/response schemas from internal domain/ORM models is a
  deliberate architectural boundary — it lets internal models evolve
  without breaking the public API contract, and prevents accidental data
  leakage. For example, an internal `User` model may have
  `hashed_password` and `is_admin`, while `UserOut` should expose neither.
- Pydantic's validation happens on every request — for very
  high-throughput services, validation overhead is measurable; Pydantic
  v2's Rust core significantly reduced this cost, which is one reason
  version awareness matters here. For example, a hot ingestion endpoint
  doing thousands of validations per second feels the difference more than
  a low-traffic admin API.
- Centralizing validation in Pydantic models (rather than scattering
  manual checks through handlers) makes the API contract self-documenting
  and keeps validation logic testable in isolation from HTTP concerns. For
  example, you can unit test `UserCreate.model_validate(...)` without going
  through a FastAPI test client.
