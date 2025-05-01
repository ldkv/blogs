---
layout: post
title: Tuples filter on multi columns with Django ORM
date: 2025-05-01 00:19 +0200
---

## Introduction

As a daily Django developer, I often run into the need to filter on multiple columns with a tuple of values. For a long time, there was no built-in support in Django ORM for this use case, and developers have to come up with workarounds to achieve the desired result.

In this post, I will discuss some of the common workarounds, then present a solution that was "shadow" released together with [Django 5.2](https://docs.djangoproject.com/en/5.2/releases/5.2/), but has not been made public yet.

I will also dedicate a section to run some benchmarks (with code) to compare the performance of the different solutions using a PostgreSQL database.

## Problem statement

Let's take a simple example to better illustrate the use case, given the following model:

```python
class People(models.Model):
    first_name = models.CharField(max_length=255)
    last_name = models.CharField(max_length=255)
    age = models.IntegerField()
```

We want to calculate the average age of some people from the table, based on an input list of first and last names. Ideally, the raw query should look like this:

```sql
SELECT AVG(age) FROM people
WHERE (first_name, last_name) IN (
    ('John', 'Doe'),
    ('Jane', 'Doe'),
    ...,
    ('John', 'Smith')
);
```

## Solutions

### 1. Build the filter conditions manually

This is the most common workaround, using the `Q` object to generate the filter conditions.

```python
from django.db.models import Q

filter_conditions = Q()
for first_name, last_name in input_names:
    filter_conditions |= Q(first_name=first_name, last_name=last_name)

queryset = People.objects.filter(filter_conditions)
average_age = queryset.aggregate(Avg('age'))['age__avg']
```

Which will generate the following SQL query:

```sql
SELECT AVG(age) FROM people
WHERE (first_name = 'John' AND last_name = 'Doe')
    OR (first_name = 'Jane' AND last_name = 'Doe')
    OR ...
    OR (first_name = 'John' AND last_name = 'Smith')
;
```

At first glance, some (myself included) might assume that this is quite different from the desired query, and the performance might be worse.

However, for [PostgreSQL engine](https://www.postgresql.org/docs/current/functions-comparisons.html#FUNCTIONS-COMPARISONS-IN-SCALAR), the `IN` clause is translated to a `WHERE` clause with multiple `OR` conditions, so our generated SQL query is actually closer to what will be executed by the database engine.

Even though this solution is valid on the database level, it still has some drawbacks:

-   Additional code to generate the `Q` object and harder to extend, for example when there are more than two columns to filter on.
-   The generation of the `Q` object will consume more memory and time on the Python side. In most cases, this is negligible, but it might become significant when the input list is much larger.
-   The generated SQL query is larger, and will take longer to send to the database.

### 2. Custom Tuple function

This [workaround](https://stackoverflow.com/a/75567203/21333661) involves creating a custom function that will generate the tuple field via an alias, then use it in the `filter` method with normal `in` lookup.

```python
from django.db.models import Func, Value

def ValueTuple(items: list[tuple]):
    return tuple(Value(i) for i in items)

class Tuple(Func):
    function = ""

queryset = (
    People.objects
    .alias(a=Tuple("first_name", "last_name"))
    .filter(a__in=ValueTuple(input_names)
)
```

The generated SQL query should be identical to the desired query. However, in addition to the same drawbacks as the previous solution, another issue is that it only works with `psycopg2` (not `psycopg3` due to this [change](https://www.psycopg.org/psycopg3/docs/basic/from_pg2.html#you-cannot-use-in-s-with-a-tuple)), which is no longer in active development.

All other flavors of this workaround (such as `RawSQL`) also share the same problems, so I won't cover them here.

### 3. Hidden feature in Django 5.2

Django 5.2 was released recently and introduced the long-awaited support for [Composite Primary Keys](https://docs.djangoproject.com/en/5.2/releases/5.2/#composite-primary-keys). A new internal module was added as part of this feature, [`django.db.models.fields.tuple_lookups`](https://github.com/django/django/blob/0f5dd0dff3049189a3fe71a62670b746543335d5/django/db/models/fields/tuple_lookups.py#L280), which includes built-in support for tuples with `IN` lookup.

Be aware that this internal module has not been made public in the Django API or documentation yet. This means that it might not be stable and is subject to change in future versions of Django, so **use it at your own risk**.

The usage is straightforward and simple:

```python
from django.db.models.fields.tuple_lookups import Tuple, TupleIn

queryset = People.objects.filter(TupleIn(Tuple("first_name", "last_name"), input_names))
```

The generated SQL query should be identical to the desired query.
