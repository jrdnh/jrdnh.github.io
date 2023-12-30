+++
title = 'Caching Generator Methods in Python'
date = 2023-12-30T13:25:41-05:00
draft = false
+++

*Each code snippet should run as a standalone example (based on Python 3.12).*

The standard library caching decorator `functools.lru_cache` has [known](https://docs.python.org/3/faq/programming.html#how-do-i-cache-method-calls) [limitations](https://rednafi.com/python/lru_cache_on_methods/) when used with instance methods. In particular, the cache is a property of the class and holds references to each function argument. Since instances themselves are the first argument to instance methods (`self`), the class will end up storing a reference to each instance. This can lead to memory leaks where the instance's reference count never reaches zero, so it's never removed by the garbage collector.

```python
from functools import lru_cache
import gc

class Foo:
    @lru_cache
    def bar(self):
        print('bar called')
        return 'bar'

    def __del__(self):
        print(f'deleting Foo instance {self}')

foo = Foo()
foo.bar()
# bar called
# 'bar'

# delete foo
del foo

gc.collect()
# 0, nothing collected
```

The simplest solution is use `@staticmethod`s. The instance object is not passed as the first argument to static methods, so the cache will not hold a reference to the instance. However, this is not always possible.

There has been some [discussion](https://github.com/python/cpython/issues/102618#issuecomment-1470778903) among core developers about addressing this issue in the standard library. In the meantime, there are a few workarounds.

1. [Use `weakref` to store a reference to the instance](###Use-weakref-to-store-a-reference-to-the-instance)
2. [Make `lru_cache` an instance attribute](###Making-the-cached-function-a-property-of-the-instance)
3. [Make the *cache* an instance attribute](###Store-just-the-cache-as-a-property-of-the-instance)

### Use `weakref` to store a reference to the instance

The first workaround is to use `weakref` to store a reference to the instance (i.e. the `self` argument). In fact, caching is a use case explicitly cited by the `weakref` [module](https://docs.python.org/3/library/weakref.html) documentation. A solution using `weakref` might look like this:

```python
import functools
import weakref

def weak_cache(func):
    cache = {}

    @functools.wraps(func)
    def wrapper(self, *args, **kwargs):
        # wrap `self` with weakref to avoid memory leaks
        weak_self = weakref.ref(self)
        key = (weak_self, args, tuple(sorted(kwargs.items())))

        if key in cache:
            return cache[key]
        
        result = func(self, *args, **kwargs)
        cache[key] = result
        return result

    return wrapper
```

Confirm that it works:

```python
class Foo:
    @weak_cache
    def bar(self):
        print('bar called')
        return 'bar'
    
    def __del__(self):
        print(f'deleting Foo instance {self}')

foo = Foo()
foo.bar()
# bar called
# 'bar'

del foo
# deleting Foo instance <__main__.Foo object at 0x10d2e1a30>
```

This approach works fine in many cases. However, not all objects can be weakly referenced. For example, in CPython built-ins such as `tuple` and `int` do not support weak references even when subclassed. Additionally, types with `__slots__` cannot by weakly referenced unless they have a `__weakref__` slot.

### Make `lru_cache` an instance attribute

Another solution is to make the cached function a property of the instance. Since the *class* no longer holds a reference to the instance through the cache, it does not keep the instance alive.

```python
class Foo:
    def __init__(self) -> None:
        self.bar = lru_cache()(self._bar)
    
    def _bar(self):
        return 'bar'
    
    def __del__(self):
        print(f'deleting Foo instance {self}')

foo = Foo()
foo.__dict__  # show that the `lru_cache` warpper is an instance attribute
# {'bar': <functools._lru_cache_wrapper object at 0x10a614bf0>}
foo.bar()
# 'bar'
foo = None
gc.collect()
# deleting Foo instance <__main__.Foo object at 0x103298b30>
# 9
```

The primary drawback with this approach is that it requires additional work to set the cached function as an attribute in the `__init__` method.

### Make the *cache* and instance attribute

A third solution is to store just the *cache* as a property of the instance. This is similar to the previous solution, but just the cache is set as the instance attribute. The function wrapper—which we will have to implement—remains an attribute of the class. This avoids the need to set the cached function as an attribute in `__init__`. It can be used simply as a decorator on instance methods.

```python
from functools import wraps

def caching_decorator(func):
    cachename = f"_cached_{func.__qualname__}_"

    @wraps(func)
    def wrapper(self, *args):
        # try to get the cache from the object
        cachedict = getattr(self, cachename, None)
        # if the object doen't have a cache, try to create and add one
        if cachedict is None:
            cachedict = {}
            setattr(self, cachename, cachedict)
        # try to return a cached value,
        # or if it doesn't exist, create it, cache it, and return it
        try:
            return cachedict[args]
        except KeyError:
            pass
        value = func(self, *args)
        cachedict[args] = value
        return value

    return wrapper
```

The decorator wraps instance methods with a function that checks the instance object for a matching cache attribute name. If the cache attribute exists, it is assumed to be a dictionary mapping function arguments to results. If the attribute does not exist, it is created. The function is then called and the result is stored in the dictionary before it is returned.

```python
class Foo:
    @caching_decorator
    def bar(self, num: int):
        print(f'bar called with {num}')
        return 'bar'
    
    def __del__(self):
        print(f'deleting Foo instance {self}')

foo = Foo()
foo.bar(1)
# bar called with 1
# 'bar'
foo.bar(1)  # cached result is returned
# 'bar'
foo.__dict__  # the cache is stored as an instance attribute
# {'_cached_Foo.bar_': {(1,): 'bar'}}
foo = None
# deleting Foo instance <__main__.Foo object at 0x108dc6b70>
```

This approach requires work to implement. For instance, this version only works with positional arguments. There is also a risk of name collisions and name pollution. It is flexible and exposes the cache to the user which is helpful for building the generator cache in the next section.

### Dealing with mutability

Mutability and hash-ability are a pervasive issues with caching in Python. The `functools.lru_cache` decorator, as well as all the other approaches discussed here, store results in a dictionary. Dictionary keys must be hashable. The function arguments are used as keys, which means they must be hashable as well. This is not a problem for most built-in, non-collection types, but it can be an issue for custom classes. Since the first argument to instance methods is the instance itself, the instance must be hashable. This [Lyft Engineering blog post](https://eng.lyft.com/hashing-and-equality-in-python-2ea8c738fb9d#:~:text=Retrieving%20an%20object%20by%20key&text=To%20have%20functionally%20correct%20dictionaries,computed%20once%20it%20is%20inserted.) has a good discussion of the issue.

The default behavior of `hash(self)` for custom classes is based on the object's ID. Object ID's are guaranteed to remain the same over the life of the object. As long as objects are immutable, this is fine. However, mutable objects with methods that access object properties can lead to unexpected behavior. Specifically, if a property that the method relies on is changed, the incorrect cached value will continue to be used because the default `hash` value will not change. 

If you can't make objects (faux) immutable, there really isn't a great solution. One option is to use a custom `__hash__` method that returns a hash based on the object's properties. This approach has the desirable effect of invalidating the cache whenever a property is changed. However, is approach is generally not a good idea because it can lead to unexpected behavior in other places the object is used in a `dict` or `set`. These issues tend to be hard to debug. For example, if the object is used as a dictionary key elsewhere then changing the object's properties will change the hash value and the item will no longer be retrievable from the dictionary. Additionally, the cache will be invalidated if *any* property is changed, even if the method does not rely on that property.

A similar option would be to create the cache's key value inside the wrapper based on the object's `self.__dict__` key-value pairs (e.g. `hash(k, v for k, v in self.__dict__.items())`). This would avoid causing issues in other placess that `__hash__` is used. However, if the method relies on mutable properties, the cache could still return the incorrect value. It also continues to invalidate the cache whenever any property is changed, even if such property isn't used by the method.

> **Conclusion:** Mutability is an issue with caching in Python. If you can't protect against properties that the method relies on from changing, consider using a different approach.

## Using `Tee` to cache generator results

The goal is to return a generator method that caches its results as `next` is called. So, if two generators are created from the method, they will share the same cache. The generator will only run once for each element of the series. This is useful for generators that are expensive to compute and are used multiple times.

This approach is inspired by [this](https://stackoverflow.com/a/10726355/18582661) Stack Overflow answer. It uses `itertools.tee` to create two copies of the generator. `tee` takes an iterable and returns multiple independent iterators that pull from the same source. The documentation has a [full explanation](https://docs.python.org/3/library/itertools.html#itertools.tee) with example implementation.

Here, `tee` is used to create two iterators with the same source generator. One iterator is returned to the user, and the other copy is stored in the cache. The next time the generator is called, the cached copy is used to create two new copies and the process repeats.

```python
from itertools import tee
from types import GeneratorType

# Get the class returned by `itertools.tee` so we can check against it later
Tee = tee([], 1)[0].__class__

def memoized(f):
    cache={}
    def ret(*args):
        # check whether the generator has been called before with same arguments
        if args not in cache:
            # if not, call the generator function
            cache[args]=f(*args)
        # check whether the result is a generator (generator method has not been called before)
        # or a Tee (generator method has been called before).
        # this should be `True` unless the decorated method doesn't return a generator (e.g. regular function)
        if isinstance(cache[args], (GeneratorType, Tee)):
            # create two new iterator copies, store one and return one
            cache[args], r = tee(cache[args])
            return r
        return cache[args]
    return ret
```

Using it with the fibonacci sequence shows that the print function is only called once for each element in the sequence.

```python
@memoized
def fibonator():
    a, b = 0, 1
    while True:
        print(f'yielding {a}')
        yield a
        a, b = b, a + b

fib1 = fibonator()
next(fib1)  # will print "yielding"
# yielding 0
# 0
fib2 = fibonator()
next(fib2)  # will not print "yielding", uses cached value
# 0
```

Copying iterables with `tee` trades memory for speed. Generators are often used precisely because they don't require storing the entire series in memory. This approach conflicts with that. There are other scenarios where generators are useful, for example where the series has an indefinite length or can't be computed ahead of time, that the tradeoff makes sense.

## Putting it all together

The following class calculates periodic revenue growing at a constant annual rate. The class has two methods: `periods` which returns a generator of `(start, end)` tuples, and `amount` which calculates revenue iteratively for the given period. Growth compounds each period based on the actual number of days and a 360 day year. This type of formula is common in financial modeling. 

```python
from dataclasses import dataclass
from datetime import date
from dateutil.relativedelta import relativedelta

@dataclass
class Operations:
    start_date: date
    freq: relativedelta
    initial_rev: float
    growth_rate: float

    def periods(self):
        """Revenue growth periods"""
        curr = (self.start_date, self.start_date + self.freq)
        while True:
            yield curr
            curr = (curr[1], curr[1] + self.freq)
    
    def amount(self, period_end: date):
        """Revenue for period"""
        revenue = self.initial_rev
        for start, end in self.periods():
            if period_end <= start:
                return revenue
            revenue *= 1 + self.growth_rate * (end - start).days / 360
```
```python
rev = Operations(start_date=date(2020, 1, 1), 
                 freq=relativedelta(months=1), 
                 initial_rev=1000.0, 
                 growth_rate=0.1)
rev.amount(date(2021, 1, 1))
# 1106.5402134963185
```
Each time `amount` is called, it iterates through a new generator returned by `periods`. This is ineffecient since the same time periods are used each time. Printing 10 years of monthly revenue requires iterating through the `while` loop 7,260 times. More generally, it requires `n * (n + 1) / 2` iterations where `n` is the number of periods.

```python
from timeit import timeit

def revenue_series():
    return [rev.amount(dt) for dt in (date(2020,1,1) + relativedelta(months=i) for i in range(120))]

count = 100
time = timeit(revenue_series, number=count)
print(f'{time / count * 1000:.2f} ms per iteration')
# 41.06 ms per iteration
```

 In a larger operating model with hundreds of line items instead of just `revenue`, this can add up to a significant amount of time. We can avoid this by caching the result each time a value is calculated in `periods`.

The following wrapper combines the instance method decorator with the generator caching decorator. 

```python
from itertools import tee
from types import GeneratorType
from functools import wraps

Tee = tee([], 1)[0].__class__

def cached_generator(func):
    cachename = f"_cached_{func.__qualname__}_"

    @wraps(func)
    def wrapper(self, *args):
        # try to get the cache from the object, or create if doesn't exist
        cache = getattr(self, cachename, None)
        if cache is None:
            cache = {}
            setattr(self, cachename, cache)
        # return tee'd generator
        if args not in cache:
            cache[args]=func(self, *args)
        if isinstance(cache[args], (GeneratorType, Tee)):
            cache[args], r = tee(cache[args])
            return r
        return cache[args]

    return wrapper
```
We can use it in the `Operations` class to make `amount` significantly faster. In this example, more than 10x faster.

```python
@dataclass
class Operations:
    start_date: date
    freq: relativedelta
    initial_rev: float
    growth_rate: float

    @cached_generator
    def periods(self):
        """Revenue growth periods"""
        curr = (self.start_date, self.start_date + self.freq)
        while True:
            yield curr
            curr = (curr[1], curr[1] + self.freq)
    
    def amount(self, period_end: date):
        """Revenue for period"""
        revenue = self.initial_rev
        for start, end in self.periods():
            if period_end <= start:
                return revenue
            revenue *= (1 + self.growth_rate * (end - start).days / 360)
```
```python
rev = Operations(start_date=date(2020, 1, 1), 
                 freq=relativedelta(months=1), 
                 initial_rev=1000.0, 
                 growth_rate=0.1)

def revenue_series():
    return [rev.amount(dt) for dt in (date(2020,1,1) + relativedelta(months=i) for i in range(120))]

count = 100
time = timeit(revenue_series, number=count)
print(f'{time / count * 1000:.2f} ms per iteration')
# 3.05 ms per iteration
```

This example highlights a scenario where caching generator methods might be helpful. There is an indefinite number of periods, and many generators are created returning the same values. The generator results are not large, so they can be stored in memory easily. `start_date` and `freq` can both be changed, so additional care is needed to ensure the cache remains valid (e.g. protecting with `@dataclass(frozen=True)` to make the attributes harder to change).

## Final thoughts

* Don't use `functools.lru_cache` with instance methods. It can lead to memory leaks.
* If you can't use `@staticmethod`, consider using `weakref` or storing the cache as an instance property.
* Use `itertools.tee` to cache generator results (or consider calculating values ahead of time if possible).
* Mutability is an issue with caching in Python. If you can't protect properties that the method relies on from changing, consider using a different approach.
