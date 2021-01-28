Handling country codes
**********************
:tags: pandas, python, research, software

In research with global scope and country- or country-group resolution, it's common to handle data with one or more [1]_ dimension(s) identifying a country (countries) for each observation.
Problems can arise when inconsistent identifiers—“United States” vs. “United States of America”—are used to label this dimension, either across different data sets, or within one data set.

The best precaution against these problems is to convert idiosyncratic identifiers to short, standard ones, as soon as possible.
`ISO 3166 alpha-2 or alpha-3 codes <https://en.wikipedia.org/wiki/List_of_ISO_3166_country_codes>`_ (``CA`` or ``CAN`` for Canada) are a natural choice for standard identifiers. [2]_

In this post, I show a way to do this in a few lines of performant Python code, using some common packages and tools.
The sections are titled with some general points of advice that also apply to other programming tasks.

.. contents::
   :local:
   :backlinks: none

1. Find and use well-tested tools
=================================

In this example, we'll want to use `pycountry <https://pypi.org/project/pycountry/>`_, a package that has existed since 2008 and, per its documentation:

    …provides the ISO databases for the standards:

    - 3166 Countries

    The package includes a copy from Debian’s pkg-isocodes and makes the data accessible through a Python API.

It's easy to use and fast:

.. ipython:: python

   from pycountry import countries

   # Get a country by its code
   countries.get(alpha_2="CA")

   # Lookup on any field: alpha_2, alpha_3, or name:
   countries.lookup("Canada")

   # Fuzzy search returns a list() of results
   countries.search_fuzzy("Korea")

``pycountry`` is downloaded about 56,000 times *every day*:

.. image:: https://img.shields.io/pypi/dd/pycountry
   :alt: Rate of PyPI downloads

As with ``pandas``, below, heavy usage is both an indicator of and contributor to quality.
Hypothetically, we could copy the ISO 3166 table from Wikipedia into a CSV file and write some code to query it.
But there's no point in doing this.
The result will certainly be slower, more likely to contain bugs, and have fewer features than ``pycountry``.

2. Understand the standard library
==================================

Some example data:

.. ipython:: python

    import pandas as pd

    data = pd.DataFrame([
        ["Austria", 2021, 1.1],
        ["Bosnia", 2021, 2.2],
        ["Canada", 2021, 3.3],
        ["UK", 2021, 4.4],
        ["UK", 2022, 5.5],
    ], columns=["country", "year", "value"])
    data

What is in the "country" column/dimension?

- “Canada” and “Austria” are direct matches for the ‘name’ field of the ISO 3166-1 database.
- “Bosnia” is only a partial match for the name “Bosnia and Herzegovina”.
- “UK” is an idiosyncratic identifier: the ISO 3166-1 codes for the United Kingdom are ``GB`` and ``GBR``.

We create a mapping with the idiosyncrasies of this data set:

.. ipython:: python

    country_map = {
        "Bosnia": "Bosnia and Herzegovina",
        "UK": "United Kingdom",
    }

This allows us to look up a correct value for an incorrect one:

.. ipython:: python

   country_map["UK"]

``dict.get()``
--------------

Do we need to include “Austria” and “Canada” in this dictionary?
No.
The standard `dict.get() <https://docs.python.org/3/library/stdtypes.html#dict.Get>`_ method allows a `default` argument:

.. ipython:: python

    for name in data["country"]:
        print(country_map.get(name, name))

When ``name`` is “Canada”, the lookup fails; “Canada” is not a key in the dictionary.
The `default` argument is returned: “Canada”.
This is a **no-op** or **pass-through**; it does nothing.
Code that passes through some values while altering others will always perform better than code that does comparisons (``if name == "UK": …``).

Undersetanding that ``get()`` takes a `default` argument allows us to make ``country_map`` **parsimonious** and easy to read.
It includes only important information (the incorrect labels appearing in this data set) and does not obscure them with unnecessary information.


Decorate with ``functools.lru_cache()``
---------------------------------------

A real data set, unlike our example ``data``, will contain many rows, sometimes with the same incorrect identifiers repeated many times.
To speed up any operation we want to do with this data, we can use `lru_cache() <https://docs.python.org/3/library/functools.html#functools.lru_cache>`_, [3]_ a function from the ``functools`` module of Python's standard library.

Go read the documentation! I'll wait.

We use this to **decorate** a function.
It matches return values to input, and avoids running the (potentially slow) function when it sees a value for which a result has already been computed:

.. NB use the "In […]:" syntax here to prevent the ipython.sphinxext code from eating the @lru_cache()

.. ipython::

   In [1]: from functools import lru_cache

   In [2]: @lru_cache()
      ...: def fix_name(value):
      ...:     return country_map.get(value, value)

   In [3]: for name in data["country"]:
      ...:     print(fix_name(name))

