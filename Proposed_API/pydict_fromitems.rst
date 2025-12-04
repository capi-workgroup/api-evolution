++++++++++++++++++++++++++++++++++++++++++++++++
Add ``PyDict_FromItems()`` function to the C API
++++++++++++++++++++++++++++++++++++++++++++++++

Rationale
=========

For historical reasons, Python exposes private functions in its public C API.
These functions are not tested, not documented, can change or even be removed
anytime without notifying users. Private functions used by 3rd party projects
should be promoted to public functions to add tests, documentation and
backward compatibility warranties.

The private ``_PyDict_NewPresized()`` function is used by 11 projects of the
PyPI top 15,000. It has a *min_used* argument to preallocate enough items for
the created dictionary. Dictionaries can be optimized for Unicode keys, but
``_PyDict_NewPresized()`` doesn't support that.

Specification
=============

Add ``PyDict_FromItems()`` function to the C API::

    PyObject* PyDict_FromItems(
        PyObject *const *keys,
        Py_ssize_t keys_stride,
        PyObject *const *values,
        Py_ssize_t values_stride,
        Py_ssize_t length)

Usage:

* ``PyDict_FromItems(items, 2, items + 1, 2, length)``: *items* is an array of
  keys and values (``key1, value1, key2, value2, ..., keyN, valueN``).
* ``PyDict_FromItems(keys, 1, values, 1, length)``: *keys* is an array of keys
  and *values* is an array of values.
* ``PyDict_FromItems(keys, 1, &value, 0, length)``: *keys* is an array of keys
  and *value* is copied for all items.

On CPython, the function preallocates *length* items and scans *keys* to check
if keys are all Unicode strings.

Calls to the private ``PyObject* _PyStack_AsDict(PyObject *const *values,
PyObject *kwnames)`` function can be replaced with::

    PyDict_FromItems(&PyTuple_GET_ITEM(kwnames, 0), 1,
                     values, 1,
                     PyTuple_GET_SIZE(kwnames));


Benchmark
=========

`Benchmark
<https://github.com/python/cpython/pull/139963#issuecomment-3412864991>`__ on a
regular Python release build with Unicode keys and integer values:

===============   =======  =====================
 Benchmark        setitem  fromitems
===============   =======  =====================
 dict-1           278 ns   319 ns: 1.15x slower
 dict-10          2.69 us  2.48 us: 1.08x faster
 dict-100         29.6 us  24.4 us: 1.21x faster
 dict-1,000       301 us   244 us: 1.23x faster
 dict-10,000      3.51 ms  2.84 ms: 1.24x faster
 Geometric mean   (ref)    **1.12x faster**
===============   =======  =====================

`Benchmark
<https://github.com/python/cpython/pull/139963#issuecomment-3612465437>`__ on
**Free Threaded** build with Unicode keys and integer values:

==============   =======   =====================
Benchmark        setitem   fromitems
==============   =======   =====================
dict-1           467 ns    447 ns: 1.04x faster
dict-5           1.95 us   1.64 us: 1.19x faster
dict-10          3.71 us   3.22 us: 1.15x faster
dict-25          9.05 us   7.78 us: 1.16x faster
dict-50          17.9 us   15.2 us: 1.18x faster
dict-100         35.4 us   30.0 us: 1.18x faster
dict-500         164 us    137 us: 1.20x faster
dict-1,000       330 us    274 us: 1.20x faster
Geometric mean   (ref)     **1.16x faster**
==============   =======   =====================


PyPI projects using ``_PyDict_NewPresized()``
=============================================

* Nuitka (2.7.16)
* cython (3.1.4): see `issue <https://github.com/cython/cython/issues/7201>`__.
* frozendict (2.4.6)
* gevent (25.9.1)
* mercurial (7.1.1): see `issue
  <https://foss.heptapod.net/mercurial/mercurial-devel/-/issues/10029>`__.
* mypy (1.18.2)
* mypy_dev-1.19.0a2
* numba (0.62.0)
* orjson (3.11.3): see `comment
  <https://github.com/python/cpython/issues/139772#issuecomment-3393830209>`__.
* ormsgpack (1.10.0)
* serpyco_rs (1.17.1)


Rejected Ideas
==============

Add ``PyDict_NewPresized()`` function
-------------------------------------

Add ``PyDict_NewPresized()`` function to the C API::

    PyObject* PyDict_NewPresized(Py_ssize_t size, int unicode_keys)

If *unicode_keys* is true, optimize the created dictionary for keys which are
only Unicode strings.

The problem of this API is that it's too low-level: it exposes *unicode_keys*
optimization which is an implementation detail.

INADA-san wrote that most users either overestimate its effectiveness or don't
fully understand how it operates.


Add ``PyDict_FromKeysAndValues()`` and ``PyDict_FromItems()``
-------------------------------------------------------------

Add ``PyDict_FromKeysAndValues()`` and ``PyDict_FromItems()`` functions to the
C API::

    PyObject* PyDict_FromKeysAndValues(
        PyObject *const *keys,
        PyObject *const *values,
        Py_ssize_t length)

    PyObject* PyDict_FromItems(
        PyObject *const *items,
        Py_ssize_t length)

These functions are very close to the API proposed in this PEP, but have no
"stride" argument. They are less error-prone since *stride* arguments don't
exist and so cannot be misused.

But these functions are less generic and don't support ``values_stride=0`` to
reuse the same value for all items, or strides greater than ``2`` for more
complex arrays.

Add ``PyDict_MergeItems()`` function
------------------------------------

Add ``PyDict_MergeItems()`` function to the C API::

    PyObject* PyDict_MergeItems(
        PyObject *dict,
        PyObject *const *keys,
        Py_ssize_t keys_stride,
        PyObject *const *values,
        Py_ssize_t values_stride,
        Py_ssize_t length)

Create a new dictionary if *dict* is ``NULL``, or update an existing dictionary
otherwise.

Such function lacks an *override* argument to decide how to deal with
overridden keys on updating an existing dictionary.


Discussions
===========

* Issue `Add PyDict_NewPresized() function
  <https://github.com/python/cpython/issues/139772>`_
* PR `Add PyDict_NewPresized() function
  <https://github.com/python/cpython/pull/139773>`__
* PR `Add PyDict_FromItems() function
  <https://github.com/python/cpython/pull/139963>`__
* PR `Add PyDict_FromKeysAndValues() function
  <https://github.com/python/cpython/pull/141682>`__
* C API Working Group decision issue `Add PyDict_NewPresized() function
  <https://github.com/capi-workgroup/decisions/issues/80>`__
* C API Working Group decision issue `Add PyDict_FromItems() function
  <https://github.com/capi-workgroup/decisions/issues/90>`__


Copyright
=========

This document is placed in the public domain or under the
CC0-1.0-Universal license, whichever is more permissive.
