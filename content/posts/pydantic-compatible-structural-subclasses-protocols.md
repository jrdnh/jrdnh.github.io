+++
title = 'Pydantic Compatible Structural Subclasses (Protocols)'
date = 2024-05-31T18:26:12-04:00
draft = true
+++

[`Protocols`](https://typing.readthedocs.io/en/latest/spec/protocol.html) are Python's built-in mechanism to define interfaces that don't require explicit, nominal subtyping for comparison. They were introduced in [`PEP 544`](https://peps.python.org/pep-0544/) to accomodate Python's duck typing paradigm where any object is acceptable as long as it looks and quacks correctly.

`Protocols` can be used as Pydantic fields [if they are `@runtime_checkable`](https://github.com/pydantic/pydantic/discussions/5767#discussioncomment-5919490). Values are checked against the protocol field using an `isinstance` comparison, and models with protocol fields must allow arbitrary types.

```python
from typing import Protocol, runtime_checkable
from pydantic import BaseModel, ConfigDict

@runtime_checkable
class FooProtocol(Protocol):  # Protocol definition
    bar: int

    def baz(self, x: int) -> int: ...

class GoodFoo(BaseModel):  # Good concrete implementation
    bar: int

    def baz(self, x: int) -> int:
        return x

class FooContainer(BaseModel):  # Model with protocol as a field
    model_config = ConfigDict(arbitrary_types_allowed=True)

    foo: FooProtocol

FooContainer(foo=GoodFoo(bar=42))  # No errors
FooContainer(foo='Foo')  # Raises a `ValidationError`
```

However, there are a few shortfalls with this method that prevents the `foo` field from being fully Pydantic compatible.

1. **Schema generation:** Unsurprisingly, schema generation fails.

```python
FooContainer.model_json_schema()
# raises pydantic.errors.PydanticInvalidForJsonSchema
```

2. **Model validation:** Similarly, `FooContainer` can't validate models because it don't know what concrete type of `FooProtocol` to use.

```python
FooContainer.model_validate(FooContainer(foo=GoodFoo(bar=42)).model_dump())
# raises pydantic.errors.PydanticValueError
```

3. **Protocol member validation:** At runtime, `isinstance` does not verify attribute types. Any object that has attributes with matching names will pass at runtime, even if they are the wrong types.

```python
class BadFoo(BaseModel):
    bar: str  # this should be an `int`

    def baz(self, x: str) -> list[str]:  # `x` and the return type should be `int`
        return [x]

FooContainer(foo=BadFoo(bar='bar'))  # No errors at runtime
```

This behavior matches Python's intent as a dynamically typed language and `Protocol`'s motivation as primarily a static type checking tool. Running this code through Mypy or Pyright will correctly identify the typing issues with this code.

```
Argument "foo" to "FooContainer" has incompatible type "BadFoo"; expected "FooProtocol"
```
