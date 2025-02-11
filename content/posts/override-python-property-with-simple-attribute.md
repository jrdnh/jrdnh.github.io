+++
title = 'Override Python Properties With Simple Attributes'
date = 2025-02-09T18:49:02-06:00
draft = false
+++

Both Mypy and Pyright raise errors when you try to override a property declared in an interface with a simple attribute.

```python
from abc import ABC, abstractmethod

class Base(ABC):
    @property
    @abstractmethod
    def value(self) -> int: ...

class Sub(Base):
    def __init__(self, value: int):
        self.value = value

# pyright
# error: Cannot assign to attribute "value" for class "Sub*"
#     "int" is not assignable to "property" (reportAttributeAccessIssue)

# mypy
# error: Property "value" defined in "Base" is read-only  [misc]
```

Mypy interprets `value` as being strictly read-only. Pyright considers properties and simple attributes to be [semantically](https://github.com/microsoft/pyright/issues/2678) [different](https://github.com/microsoft/pyright/issues/2072).

An alternative approach might be overriding a simple attribute in the interface with a computed property in the concrete implementation. However, this causes issues with Mypy and Pyright.

```python
class Base:
    value: int

class Sub(Base):
    @property
    def value(self) -> int:
        return 42

# pyright
# error: "value" overrides symbol of same name in class "Base"
#    "property" is not assignable to "int" (reportIncompatibleVariableOverride)

# mypy
# error: Cannot override writeable attribute with read-only property  [override]
```

Both Pyright and Mypy raise fair issues, but there still might be [cases](https://stackoverflow.com/questions/58349417/how-to-annotate-attribute-that-can-be-implemented-as-property) where you want to ensure read access to some attribute exists and don't care whether it's a i) computed property or simple attribute, or ii) whether is read-only or read-write.

One way to override simple attributes with a computed property is wrapping `property` with a return type annotation matching the decorated function.

```python
from typing import Callable, Any

def typed_property[T](func: Callable[[Any], T]) -> T:
    return property(fget=func)  # type: ignore

class NewBase:
    value: int

class AttrSub(NewBase):
    def __init__(self, value: int):
        self.value = value

class ComputedSub(NewBase):
    @typed_property
    def value(self) -> int:
        return 42

# pyright: 0 errors, 0 warnings, 0 informations
# mypy: Success: no issues found in 1 source file
```

This approach allows subclasses to override `value` using either a simple attribute or a computed property. Note that since `typed_property` only sets a getter method, trying to either set or delete `ComputedSub().value` will cause a runtime error.

The revised approach below allows users to assign a new value to `value`. Assignment will set key-value pair in the instance's `__dict__` which will be retrieved when `value` is accessed. Data descriptors on the class [take precedence](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/#object-attribute-lookup) over `__dict__` values in Python's attribute lookup process, but non-data descriptors come after `__dict__` lookups. Since the built-in `property` type is a data descriptor, the implementation below defines a new property type (that is a non-data descriptor).

```python
class TypedProperty:
    def __init__(self, func):
        self.func = func

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return self.func(obj)


def typed_property[T](func: Callable[[Any], T]) -> T:
    return TypedProperty(func)  # type: ignore


class ComputedSub(NewBase):
    @typed_property
    def value(self) -> int:
        return 42

cs = ComputedSub()
cs.value
# 42

cs.value = 12
cs.__dict__
# {'value': 12}
cs.value
# 12

del cs.value
cs.value
# 42

# pyright: 0 errors, 0 warnings, 0 informations
# mypy: Success: no issues found in 1 source file
```
