++++++++++++++++++++++++++++++++++++++++++++++++
Add ``PyDict_FromItems()`` function to the C API
++++++++++++++++++++++++++++++++++++++++++++++++

Rationale
=========

Private C API
-------------

For historical reasons, Python exposes private functions in its public C API.
These functions are not tested, not documented, can change or even be removed
anytime without notifying users. Private functions used by 3rd party projects
should be promoted to public functions to add tests, documentation and
backward compatibility warranties.

The private ``_PyDict_NewPresized()`` function is used by 11 projects of the
PyPI top 15,000. API::

    PyObject* _PyDict_NewPresized(Py_ssize_t minused)

It preallocates *minused* items for the created dictionary.

Unicode issue
-------------

The dictionary lookup can be optimized for Unicode keys if all keys are
Unicode. ``_PyDict_NewPresized()`` doesn't support this optimization.

On the other hand, ``PyDict_New()`` doesn't allocate the hashtable. The first
``PyDict_SetItem()`` call allocates *minsize* (``8``) keys table with the
information if the first key is Unicode or not.

This is the main reason why CPython itself doesn't use
``_PyDict_NewPresized()``, but use ``_PyDict_FromItems()`` instead. The private
``_PyDict_FromItems()`` is similar to the public ``PyDict_FromItems()``
proposed in this PEP.


The false size header problem
-----------------------------

msgpack includes the size in the container header for preallocation. But by
crafting a false size header, you can request the preallocation of a dictionary
with 100 million elements using just 10 bytes of msgpack data.

This is a problem for lists as well, but it becomes more serious for
dictionaries, because they have a larger memory overhead per element.

That's why in msgpack-python, ``_PyDict_NewPresized()`` is not used.


Specification
=============

API
---

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
if all keys are Unicode strings.

Advantages
----------

``PyDict_FromItems()`` solves the different issues:

* `Unicode issue`_: it scans keys to check if all keys are Unicode to enable
  the Unicode keys lookup optimization.
* `The false size header problem`_: the *length* argument provides exactly the
  number of items.

It's also faster than calling ``PyDict_New()`` + ``PyDict_SetItem()`` on each
item: see `Benchmark`_.

Replace private ``_PyStack_AsDict()``
-------------------------------------

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
optimization which is an implementation detail. ``PyDict_FromItems()`` scans
keys internally for the Unicode keys optimization

INADA-san `wrote
<https://github.com/python/cpython/issues/139772#issuecomment-3385134698>`__
that most users either overestimate its effectiveness or don't fully understand
how it operates.


Add ``PyDict_SetAssumptions()`` function
----------------------------------------

Add ``PyDict_SetAssumptions()`` function to the C API::

    int PyDict_SetAssumptions(Py_ssize_t size, unsigned int flags)

    #define PY_DICTFLAG_UNICODE_KEYS 0x01

Return ``1`` if the dictionary has been adjusted properly. Return ``0`` if the
dictionary cannot be adjusted: the dictionary is left unchanged in this case.

If the ``PY_DICTFLAG_UNICODE_KEYS`` flag is used, optimize the dictionary for
Unicode keys.

This API is similar to the ``PyDict_NewPresized()`` API, but it doesn't prevent
using the dictionary on failure and it can be used on a non-empty dictionary.

As ``PyDict_NewPresized()``, ``PyDict_SetAssumptions()`` is also too low-level
with its ``PY_DICTFLAG_UNICODE_KEYS`` flag, compared to ``PyDict_FromItems()``
which scans keys internally for the Unicode keys optimization.


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
        Py_ssize_t length,
        uint64 flags)

    #define Py_DICTFLAG_KEEP_EXISTING 0x01

Create a new dictionary if *dict* is ``NULL``, or update an existing dictionary
otherwise.

Keys are overridden by default (*flags*=0). If the ``Py_DICTFLAG_KEEP_EXISTING``
flag is set, existing keys are kept instead.

This API has two more parameters (*dict*, *override*) than
``PyDict_FromItems()`` which already has 5 arguments, so it's
and harder to use. It's way more common to create a dictionary (``dict=NULL``)
than updating an existing dictionary (non-NULL ``dict``), but that might change
if the ``PyDict_SetAssumptions()`` function is added.


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