We can inspect the cache that is created as ``fix_name()`` is called multiple times:

.. ipython:: python

    fix_name.cache_info()

``currsize=4`` tells us that 4 values have been cached over 5 calls.
``hits=1`` tells us that the second occurrence of “UK” was handled by returning a cached value, instead of running the body of ``fix_name()`` again with the same input. [4]_

3. Find and use optimized code
==============================

Our ``data`` is a `pandas <https://pandas.pydata.org/docs/reference/index.html>`_ data structure.
Pandas is a very widely used library with many users and contributors. [5]_
For these reasons, it is internally very sophisticated; many common operations—indeed, almost all operations that most of us will perform in daily use—have been optimized, some using low-level C code, for speed and memory performance.

The simple upshot is that **it is almost never necessary to write Python loops** (``for …:`` and ``while …:``) when using pandas.
If you *do* find yourself writing loops, it is likely that you can instead find and use existing, optimized functionality in pandas.

I strongly recommend doing this: your code will be both simpler and more performant.

Above, we looped over the values in the "country" column of ``data``.
(Remember: each column of a ``pandas.DataFrame`` is a ``pandas.Series``.)
The feature we should use instead is `pandas.Series.apply() <https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.Series.apply.html>`_.
(Again: go read the documentation, all of it.)

This looks like:

.. ipython:: python

   data["country"].apply(fix_name)

Pandas takes care of calling ``fix_name()`` many times, once for each value in this column, and assembling the result into a new series.
It internally speeds up and, where possible, parallelizes these calls: the point is *we don't need to think about how it does this*.

Also, because we've memoized with ``lru_cache()``, after a certain point each call becomes as quick as a dictionary lookup, no matter how expensive ``fix_name()`` is:

.. ipython:: python

    fix_name.cache_info()

More cache hits have occurred.

Wrapping up
===========

Now we finally bring in ``pycountry``:

.. ipython::

   In [1]: @lru_cache()
      ...: def code_for_name(value):
      ...:     return countries.get(name=country_map.get(value, value)).alpha_3

This function first uses ``country_map`` to correct data set idiosyncrasies, then uses ``pycountry.get()`` to look up a record using the ‘name’ field.
Finally, it returns the ‘alpha_3’ code from the record.

We ``apply()`` this new function to the ‘country’ column:

.. ipython:: python

    data["country"].apply(code_for_name)

    code_for_name.cache_info()

And we can even update the data, discarding [6]_ the full names:

.. ipython:: python

   data.assign(country=data["country"].apply(code_for_name))


Concluding thoughts
-------------------

The three points of advice here:

1. Find and use well-tested tools.
2. Understand the standard library.
3. Find and use optimized code.

…fit two themes.

One is that **time spent developing fluency with basic tasks pays dividends**.
For instance, we practice taking derivatives and performing integrals over and over, so that we can do them automatically and combine steps in the course of tackling more complex mathematical problems.
Writing code for research is no different: we want to minimize the space and attention taken up by routine tasks, so that the real content—the methods and theory we are trying to implement—stands out prominently.

The second is that **we follow paths blazed and walked by others**.
While our codes may investigate a new question in a novel way, the elemental building blocks of that code are shared with many others.
Almost always, someone has put in work to create an elegant and performant way to do these atomic tasks.
We ought to discover, understand, and use the tools they provide us.

----

**Footnotes**

.. [1] Trade data, for instance, can have two country dimensions: origin and destination.
.. [2] These are sometimes called “ISO codes.”
   **Do not** do this.

   The ISO publishes (and ``pycountry`` supports) standard code lists for languages, currencies, and many other concepts.
   As soon as your software needs to handle data with both country and ≥1 of these other concepts—for instance, measures of economic activity denoted in local currency—the dimension label “ISO code” becomes ambiguous.

   Label dimensions with the *concept represented*, not the *kind of representation*.

   As well, **do not** invent new codes.

.. [3] `cache() <https://docs.python.org/3/library/functools.html#functools.cache>`_ is also available from Python 3.9.

.. [4] The code examples in this post are all fairly fast to begin with, so the gain from using ``lru_cache()`` is not very large.
   However, this is a good pattern to learn and apply when the repeated function (here ``country_code``) has some slow steps, or the data is big.

.. [5] There have been `over 25,000 commits <https://github.com/pandas-dev/pandas>`_ modifying the ``pandas`` code!

.. [6] Exercise: with what we've covered here, write a simple function that restores ‘country’ to the ‘name’ field of each record.

   Again following the principle of parsimony, it is better to keep only the short, ISO 3166 alpha codes in ‘internal’ data throughout most of a program.
   Restore the ‘country’ dimension to the full name(s) only if and when necessary, e.g. to make output more intelligible to a user.
