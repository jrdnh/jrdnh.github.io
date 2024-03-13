+++
title = 'Pydantic With Third Party Types'
date = 2023-12-13T21:52:40-05:00
draft = false
+++

Validating and structuring data models with [Pydantic](https://docs.pydantic.dev/latest/) is a breeze, until you need to use incompatible third-party data types. This post walks through the process of making the `dateutil.relativedelta.relativedelta` type from the widely-used [`python-dateutil`](https://github.com/dateutil/dateutil) package fully Pydantic-compatible, including JSON schemas.

----

There are several straight-forward approaches if you only need to [validate](https://docs.pydantic.dev/latest/concepts/validators/) or [serialize](https://docs.pydantic.dev/latest/concepts/serialization/) custom types. It gets tricker if you also want schemas. The process here follows the principles from the Pydantic [documentation](https://docs.pydantic.dev/latest/concepts/types/#customizing-validation-with-__get_pydantic_core_schema__):

1. Create a Pydantic model that represents the third-party type
2. Modify the model's validation function to return an instance of the *third-party* type
3. Add metadata to the third-party type that tells Pydantic how to validate, serialize, and generate schemas for it

----
## Exploring `relativedelta`

`relativedelta` is a date offset type. It's similar to the `datetime.timedelta` type, but it includes other relative offsets such as `years` and `months` as well absolute offests such as the year or the calendar day of the month. Relative offsets end with an "s" (e.g. `month=1` offsets to January, `months=1` increments by one month).

The internal state of a `relativedelta` object is a dictionary with the following keys:

```python
import json
from dateutil.relativedelta import relativedelta

print(json.dumps(relativedelta().__dict__, indent=2))
# {
#   "years": 0,
#   "months": 0,
#   "days": 0,
#   "leapdays": 0,
#   "hours": 0,
#   "minutes": 0,
#   "seconds": 0,
#   "microseconds": 0,
#   "year": null,
#   "month": null,
#   "day": null,
#   "hour": null,
#   "minute": null,
#   "second": null,
#   "microsecond": null,
#   "weekday": null,
#   "_has_time": 0
# }
```

We can ignore `_has_time` which is set at initialization. All of the other attributes are (optional) `int`s except for `weekday` which is a `dateutil._common.weekday` object. In order to make `relativedelta` Pydantic compatible, we should also make `weekday` Pydantic compatible. 

## Make `weekday` Pydantic compatible

### 1. Create a Pydantic model for `weekday`

`weekday` uses [slots](https://docs.python.org/3/reference/datamodel.html#slots) and has the following state:

```python
from dateutil._common import weekday

weekday(0).__slots__
# ['weekday', 'n']
```

The first step is to create a Pydantic model that represents `weekday`:

```python
from pydantic import BaseModel, Field

class WeekdayAnnotations(BaseModel):
    weekday: int = Field(ge=0, le=6)
    n: int | None = None
```

The `weekday` attribute is the day of the week, so it must be bounded by 0 and 6. `n` is the offset for number of weeks (e.g. `{weekday: 0, n: 2}` is the second Monday after the current date). `n` is optional and unbounded.

### 2. Modify the `WeekdayAnnotations` validation method

The `model_validator` decorator can be used to hook into the validate process. [`model_validator`](https://docs.pydantic.dev/latest/api/functional_validators/#pydantic.functional_validators.model_validator) has several modes which allow you to determine when and how customized validation occurs.

In this case, if the user passes in an instance of the `dateutil._common.weekday` type, we don't need to do any more validation and will return it immediately. Otherwise, we want to run the normal Pydantic validation logic on the value that was passed in (i.e. make sure there is a `weekday` attribute that has an integer value between 0 and 6, etc.). The `"wrap"` mode passes a `handler` function as the second argument. The `handler` function calls the next validation step. In other words, it will call the normal model validation logic. 

The return value from the `handler` will either be a `WeekdayAnnotations` object or a `dateutil._common.weekday` object. If it is a `WeekdayAnnotations` object, we need to convert it to a `dateutil._common.weekday` object. 

```python
from pydantic import model_validator

class WeekdayAnnotations(BaseModel):
    weekday: int = Field(ge=0, le=6)
    n: int | None = None

    @model_validator(mode="wrap")
    def _validate(value, handler) -> weekday:
        # if already dateutil._common.weekday instance, return it
        if isinstance(value, weekday):
            return value

        # otherwise run model validation which returns either a
        # a WeekdayAnnotations or dateutil._common.weekday object
        validated = handler(value)

        if isinstance(validated, weekday):
            return validated

        kwargs = {k: v for k, v in dict(validated).items() if v is not None}
        return weekday(**kwargs)
```

For types that use `__slots__`, we also need to define a serialization function. 

> **Note**: Defining a serialization function is only required for types that use `__slots__`. It is not required for most types that use the typical `__dict__` structure. There isn't a clear reason for this behavior as far as I can [tell](https://github.com/pydantic/pydantic/discussions/8351).

Defining a [serialization](https://docs.pydantic.dev/latest/api/functional_serializers/#pydantic.functional_serializers.model_serializer) function is similar defining a validation function, except the serialization modes are different. We'll use the `"plain"` mode which just replaces the built-in serializtion process the our custom function.

```python 
from pydantic import model_serializer

class WeekdayAnnotations(BaseModel):
    ...
    @model_serializer(mode="plain")
    def _serialize(self: weekday):
        return {"weekday": self.weekday, "n": self.n}
```

Note that the first argument to our serialization function *must* be named `self` when using the `@model_serializer` decorator. The custom function just returns a `dict` with the `weekday` and `n` attributes.

### 3. Add metadata to `weekday`

The final step is to tell Pydantic to use the validation, serialization, and schema generation logic contained in the `WeekdayAnnotations` model for the `dateutil._common.weekday` type. We can do this by adding metadata to the `weekday` type using `Annotated`.

```python
from typing import Annotated

Weekday = Annotated[weekday, WeekdayAnnotations]
```

If you examine `Weekday`'s metadata, you'll see that it contains the `WeekdayAnnotations` model.

```python
Weekday.__metadata__
# (<class '__main__.WeekdayAnnotations'>,)
```

Now we can use `Weekday` directly using `pydantic.TypeAdapter`.

```python
from pydantic import TypeAdapter

WeekdayAdapter = TypeAdapter(Weekday)
WeekdayAdapter.json_schema()
# {'properties': {'weekday': {'maximum': 6, 'minimum': 0, 'title': 'Weekday', 'type': 'integer'}, 'n': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'N'}}, 'required': ['weekday'], 'title': 'WeekdayAnnotations', 'type': 'object'}
my_day = WeekdayAdapter.validate_python({"weekday": 2, "n": 2})
WeekdayAdapter.dump_python(my_day)
# {'weekday': 2, 'n': 2}
```

Notice that values returned by `WeekdayAdapter.validate_python` are pure `dateutil._common.weekday` objects, they are *not* any sort of wrapped object.

```python
type(my_day) == weekday
# True
```

`Weekday` can also be used as a field type in other Pydantic models. Again, the attribute types will be preserved as straight `dateutil._common.weekday` types. Since our validator accepts both `dict`s and `dateutil._common.weekday` objects, we can pass either into the model constructor as well.

```python
class Foo(BaseModel):
    day: Weekday

Foo(day={"weekday": 2, "n": 3})
# Foo(day=WE(+3))
Foo(day=weekday(2, 3))
# Foo(day=WE(+3))
type(Foo(day=weekday(2, 3)).day) == weekday
# True
```


## Make `relativedelta` Pydantic compatible

Using the `Weekday` type we just created, we can follow the same process to make `relativedelta` compatible with Pydantic.

### 1. Create a Pydantic model for `relativedelta`

If you look at the `dateutil` [documentation](https://dateutil.readthedocs.io/en/stable/relativedelta.html), there are several additional fields that may be used in the `relativedelta` constructor which are converted in initialization. These are: `weeks` (converted to `days`), `yearday` and `nlyearday` (converted to `day`/`month`/`leapdays`). Finally, a `relativedelta` object can be created by passing two dates spanning the offset interval. The `relativedelta` object will be the difference between the two dates.

The full corresponding Pydantic model looks like this:

```python
from typing import Optional

class RelativeDeltaAnnotation(BaseModel):
    years: int | None = None
    months: int | None = None
    days: int | None = None
    hours: int | None = None
    minutes: int | None = None
    seconds: int | None = None
    microseconds: int | None = None
    year: int | None = None
    # recommended way to avoid potential errors for compound types with constraints
    # https://docs.pydantic.dev/dev/concepts/fields/#numeric-constraints
    month: Optional[Annotated[int, Field(ge=1, le=12)]] = None
    day: Optional[Annotated[int, Field(ge=0, le=31)]] = None
    hour: Optional[Annotated[int, Field(ge=0, le=23)]] = None
    minute: Optional[Annotated[int, Field(ge=0, le=59)]] = None
    second: Optional[Annotated[int, Field(ge=0, le=59)]] = None
    microsecond: Optional[Annotated[int, Field(ge=0, le=999999)]] = None
    weekday: Weekday | None = None
    leapdays: int | None = None
    # validation only fields
    yearday: int | None = Field(None, exclude=True)
    nlyearday: int | None = Field(None, exclude=True)
    weeks: int | None = Field(None, exclude=True)
    dt1: int | None = Field(None, exclude=True)
    dt2: int | None = Field(None, exclude=True)
```

Note that the validation-only fields have `exclude=True` so that they are _not_ included in the serialization schema. They _will_ still be included in the validation schema.

```python
from pydantic.json_schema import model_json_schema

class SchemaTest(BaseModel):
    validation_only: None = Field(None, exclude=True)

## Validation schema includes validation-only fields
print(json.dumps(model_json_schema(SchemaTest, mode='validation'), indent=2))
# {
#   "properties": {
#     "validation_only": {
#       "default": null,
#       "title": "Validation Only",
#       "type": "null"
#     }
#   },
#   "title": "SchemaTest",
#   "type": "object"
# }

## Serialization schema does not include validation-only fields
print(json.dumps(model_json_schema(SchemaTest, mode='serialization'), indent=2))
# {
#   "properties": {},
#   "title": "SchemaTest",
#   "type": "object"
# }
```

### 2. Modify `RelativeDeltaAnnotation` validation method

Similar to the `WeekdayAnnotations` model, we need to modify the validation function to return the third-party type instead of the Pydantic model.

```python
from pydantic_core import core_schema

class RelativeDeltaAnnotation(BaseModel):
    ...
    @model_validator(mode="wrap")
    def _validate(
        value, handler: core_schema.ValidatorFunctionWrapHandler
    ) -> relativedelta:
        if isinstance(value, relativedelta):
            return value

        validated = handler(value)
        if isinstance(validated, relativedelta):
            return validated

        kwargs = {k: v for k, v in dict(validated).items() if v is not None}
        return relativedelta(**kwargs)
```

Since the `relativedelta` does *not* use `__slots__`, we don't need to define a custom serialization function.

### 3. Add metadata to `relativedelta`

Finally, we can use `Annotated` again to add metadata to the `relativedelta` type.

```python
RelativeDelta = Annotated[relativedelta, RelativeDeltaAnnotation]
```

Just like `Weekday`, `RelativeDelta` fields will resolve to pure `relativedelta` objects. `RelativeDelta` field annotations can be used in Pydantic models to provide full validation, serialization, *and* schema generation support while the actual model attributes remain `relativedelta` objects.

```python
from datetime import date
from dateutil.relativedelta import TU

from datetime import date

class RecurringPayment(BaseModel):
    amount: float
    origination: date
    frequency: RelativeDelta
    periods: int

    def payments(self):
        for i in range(self.periods):
            yield self.origination + self.frequency * i


mortgage = RecurringPayment(amount=100.0,
                            origination=date(2020, 1, 1),
                            frequency=relativedelta(months=1),
                            periods=12)

list(mortgage.payments())
# [datetime.date(2020, 1, 1), datetime.date(2020, 2, 1), ...]
mortgage.frequency
# relativedelta(months=+1)
type(mortgage.frequency)
# <class 'dateutil.relativedelta.relativedelta'>
mortgage.model_dump_json()
# '{"amount":100.0,"origination":"2020-01-01","frequency":{"years":0,"months":1,"days":0,"leapdays":0,"hours":0,"minutes":0,"seconds":0,"microseconds":0,"year":null,"month":null,"day":null,"hour":null,"minute":null,"second":null,"microsecond":null,"weekday":null},"periods":12}'
RecurringPayment.model_json_schema(mode="serialization")
#{'$defs': {'RelativeDeltaAnnotation': {'properties': {'years': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Years'}, 'months': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Months'}, 'days': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Days'}, 'hours': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Hours'}, 'minutes': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Minutes'}, 'seconds': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Seconds'}, 'microseconds': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Microseconds'}, 'year': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Year'}, 'month': {'anyOf': [{'maximum': 12, 'minimum': 1, 'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Month'}, 'day': {'anyOf': [{'maximum': 31, 'minimum': 0, 'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Day'}, 'hour': {'anyOf': [{'maximum': 23, 'minimum': 0, 'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Hour'}, 'minute': {'anyOf': [{'maximum': 59, 'minimum': 0, 'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Minute'}, 'second': {'anyOf': [{'maximum': 59, 'minimum': 0, 'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Second'}, 'microsecond': {'anyOf': [{'maximum': 999999, 'minimum': 0, 'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Microsecond'}, 'weekday': {'anyOf': [{'$ref': '#/$defs/WeekdayAnnotations'}, {'type': 'null'}], 'default': None}, 'leapdays': {'anyOf': [{'type': 'integer'}, {'type': 'null'}], 'default': None, 'title': 'Leapdays'}}, 'title': 'RelativeDeltaAnnotation', 'type': 'object'}, 'WeekdayAnnotations': {'title': 'WeekdayAnnotations', 'type': 'object'}}, 'properties': {'amount': {'title': 'Amount', 'type': 'number'}, 'origination': {'format': 'date', 'title': 'Origination', 'type': 'string'}, 'frequency': {'$ref': '#/$defs/RelativeDeltaAnnotation'}, 'periods': {'title': 'Periods', 'type': 'integer'}}, 'required': ['amount', 'origination', 'frequency', 'periods'], 'title': 'RecurringPayment', 'type': 'object'}
```

## Alternative approaches

This approach uses the `@model_validator` decorator to provide an updated validation function that returns the third-party type. The Pydantic documentation uses a [different](https://docs.pydantic.dev/latest/concepts/types/#handling-third-party-types) hook into the validation process (`__get_pydantic_core_schema__`). The documentation approach requires you to define the `core_schema.CoreSchema` response directly. 

If the third-party type can be easily constructed from the `core_schema` helper functions (e.g. `core_schema.int_schema/str_schema/etc.`) or if you need ultimate flexibility in defining the schema, then `__get_pydantic_core_schema__` approach may be better. However, if the third-party type has many attributes like in the case of `relativedelta`, I find it more natural to define the shape of the schema using common `BaseModel` field annotations and overriding using the `@model_validator` decorator. These approaches aren't mutually exclisive and can be used together.

The example in the documentation also defines a `__get_pydantic_json_schema__` function. Unless you want to change the JSON schema (for example, to add examples), this function is not required.
