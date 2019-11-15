## Context, disclaimers, and assumptions
* SQL is the most important language in Data Science, but...
* ...when correctness _matters_ SQL is unsafe
* ...when iteration is a better idiom SQL is awkward
* Dropping down to pure Python...
* ...opens the door for a more expressive interface
* ...allows you to leave behind tests
* ...increases safety at the cost of performance


## Example: Date spans
* A span is an interval
* Each span is characterized by a lower bound and upper bound
* You're either in the interval or you're not


## Data
* Imagine we have the following data...
```
id    user_id    ds_begin    ds_end
1     1          2019-01-01  2019-01-08
2     1          2019-01-08  2019-01-15
3     1          2019-01-16  2019-01-17
```
* ...and we want to aggregate each of the intervals

## SQL
* SQL probably gets you performance out of the box
* You can write brittle `case when` statements
* You can also write fancy SQL (recursive CTE, Oracle `connect by`)
* Unless you have tooling, you can't write tests
* If you have different definitions, you reuse code by copying and pasting

## Type behavior: Instantiation
* Reasonable to instantiate with two `dt.datetime` values
```
>>> class Span:
...    def __init__(self, begin, end):
...        self.begin = begin
...	   self.end = end
>>> span = Span(begin=dt.date(2019, 1, 1), end=dt.date(2019, 1, 7))
```

## Type behavior: Intersection
* Do spans overlap?
```
>>> span1 = Span(begin=dt.date(2019, 1, 1), end=dt.date(2019, 1, 7))
>>> span2 = Span(begin=dt.date(2019, 1, 7), end=dt.date(2019, 1, 8))
>>> span3 = Span(begin=dt.date(2019, 1, 9), end=dt.date(2019, 1, 14))
>>> span1.intersects(span2)
True
>>> span2.intersects(span3)
False
```

## Type behavior: Adjoinment
* Do spans adjoin?
```
>>> span1 = Span(begin=dt.date(2019, 1, 1), end=dt.date(2019, 1, 7))
>>> span2 = Span(begin=dt.date(2019, 1, 7), end=dt.date(2019, 1, 8))
>>> span3 = Span(begin=dt.date(2019, 1, 9), end=dt.date(2019, 1, 14))
>>> span1.adjoins(span2)
False
>>> span2.adjoins(span3)
True
```

## Type behavior: Addition
```
class DateSpan:
    # previous stuff
    def __add__(self, other):
        if not self.adjoins(other) or self.intersects(other):
	    raise ValueError
	lower_bound = min(self.begin, other.begin)
	upper_boud = max(self.end, other.end)
	return DateSpan(begin=lower_bound, end=upper_bound)
```


## My workflow
* Think of all the things that would annoy me if I were inlining business logic everywhere

## Type-powered rollup method
```
def roll_up_spans(spans, acc=None):
    acc = acc if acc is not None else []

    if spans == []:
        return acc

    current_span = spans[0]
    if acc == []:
        return roll_up_spans(current_span[1:], acc=[current_span])

    last_span = acc[-1]
    if last_span.adjoins(current_span) or last_span.intersects(current_span):
        acc = acc + [current_span + last_span]
    else:
        acc = acc + [current_span]
    return roll_up_spans(current_spans[1:], acc=acc)
```


## This might be worse than SQL

## This makes it better: Tests
* The testing story in SQL is not good
* Tests of the type are high-leverage
* Tests of the pipeline task are simpler

## This makes it better: Code reuse
* The code reuse story in SQL is not good
* Writing down a useful type is high-leverage

## Performance
* In pipelines I expect SQL to be faster
* I expect performance gains in Python to be costly
* Optimize the method, e.g. `line_profiler`, Cython, Numba
* Parallelize on the worker, e.g. `concurrent.futures`, `asyncio`
* Parallelize on a cluster, e.g. Dask, Spark

## Usability and maintainability
* Use of these patterns seems unlikely within Data Science teams
* Depending on the Data Engineering team, this might also be uncomfortable
* Feasibility depends on who you've hired and business conditions
* But have you _seen_ pipelines that are purely developed in SQL?

## An intermediate possibility
* Pandas!
* It is totally reasonable to assume knowledge of the `pandas.DataFrame` API
* My untested hunch is Dask or Spark DataFrames is the least bad end state

## Inspirations
* `Fluent Python`
* `A Philosophy of Software Design`
* Project Euler + Go
