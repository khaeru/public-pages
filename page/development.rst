Research software development
*****************************

:status: hidden

Contents:

.. contents::
   :local:
   :backlinks: none


General
=======
- RTFD.

Continuous integration
======================

AppVeyor
   …

Codecov
   Preferred to coveralls.

GitHub Actions
   …

Travis
   …


Editors
=======
`Atom <https://atom.io/>`_.


Python
======

`click <https://click.palletsprojects.com/en/7.x/>`_
   - Write command-line interfaces (CLI).
     Every step of analysis for preparing a paper should be condensed to invoking a CLI command.
   - Use ``print()`` for output.

Logging
   - Avoid ``print()`` except when debugging.
   - Use the following at the top of every file:

     .. code-block:: python

        import logging

        log = logging.getLogger(__name__)

     The use of ``__name__`` ensures that messages are attached to ``mypackage.foo.bar`` or whichever submodule, and can be filtered by all-purpose code.

``pandas``
   - Chain methods:

     .. code-block:: python

        df = df.unstack('foo') \
               .rename_axis(bar) \
               .reset_index()

     - Align on the “``.``”.
     - Use ``\`` for line continuations.
     - Write functions and then use `pipe() <https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.pipe.html>`_ and `apply() <https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.apply.html>`_ to inline them into chains.

     Most data manipulation routines can be reduced to one or a few chains.
     Using pandas' built-in methods and chaining allow its optimizations to improve performance; this will almost always be more performant than writing new routines.
     Loops are almost never necessary.

``pip``
   - ``pip install --editable .`` any package you are actively working on.
     This saves the mental effort of re-installing packages when changing branches or making small modifications.

``pytest``
   - Use `pytest-watch <https://pypi.org/project/pytest-watch/>`_: leave it running in a terminal as you code.

``setuptools``
   - Put all information in setup.cfg.
   - setup.py should be only 2 lines.

``setuptools-scm``
   Preferred to versioneer.


Version control
===============
``git``
   - Memorize and apply the `7 rules of a great git commit message <https://chris.beams.io/posts/git-commit/#seven-rules>`_.
   - Rebase instead of merging.

GitHub
   - Also use the 7 rules (above) for pull request (PR) titles.
