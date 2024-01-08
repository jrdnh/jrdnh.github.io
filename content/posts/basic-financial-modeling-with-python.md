+++
title = 'Basic Financial Modeling With Python'
date = 2024-01-07T15:28:54-05:00
draft = false
+++

Spreadsheets remain undefeated for financial modeling[^1]. There are a number of generic tools (e.g. [Anaplan](https://www.anaplan.com)) and industry specific tools (e.g. [Argus](https://www.altusgroup.com/argus/)), but they are best used in portfolio management-type scenarios where scale, structure, or auditability are valued over flexibility. Even then, many companies still use Excel for these activities. In underwriting and valuation situtaions, Excel dominates.

Excel is fantastic, but there a times I wish it had the full power of programming environment. Excel struggles with certain data transformations (e.g. unique attribute counts with filtering and grouping) and importing/exporting data to/from external sources.

This article plays with an approach to fundamental modeling in Python. I haven't seen other robust approaches, but I'd be interested in hearing about them![^2]

## The setup--a time-based loop

I'd like to be able to do something like this where `from_date` and `to_date` can be any two arbitrary dates. In Excel, you're generally limited to calculating values at whatever frequency columns represent (e.g. monthly, quarterly, annually). Sometimes it's helpful to have intra-period granularity to capture specific events or accruals. Since we're not restricted to a graphical grid, there's no reason to limit ourselves to a fixed intervals.

```python
obj.revenue(from_date, to_date)
# sum of revenue between dates
```

Cash flows are frequently are modeled by growing some starting value at a periodically compounding rate. In the most basic cases, such series can be modeled as a geometric series. In most real-world scenarios, growth is more complex and needs to be modeled iteratively.

> A recursive approach would also generally work. Unfortnately, when I've tried to create even medium-sized models using recursive calls, the call stack blows up and I hit the recursion limit (default of 1,000 frames typically).

The cash flow function needs to know the periods over which to compound. We can start by creating a generator function that drives each period of the model.

```python
from datetime import date
from pydantic import BaseModel
# from relativedelta import RelativeDelta, see note below

class Statement(BaseModel):
    ref_date: date
    freq: RelativeDelta

    def periods(self):
        per_start, per_end = self.ref_date, self.ref_date + self.freq
        while True:
            yield (per_start, per_end)
            per_start, per_end = per_end, per_end + self.freq

statement = Statement(ref_date=date(2020, 12, 31), freq=RelativeDelta(months=1, day=31))
s = statement.periods()
[next(s) for _ in range(3)]
# [(datetime.date(2020, 12, 31), datetime.date(2021, 1, 31)), ...]
```
> `RelativeDelta` is a Pydantic-compatible version of `dateutil.relativedelta.relativedelta`. Read about it in this [previous post](https://jrdnh.github.io/posts/pydantic-with-third-party-types/), or copy the code from [this Gist](https://gist.github.com/jrdnh/ba4936f8e403f968c1f9b7f85f828333).

Now that we have a function that yields the compounding periods, we can add a function that calculates the appropriate flows. For example using the `revenue` line item:

```python
class Statement(BaseModel):
    ref_date: date
    freq: RelativeDelta
    ref_revenue: float  # reference revenue
    rev_growth: float  # annual revenue growth rate

    def periods(self):
        per_start, per_end = self.ref_date, self.ref_date + self.freq
        while True:
            yield (per_start, per_end)
            per_start, per_end = per_end, per_end + self.freq

    def revenue(self, from_date: date, to_date: date):
        accum_revenue, period_rev = 0, self.ref_revenue
        for per_start, per_end in self.periods():
            # if the period is past the end date, break
            if per_start >= to_date:
                break
            # otherwise add pro rata period revenue to accumulated revenue
            period_length = (per_end - per_start).days
            period_rev = period_rev * (1 + self.rev_growth * period_length / 365)
            accum_revenue += (
                period_rev
                * max((min(per_end, to_date) - max(per_start, from_date)).days, 0)
                / period_length
            )
        return accum_revenue
```

First, add attributes for the starting amount of revenue and annual growth rate. Note that in this case the `ref_revenue` is the amount or revenue for the period ending on `ref_date`. In other words, revenue for the first period of the model equals `ref_revenue * (1 + rev_growth * year_frac)`.

The `revenue` function initializes a variable to hold accumulated revenue between the two input dates and a variable to hold revenue for the current period. For each `period`, the function checks if we've gone past the `to_date` and breaks if so. Otherwise, it calculates revenue for the current period and adds it to the accumulated revenue (for any portion of the current period that overlaps with the input dates).

```python
statement = Statement(
    ref_date=date(2019, 12, 31),
    freq=RelativeDelta(months=1, day=31),  # month end frequency
    ref_revenue=1000,
    rev_growth=0.1,
)
# revenue between two random dates
statement.revenue(from_date=date(2020, 2, 3), to_date=date(2020, 7, 11))
# 5439.378625854814

s = statement.periods()
[f"{statement.revenue(*next(s)):.2f}" for _ in range(5)]
# ['1008.49', '1016.51', '1025.14', '1033.56', '1042.34']
```

## Refactor to make it a bit more generic

The operating statement has a revenue line item. Let's add a line for fixed costs next. We could model fixed costs using the same method: an initial value that grows at an annual rate, compounding over each period. The code is almost exactly the same as `revenue`. This is a common pattern, so let's pull it out into a separate function.

```python
from typing import Callable, Iterable

# pull out the accumulation code from `revenue` into a separate function
def accumulate(
    from_date: date,
    to_date: date,
    start_amt: float,
    growth_rate: float,
    year_frac: Callable[[date, date], float],
    periods: Iterable[tuple[date, date]],
):
    accum_amt = 0
    period_amt = start_amt
    for per_start, per_end in periods:
        if per_start >= to_date:
            break
        period_yf = year_frac(per_start, per_end)
        period_amt = period_amt * (1 + growth_rate * period_yf)
        accum_amt += (
            period_amt
            * max(year_frac(max(per_start, from_date), min(per_end, to_date)), 0)
            / period_yf
        )
    return accum_amt


def actual365(from_date: date, to_date: date):
    """Year fraction on an actual/365 basis"""
    return (to_date - from_date).days / 365


class Statement(BaseModel):
    ref_date: date
    freq: RelativeDelta
    ref_revenue: float  # reference revenue
    rev_growth: float  # annual revenue growth rate
    ref_fixed_costs: float  # reference fixed costs
    fixed_costs_growth: float  # annual fixed costs growth rate

    def periods(self):
        per_start, per_end = self.ref_date, self.ref_date + self.freq
        while True:
            yield (per_start, per_end)
            per_start, per_end = per_end, per_end + self.freq

    def revenue(self, from_date: date, to_date: date):
        return accumulate(
            from_date,
            to_date,
            self.ref_revenue,
            self.rev_growth,
            actual365,
            self.periods(),
        )

    def fixed_costs(self, from_date: date, to_date: date):
        return accumulate(
            from_date,
            to_date,
            self.ref_fixed_costs,
            self.fixed_costs_growth,
            actual365,
            self.periods(),
        )
```

This is still a lot of code, but at least it's easier to add new line items to the operating statement. However, this model only works with `revenue` and `fixed_costs` compound at the same frequency. Additionally, `revenue` and `fixed_costs` are not re-usable if we wanted to create another operating model. 

There are two main concepts here. 
1. A **driver function** that yields each successive period in the series. In this case, each period is at a fixed interval, but you could easily imagine other scenarios.
2. A **growth function** that accumulates the flow over the appropriate period. In this case, accumulation is based *i)* on a constant annual growth rate, and *ii)* the proportionate amount of any overlapping periods. In other cases, the growth rate might not be constant or the accumulation function might not include partial periods (e.g. cash accounting for payments received on the final day of the period).

Let's create driver and growth classes that we can use for `revenue` and `fixed_costs`.

```python
class FixedPeriodSeries(BaseModel):
    ref_date: date
    freq: RelativeDelta

    def periods(self):
        per_start, per_end = self.ref_date, self.ref_date + self.freq
        while True:
            yield (per_start, per_end)
            per_start, per_end = per_end, per_end + self.freq

class GrowingSeries(FixedPeriodSeries):
    ref_amount: float
    growth_rate: float

    def __call__(self, from_date: date, to_date: date):
        return accumulate(
            from_date,
            to_date,
            self.ref_amount,
            self.growth_rate,
            actual365,
            self.periods(),
        )

class OperatingStatement(BaseModel):
    revenue: GrowingSeries
    fixed_costs: GrowingSeries

    # income
    def __call__(self, from_date: date, to_date: date):
        return self.revenue(from_date, to_date) - self.fixed_costs(from_date, to_date)
```

`FixedPeriodSeries` is the period driver function. It yield periods based on a fixed time interval. The `GrowingSeries` class is the growth function. It determines amount based on fixed annual growth and includes partial periods. They are tied together to reflect the chosen modeling structure.

This is the extent of the line item modeling we are going to do here. If we were building out a full three statement model in practice, we'd probably want to think through the common patterns and create a more robust set of helper classes. For example, we might have line items with contingent events (e.g. an earnout paid when certain performance metrics are achieved) where driver periods are dynamic. Or you might want to different short- and long-term growth rates. Having the growth function inherit from the driver function may not make sense in a larger model either.

```python
start_date = date(2019, 12, 31)

statement = OperatingStatement(
    revenue=GrowingSeries(
        ref_date=start_date,
        freq=RelativeDelta(months=1, day=31),
        ref_amount=1000,
        growth_rate=0.1,
    ),
    fixed_costs=GrowingSeries(
        ref_date=start_date,
        freq=RelativeDelta(months=3, day=31),
        ref_amount=500,
        growth_rate=0.05,
    ),
)

statement(start_date, date(2020, 12, 31))  # income for the next year
# 10607.424004381839
statement.revenue(start_date, date(2020, 12, 31))  # revenue for the next year
# 12670.746995800495
```

This setup allows us to nicely access line items (and sub-line items if we had anything) using regular dot notation since each line item is a regular attribute.

We can confirm that there aren't any recursion issues as well. We can recreate the same model with *daily* revenue compounding and calculate total income over the next 10,000 days (December 31, 2019 to May 18, 2047).

```python
daily_statement = OperatingStatement(
    revenue=GrowingSeries(
        ref_date=start_date,
        freq=RelativeDelta(days=1),  # daily frequency
        ref_amount=1000,
        growth_rate=0.1,
    ),
    fixed_costs=GrowingSeries(
        ref_date=start_date,
        freq=RelativeDelta(months=3, day=31),
        ref_amount=500,
        growth_rate=0.05,
    ),
)

end_date = start_date + RelativeDelta(days=10_000)
daily_statement.revenue(start_date, end_date)
# 52855286.28641588
```

## Prettier outputs

One of the great things about Excel is that you can see the data easily. Every cell is visible. It would be really nice to present the outputs in a familiar format here: periods as columns and line items as rows.

Each node in our model is a callable Pydantic `BaseModel`. We can create some helper functions to iterate over each model field and recursively call the node for each period. This will give us a nested dictionary of values representing our pro forma model.

```python
def field_values(series: BaseModel, periods):
    """Recursively get values of all FixedPeriodSeries fields"""
    values = {}
    try:
        for field in series.model_fields:
            try:
                values[field] = field_values(getattr(series, field), periods)
            except TypeError:
                pass
    except AttributeError:
        pass
    values.update({"": [series(*period) for period in periods]})
    return values
```
Let's print out the first year of the model monthly.
```python
from itertools import pairwise
import json

periods = list(pairwise([start_date + RelativeDelta(months=1, day=31) * i for i in range(13)]))
pro_forma = field_values(statement, periods)
print(json.dumps(pro_forma, indent=2))
# {
#   "revenue": {
#     "": [1008.4931506849315, 1016.5058359917433, 1025.1391732289333, ...]
#   },
#   "fixed_costs": {
#     "": [172.4529580009032, 161.32696071052234, 172.4529580009032, ...]
#   },
#   "": [836.0401926840283, 855.1788752812209, 852.68621522803, ...]
# }
```

We can improve the nested dictionary by flattening it out.

```python
def _flatten_gen(d: dict, prefix="."):
    for key, value in d.items():
        if isinstance(value, dict):
            yield from _flatten_gen(value, prefix + key)
        else:
            yield prefix + key, value


def flatten(d: dict):
    """Flatten a nested dictionary"""
    return dict(_flatten_gen(d))
```

Each line item and subline item is delimited by a period.

```python
flat_pro_forma = flatten(pro_forma)
print(json.dumps(flat_pro_forma, indent=2))
# {
#   ".revenue": [1008.4931506849315, 1016.5058359917433, 1025.1391732289333, ...]
#   ".fixed_costs": [172.4529580009032, 161.32696071052234, 172.4529580009032, ...]
#   ".": [836.0401926840283, 855.1788752812209, 852.68621522803, ...]
# }
```

Incorporating these pretty outputs into the base objects would be helfpul, but I'm not going to spend the time to do that here.

The pretty pro forma dictionary makes exporting results to `pandas` easy. From there, you could easily copy to the clipboard or save to Excel.

```python
import pandas as pd

df = pd.DataFrame(flat_pro_forma, index=[p[1] for p in periods]).T
print(df)
#                2020-01-31   2020-02-29   2020-03-31  ...   2020-10-31   2020-11-30   2020-12-31
# .revenue      1008.493151  1016.505836  1025.139173  ...  1086.774661  1095.707056  1105.013061
# .fixed_costs   172.452958   161.326961   172.452958  ...   177.085398   171.372966   177.085398
# .              836.040193   855.178875   852.686215  ...   909.689263   924.334090   927.927663

# [3 rows x 12 columns]

# df.to_clipboard()
# df.to_excel('pro_forma.xlsx')
```

## Saving and loading models

We haven't exploited the power of Pydantic yet. In fact, nothing so far really requires Pydantic. However, it gives us a nice way to separate our data from our models. We can save our data somewhere else, load it into our model when we want to evaluate some metric, and then serialize it back out to some storage with any updates. It also gives third-parties a way to update the data independently of the model.

```python
# write the model to a file
with open("my_company.json", "w") as f:
    f.write(statement.model_dump_json())

# mimic some external system that updates the data for higher expected revenue growth
with open("my_company.json", "r") as f:
    data = json.load(f)
    data["revenue"]["growth_rate"] = 0.15

with open("my_company.json", "w") as f:
    json.dump(data, f)

# create new model from updateddata
with open("my_company.json", "r") as f:
    new_statement = OperatingStatement.model_validate_json(f.read())

assert new_statement.revenue.growth_rate == 0.15
```

Difficulty interacting with external data sources is one of the biggest challenges with Excel. Pydantic makes it significantly easier to deal with serializing to and validating data from external sources. In some cases you may need to create custom serializers or validators, but that it's much more flexible than Power Query or VBA.

## Final thoughts

This is a very basic example. It's not at all clear that it scales well to larger or more complex models. The hierarchical structure around the financial statements works well, but without better ways to explore the structure visually it would be difficult to figure out how a complex model is constructed (not that stepping through someone else's workbook is any easier). Ripe for improvements, and perhaps a different approach altogether.

The full code for this post can be found [here](https://gist.github.com/jrdnh/7621762c23157bb093ca629a0491201e).


[^1]: Specifically I mean Excel. Sheets and Numbers are not serious contenders in finance.
[^2]: I'm sure they're out there. I just haven't found them yet.