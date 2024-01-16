+++
title = 'Create a Multifamily Financial Model in Python'
date = 2024-01-15T11:15:39-05:00
draft = false
+++

*This post creates a model that mirrors the Excel workbook [here](https://gist.github.com/jrdnh/fb26772b430283344d887916b4609979).*

This post discusses a simple financial model for a multifamily property. The objective is to create an interactive financial model with structured hierarchy for operating line items and sub-items. For example, net operating income (NOI) should be accessed with:
```python
noi(from_date=date(yyyy, mm, dd), to_date=date(yyyy, mm, dd))
```
Sub-items, effective gross income (EGI) and operating expenses (opex) in the case of NOI, should similarly be accessible all the way down the basic operating assumptions.
```python
noi.egi(from_date=date(yyyy, mm, dd), to_date=date(yyyy, mm, dd))
noi.opex(from_date=date(yyyy, mm, dd), to_date=date(yyyy, mm, dd))

# net operating income > effective gross income > potential gross rent > average monthly rent per square foot
noi.egi.gross_potential_rent.avg_monthly_rent_psf(from_date=date(yyyy, mm, dd), to_date=date(yyyy, mm, dd))
```
Line items should accept any arbitrary `from_date` and `to_date`. Annual, quarterly, monthly, or intra-period dates might be required depending on the context of analysis.

Finally, the model needs to be able to match the equivalent Excel model. For example, we want the ability to capture the effect of discrete monthly changes (to mirror an Excel model where columns represent months) even though line item functions should accept arbitrary date ranges.

This post discusses some of the successes and challenges in meeting these modeling objectives, it isn't a definitive guide. It also does not step through every step of creating this model.

* [Companion file setup](#companion-file-setup)
* [Model construction](#model-construction)
    * [Model overview](#model-overview)
    * [Line item classes](#line-item-classes)
    * [Year fractions](#year-fractions)
* [Model usage](#model-usage)
* [Challenges with this approach](#challenges-with-this-approach)
    * [Performance and profiling details](#performance-and-profiling-details)


## Companion file setup

The three companion files for this post are in [this Gist](). The files are:
* `models.py` - Pydantic-compatible wrappers `relativedelta` similar to the ones in [this previous post](https://jrdnh.github.io/posts/pydantic-with-third-party-types/), and a `FixedIntervalSeries` class that yields a series of dates at a fixed interval similar to the class in [this previous post](https://jrdnh.github.io/posts/basic-financial-modeling-with-python/#refactor-to-make-it-a-bit-more-generic).
* `utils.py` - Utility functions for working with dates, `Iterators`, and printing.
* `companion.py` - The companion script that creates the model.

The companion files were created with Python 3.12 and use `pydantic==2.5.2` and `python-dateutil==2.8.2`. Other versions may work but have not been tested.

## Model construction
### Model overview

The model is structured as tree of financial line items. The nodes are classes. Class attributes are annotated with their types.

```
NetOperatingIncome
├── EffectiveGrossIncome
│   ├── AvgVacancyRate
│   │   └── vacancy_rates: list[float]
│   └── GrossPotentialRent
│       ├── vacancy: method
│       ├── sf: int
│       └── AvgMonthlyRentPSF
│           ├── initial_rent_psf: float
│           └── rent_growth_rate: float
└── TotalExpenses
    ├── OperatingExpenses
    │   ├── units: int
    │   ├── initial_opex_pu: float
    │   └── opex_growth_rate: float 
    ├── RealEstateTaxes
    │   ├── sf: int
    │   ├── monthly_re_tax_psf: float
    │   └── ret_growth_rate: float 
    └── ReplacementReserves
        ├── units: int
        ├── annual_reserves_pu: float
        └── rr_growth_rate: float 
```

Each class is a callable that takes `from_dt: date` and `to_dt: date` arguments and returns the total (or in some cases average) line item value from but excluding the first date to and including the second date. This argument convention aligns with using consecutive `(start date, end date]` periods.

## Line item classes

Building line items as class instances with subitems as attributes gives the model structure. Subitems are accessible through regular dot notation. Each line items is a separate class, although `OperatingExpenses`, `RealEstateTaxes`, and `ReplacementReserves` are essentially the same classes with different property names. This type of periodic growth is a common pattern, so it might make sense to abstract these lines into a separate class.

Not shown in the overview diagram above is that each line item class is a subclass of `FixedIntervalSeries` from the `models.py` file. The `FixedIntervalSeries` class takes `ref_date: date` and `freq: RelativeDelta` constructor parameters. It has a single method `periods` that returns a generator yielding a series of dates with an even offset defined by the `freq` property.

```python
class FixedIntervalSeries(BaseModel):
    ref_date: date
    freq: RelativeDelta

    def periods(self):
        index = 0
        curr_date = self.ref_date
        while True:
            yield (curr_date + self.freq * index)
            index += 1
```

It can be used as follows to create a series of evenly spaced dates.

```python
from datetime import date
from models import FixedIntervalSeries, RelativeDelta

series = FixedIntervalSeries(ref_date=date(2020, 1, 1), freq=relativedelta(months=1))
series_generator = series.periods()
print([next(series_generator) for _ in range(3)])
[datetime.date(2020, 1, 1), datetime.date(2020, 2, 1), datetime.date(2020, 3, 1)]
```

Or using `itertools.pairwise`, create series of `(start_date, end_date)` tuples defining both edges of a period.

```python
from itertools import islice, pairwise

list(islice(pairwise(series.periods()), 2))
[(datetime.date(2020, 1, 1), datetime.date(2020, 2, 1)), (datetime.date(2020, 2, 1), datetime.date(2020, 3, 1))]
```

The `periods` function provides the time events required to calculate discrete time interval values. Note that most of the `periods` functions in this model yield evenly spaced dates, but yielding the next eventful date (regardless of time interval) is the only requirement.

Looping over `periods`, in combination with `itertools.takewhile` and year fraction functions gives us a way to calculate values dependent on discrete intervals over arbitrary time periods. The annotated `RealEstateTaxes` class shows how you can calculate tax expense that increases a fixed percentage at fixed intervals. Initializing with `freq=RelativeDelta(years=1)` and `ret_growth_rate=0.05` would step up the expense by 5.0% on the anniversary of each `ref_date`.

```python
class RealEstateTaxes(FixedIntervalSeries):
    sf: int  # square feet
    monthly_re_tax_psf: float  # monthly real estate taxes per square foot
    ret_growth_rate: float  # annual real estate tax growth rate

    def __call__(self, from_dt: date, to_dt: date):
        """Real estate taxes from but excluding `from_dt` to and including `to_dt`."""
        # Generator over (period index, (period start date, period end date)), 
        # stopping when periods are past the requested date range
        periods = enumerate(takewhile(lambda p: p[0] < to_dt, pairwise(self.periods())))
        # Initialize values
        accumulated_amt = 0
        period_amt = self.monthly_re_tax_psf * self.sf * 12  # first period amount, annualized
        # Loop over each period and calculate the applicable tax expense
        for i, (per_start, per_end) in periods:
            # Since we want the expense for the first period to equal the initial `monthly_re_tax_psf`,
            # only increase the expense if i > 0
            if i > 0:
                period_amt = period_amt * (1 + self.ret_growth_rate * YF.monthly(per_start, per_end))
            # Add the expense for the portion of each period that overlaps with (from_dt, to_date)
            accumulated_amt += period_amt * max(YF.monthly(max(per_start, from_dt), min(per_end, to_dt)), 0)
        return accumulated_amt
```

This approach uses the actual dates in each period to determine tax expense for each period. It would be less code to just increase the tax expense each period with something like this.

```python
    def __call__(self, from_dt: date, to_dt: date):
        ...
        for (start, end) in periods:
            period_amt *= (1 + self.ret_growth_rate * YF.thrity360(start, end))
            ...
```

The issue with this approach is that it ties the starting value `monthly_re_tax_psf` to the interval duration `freq`. If periods do not have the same duration or you want to change the period frequency, the starting value no longer works.

### Year fractions

The `YF` class in `utils.py` holds three year fraction formulas: `actual360` for an actual/360 day count convention (`=YEAR(,,2)` in Excel), `thirty360` for a 30/360 convention (`=YEAR(,,0)` in Excel), and `monthly` (which does not have an Excel equivalent).

The `monthly` function returns one 1/12th for each whole calendar month and the pro rata portion of any stub months based on the actual number of days elapsed and the actual number of days in the month. It is similar to a 30/360 convention with a few changes.


<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg td{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  overflow:hidden;padding:10px 5px;word-break:normal;}
.tg th{border-color:black;border-style:solid;border-width:1px;font-family:Arial, sans-serif;font-size:14px;
  font-weight:normal;overflow:hidden;padding:10px 5px;word-break:normal;}
.tg .tg-0pky{border-color:inherit;text-align:left;vertical-align:top}
</style>
<table class="tg">
<thead>
  <tr>
    <th class="tg-0pky"></th>
    <th class="tg-0pky">30/360</th>
    <th class="tg-0pky"><span style="color:#905;background-color:#DDD">`YF.monthly`</span></th>
  </tr>
</thead>
<tbody>
  <tr>
    <td class="tg-0pky">Whole non-calendar months<br><span style="font-weight:400;font-style:normal;text-decoration:none">(e.g. 2020-06-15 to 2020-07-15)</span><br></td>
    <td class="tg-0pky">Equals 1/12th of a year<br><br><span style="color:#905;background-color:#DDD">=YEARFRAC("2020-06-15", "2020-07-15", 0)</span><br>0.08333333333333333```</td>
    <td class="tg-0pky">Depends whether the stub starting month and ending month <br>have the same number of days<br><span style="color:#905;background-color:#DDD">&gt;&gt;&gt; YF.monthly(date(2020,6,15), date(2020,7,15))</span><br>0.08198924731182795</td>
  </tr>
  <tr>
    <td class="tg-0pky">Month end to month end periods</td>
    <td class="tg-0pky">Depends on whether the month is February or March<br><span style="color:#905;background-color:#DDD">=YEARFRAC("2020-01-31", "2020-02-29", 0)</span><br>0.0805555555555556<br><span style="color:#905;background-color:#DDD">=YEARFRAC("2020-02-29", "2020-03-31", 0)</span><br>0.0861111111111111</td>
    <td class="tg-0pky">Always equals 1/12th of a year<br><br><span style="color:#905;background-color:#DDD">&gt;&gt;&gt; YF.monthly(date(2020,1,31), date(2020,2,29))</span><br>0.08333333333333333<br></td>
  </tr>
  <tr>
    <td class="tg-0pky">Last day of a month with 31 days</td>
    <td class="tg-0pky">Equals zero<br><span style="color:#905;background-color:#DDD">=YEARFRAC("2020-07-31", "2020-07-31", 0)</span><br>0.0</td>
    <td class="tg-0pky">Equals 1/31st of a month [i.e. (1 / 31) / 12]<br><span style="color:#905;background-color:#DDD">&gt;&gt;&gt; YF.monthly(date(2020,7,30), date(2020,7,31))</span><br>0.0026881720430107525</td>
  </tr>
</tbody>
</table>

`YF.monthly` allows you to create discrete, even calendar month accruals and growth rates from two `date`s. While not necessarily required to develop a good model, it is helpful when trying to tie back to an Excel model where users model monthly columns of data with `amount or rate/12`.


## Model usage

This model is fully Pydantic-compatible. You can build the model from the JSON assumptions hosted in [this Gist](https://gist.githubusercontent.com/jrdnh/377f13e0ed0e6ac975b5d36156dd27f5/raw/44bb448d60ff6aae9cc5f5d9472edf9e0c0b85d3/noi.json). Assuming you are in the same directory has the companion file `companion.py` and have the `pydantic` and `python-dateutil` dependencies installed, you can create an instance of the model with the commands below.

```python
import json
import urllib.request
from companion import NetOperatingIncome

url = 'https://gist.githubusercontent.com/jrdnh/377f13e0ed0e6ac975b5d36156dd27f5/raw/44bb448d60ff6aae9cc5f5d9472edf9e0c0b85d3/noi.json'
with urllib.request.urlopen(url) as response:
   noi = NetOperatingIncome.model_validate_json(response.read())
```

Now you have a full model of financial projetions for this apartment building!

Try getting net operating income over the first year.
```python
from datetime import date

noi(date(2019, 12, 31), date(2020, 12, 31))
# 4667040.0
```
Get the average monthly rent per square foot between two random dates (for example, 2 June 2021 to 27 September 2023).
```python
noi.effective_gross_income.gross_potential_rent.avg_monthly_rent_psf(date(2021, 6, 2), date(2023, 9, 27))
# 3.292631202107785
```

Line items are `FixedIntervalSeries` instances. If we want to display the full pro forma for all line items on a monthly basis, we can recursively iterate over each attribute and try calling it with a series of monthly periods. The `utils.py` helper `field_values` creates a nested dictionary of line items with their values, and `flatten` flatten the dictionary.

```python
from itertools import pairwise
from utils import field_values, flatten
from models import RelativeDelta

ten_years_monthly = list(pairwise((date(2019, 12, 31) + RelativeDelta(months=1) * i for i in range(121))))
pro_forma = flatten(field_values(noi, ten_years_monthly))
print(json.dumps(pro_forma, indent=2))
# json output
```

Pandas displays tables nicely. It is also easy to copy DataFrames to the clipboard or write directly to a `.xlsx` file. Make sure you have `pandas` installed before running the following code.

```python
import pandas as pd

df = pd.DataFrame(pro_forma, index=[p[1] for p in ten_years_monthly]).T
# df.to_clipboard()
# df.to_excel('pro_forma.xlsx')
print(df)
#                                                     2020-01-31  2020-02-29  2020-03-31  ...     2029-10-31     2029-11-30     2029-12-31
# .effective_gross_income.avg_vacancy_rate                  0.10        0.10        0.10  ...       0.050000       0.050000       0.050000
# .effective_gross_income.gross_potential_rent.av...        3.16        3.16        3.16  ...       3.776493       3.776493       3.776493
# .effective_gross_income.gross_potential_rent         568800.00   568800.00   568800.00  ...  679768.653032  679768.653032  679768.653032
# .effective_gross_income                              511920.00   511920.00   511920.00  ...  645780.220381  645780.220381  645780.220381
# .total_expenses.operating_expenses                   -63250.00   -63250.00   -63250.00  ...  -75589.604965  -75589.604965  -75589.604965
# .total_expenses.real_estate_taxes                    -54000.00   -54000.00   -54000.00  ...  -64534.998706  -64534.998706  -64534.998706
# .total_expenses.replacement_reserves                  -5750.00    -5750.00    -5750.00  ...   -6871.782270   -6871.782270   -6871.782270
# .total_expenses                                     -123000.00  -123000.00  -123000.00  ... -146996.385941 -146996.385941 -146996.385941
# .                                                    388920.00   388920.00   388920.00  ...  498783.834440  498783.834440  498783.834440

# [9 rows x 120 columns]
```

## Challenges with this approach

* **Sibling dependencies:** You might have noticed that vacancy is modeled using a method of `EffectiveGrossIncome` instead of as a custom class. That's because it relies on gross potential rent which is a sibling in the hierarchy (vacany expense = gross potential rent * vacancy rate). For more complicated leasing arrangements such as net leases, reimbursement lines in the revenue section rely on lines buried all they way down in the expense section of the statement. There dependencies down the tree work well, but dependencies up and across the tree are difficult to navigate.
* **Assumption references:** If you [look](https://gist.githubusercontent.com/jrdnh/377f13e0ed0e6ac975b5d36156dd27f5/raw/44bb448d60ff6aae9cc5f5d9472edf9e0c0b85d3/noi.json) at the serialized assumptions for this model you'll see a lot of the same values. In this case, all the `ref_date`s are the same analysis start date of December 12, 2019. If you wanted to change the analysis start date, you'd have to update the assumption in many places.
* **Performance:** The `periods` date generators are inefficient. Calculating a full 10-year schedule of monthly values for all line items takes more than a third of second. `periods` is called almost 25,000 times. Practical models generally have mid to high hundreds of line items which implies this model is too slow to be useful. `periods` is a good candidate for caching though, both within subclasses and potentiall across subclasses, since the same arguments are called repeatedly and generator outputs are small. A future post will explore caching `periods`.


### Performance and profiling details

Time to create a full 10-year schedule of monthly values.
```python
from time import perf_counter

def duration():
    start = perf_counter()
    field_values(noi, ten_years_monthly)
    print((perf_counter() - start) * 1000, 'ms')

duration()
# 370.89961499441415 ms
```

Key profiling results.
```python
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()
field_values(noi, ten_years_monthly)
profiler.disable()

stats = pstats.Stats(profiler).sort_stats('tottime')
stats.print_stats(10)
#          1129336 function calls (1129297 primitive calls) in 0.870 seconds

#    Ordered by: internal time
#    List reduced from 53 to 10 due to restriction <10>

#    ncalls  tottime  percall  cumtime  percall filename:lineno(function)
#     21358    0.130    0.000    0.265    0.000 /.../dateutil/relativedelta.py:317(__add__)
#     21358    0.084    0.000    0.305    0.000 /.../dateutil/relativedelta.py:495(__mul__)
#     62662    0.083    0.000    0.152    0.000 /.../python3.12/calendar.py:154(weekday)
#     21358    0.063    0.000    0.222    0.000 /.../dateutil/relativedelta.py:105(__init__)
#     42716    0.056    0.000    0.102    0.000 {built-in method builtins.any}
#     20652    0.043    0.000    0.161    0.000 /.../utils.py:50(monthly)
#     62662    0.034    0.000    0.187    0.000 /.../calendar.py:161(monthrange)
#     62662    0.033    0.000    0.054    0.000 /.../python3.12/enum.py:709(__call__)
#     21358    0.030    0.000    0.049    0.000 /.../dateutil/relativedelta.py:231(_fix)
#     24598    0.030    0.000    0.613    0.000 /.../models.py:105(periods)
```
Most of the time is spent creating, multiplying, and adding `dateutil.relativedelta.relativedelta` objects inside the body of `FixedIntervalSeries.periods`. `FixedIntervalSeries.periods` is called almost 25,000 times (see last line).
