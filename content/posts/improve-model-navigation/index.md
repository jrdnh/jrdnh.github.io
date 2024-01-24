+++
title = 'Improve Model Navigation'
date = 2024-01-23T21:27:24-05:00
draft = true
+++

*The full example code for this post is [here](https://github.com/jrdnh/blog-files/tree/main/improve-model-navigation).*

A key limitation with the financial model from the [last post ](https://jrdnh.github.io/posts/create-a-multifamily-financial-model-in-python/) was that line items could only reference subitems. Sibling and parents were unreachable. This post improves the hierarchical structure to make line items in the structure accessible to each other while preserving autocompletion and type hints.

## Making the `parent` accessible

As a refresher from the previous model, line items are modeled as callable classes. Each line item may hold subitems as attributes. A simple model might look like this:

```
NetIncome
├── Revenue
└── Expenses
    └── VariableExpenses
```

We can navigate down the tree with regular attribute access (e.g. `NetIncome().revenue`). We can navigate back up the tree by creating a `parent` attribute for each line item node. Since we want to avoid memory leaks and parent nodes already hold a reference to each child node, we can create a managed property for `parent` that holds a weak reference to the parent object.

Since we want autocompletion to work across the entire structure, we also need to make the node class generic with respect to its parent type. This will allow us to get good autocompletion and type hints for typing things like `expenses.parent.revenue` to navigate from the expenses line to the revenue line.


```python
import weakref
from typing import Optional
from pydantic import BaseModel

class Node[P](BaseModel):
    _parent: Optional[weakref.ref] = None

    @property
    def parent(self) -> P | None:
        """Parent node or None."""
        return self._parent() if self._parent is not None else None

    @parent.setter
    def parent(self, parent: Optional[P] = None):
        """Set parent."""
        self._parent = weakref.ref(parent) if parent is not None else None
```

{{< alert >}}
**Note** This snippet uses the syntax for generics from PEP 695 introduced in Python 3.12. For previous versions of Python use:
```python
from typing import Generic, TypeVar

P = TypeVar("P")
class Node(BaseModel, Generic[P]):
    ...
```
{{< /alert >}}

Now we can create concrete subclasses of `Node` that will tell us what type the `parent` should be, enabling autocompletion. For example

```python
class Parent(Node[None]):
    luftballons: int

class Child(Node[Parent]):
    pass

child = Child()
child.parent = Parent(luftballons=99)

child.parent.luftballons
```

![ autocompletion](images/autocompletion.png)
![type hint](images/typehint.png)


## Parents for arbitrarily nested children

Reproducibility for components is limited in the current structure since parent/child relationships must be strictly defined. Deeply nested children must know what the structure looks like above them in order to traverse up the right number of nodes (`parent.parent.parent...`).

We can improve navigation by adding a helper function that returns the first parent of the desired type (or raising an error if such type doesn't exist in the parent chain).

```python
from typing import Type, TypeVar

T = TypeVar("T", bound="Node")

class Node[P](BaseModel):
    ...

    def find_parent(self, cls: Type[T]) -> T:
        """Find first parent node with class `cls`."""
        if isinstance(self.parent, cls):
            return self.parent
        try:
            return self.parent.find_parent(cls)  # type: ignore
        except AttributeError:
            raise ValueError(f"No parent of type {cls} found.")
```

Note that subclasses of `cls` will be returned in this implementation. Depending on the desired behavior, strict class comparison (or a function argument for strict comparison) might be appropriate.

## Setting the `parent` attribute

It would be helpful if children nodes had their `parent` property automatically set during construction. Luckily, Pydantic has a post initialization hook that we can use to set the `parent` for any child `Node` attributes. 

```python
class Node[P](BaseModel):
    ...

    def model_post_init(self, __context: Any) -> None:
        """Post init hook to add self as parent to any 'Node' attributes."""
        super().model_post_init(__context)
        for _, v in self:
            if isinstance(v, Node):
                v.parent = self
```

After the model has been initialized, the hook iterates over each model attribute and sets the attribute's parent to `self` if it is a `Node` object.

`pydantic.BaseModel` subclasses do *not* store attributes with a leading underscore in the object's `__dict__` (which is what is iterated over in the for-loop with Pydantic models). Instead, they are stored separately in a separate `__private_attributes__` property. If the `_parent` attribute was stored in the object's `__dict__` we would also have to make sure that we don't try to set the parent's `_parent` property to itself.

## Navigating a simple model

Using the `Node` class, we can now reference line items up and across the model. A simple model that reflects the structure at the top of the post is below. Revenue is modeled as a single line. Expenses includes both fixed expenses of `100` plus variable expenses. Variable expenses equal 60% of revenues, and requires the `VariableExpenses` class to navigate up and over to the `Revenue` class.

This is a *simple* model. All values are hardcoded into the function. It is purely meant to demonstrate how the `Node` class helps climb the model structure.

```python
class Revenue(Node["NetIncome"]):
    def __call__(self, year: int):
        return 1000 * (1.1 ** (year - 2020))


class VariableExpenses(Node["NetIncome"]):
    def __call__(self, year: int):
        # Find NetIncome class
        ni = self.find_parent(NetIncome)
        return ni.revenue(year) * -0.6  # 60% of revenue


class Expenses(Node["NetIncome"]):
    variable: "VariableExpenses"

    def __call__(self, year: int):
        # total expenses = 100 + variable expenses
        return -100 + self.variable(year)


class NetIncome(Node[None]):
    revenue: "Revenue"
    expenses: "Expenses"

    def __call__(self, year: int):
        return self.revenue(year) + self.expenses(year)
```

We can confirm it works as expected.

```python
income = NetIncome(revenue=Revenue(), expenses=Expenses(variable=VariableExpenses()))
income(2025)
# 544.2040000000002
```

## Final thoughts

**Protocols:** Typing for `parent` and in `find_parent` isn't limited to concrete classes. You can use protocols introduced in [PEP 544](https://peps.python.org/pep-0544/) and added in Python 3.8 to define or match parents using structural subtyping. Protocols allow you to define or match parents based on a class interface without the underlying implementation. 
**Validation:** `Node` currently does not validate that its parent matches the type declared in concrete subclasses. This could lead to differences between the type hints and actual parent type.
