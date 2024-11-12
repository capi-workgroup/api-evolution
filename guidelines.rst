PEP: 9999
Title: Guidelines for Python's C API
Author: Petr Viktorin <encukou@gmail.com>,
        Guido van Rossum <guido@python.org>,
        Victor Stinner <vstinner@python.org>,
        Steve Dower <steve.dower@python.org>,
        Erlend Egeberg Aasland <erlend@python.org>,
        Serhiy Storchaka,
        Michael Droettboom,
Discussions-To: https://discuss.python.org/c/c-api/30
Status: Draft
Type: Process
Requires: 731
Created: 20-Nov-2023
Post-History: XXX

.. pep-banner::

   This is a **incomplete draft**, published for initial discussion.
   It is likely to be amended substantially.


Abstract
========

TODO: Write the abstract last :)


About this document
===================

These guidelines represent the consensus of the C API Working Group,
established in :pep:`731` to oversee and coordinate
the development and maintenance of the CPython C API.

It is a living document. It can be changed with the approval of either
all members of the C API Working Group, or the Steering Council.


High-level goals
================

Our goal is that the C API is:

* **Useful**: It should provide access to all relevant functionality.
* **Ergonomic**: It should be easy to use.
* **Fair**: It should serve the purposes of a variety of stakeholders.
* **Stable**: It should avoid requiring users to update their code more than necessary.
* **Evolvable**: It should give Python implementers enough freedom to change implementation details.
* **Maintainable**: It should be easy to maintain and extend.

.. note:: 

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/26


Meta-guidelines
===============

Don't panic!
------------

We do not expect you, fellow CPython contributor, to read and remember
all of the guidelines.
If you have any doubts, or if you find that the guidelines are unclear,
incomplete or contradictory, ask the C API Working Group for help
and clarifications.


Working group approvals
-----------------------

In several cases, you should seek approval from the C API Working Group
before changing public C API.
Most notably:

* when the guidelines tell you to do so,
* when adding an exception to the rules,
* when adding to the *limited* API,
* when adding something that is likely to establish precedent,
* when adding many APIs at once, or
* when the guidelines are unclear or don't make sense.

Approvals allow the working group to coordinate designs from multiple people,
and to collect feedback needed to refine these guidelines.

Request an approval by opening `an issue in the decisions repository`_.

.. _an issue in the decisions repository: https://github.com/capi-workgroup/decisions/issues


Exceptions to the guidelines
----------------------------

Where possible, exceptions to these guidelines should be added as
*alternatives* to API that follows the guidelines.
Such exceptions should be clearly identifiable from their names.

If you want to add an exception that is not mentioned in the guidelines,
please request approval from the C API Working Group.

.. note:: 

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/4


Applicability
=============

These guidelines only apply to API that is:

* *Newly added*: existing API does not necessarily follow these guidelines;
  this PEP is not the place for plans to replace, deprecate or remove it.
* *Public*: meant for users; that is, not in the *private* or *internal*
  tiers as defined in the next section.

Note that we have a style guide, :pep:`7`, which applies to *all* C code
in CPython, including the API.


API tiers
---------

CPython has several tiers of C API.
The more stable the tier is, the stricter the rules for it are:

*  For **internal** API (only accessible with ``Py_BUILD_CORE``, or not declared
   in a public header at all), you are free to ignore these guidelines entirely.

*  **Private** API (not intended for users) should be *internal* if possible.
   Otherwise it must use the ``_Py`` prefix (see :ref:`naming`).
   The rest of these guidelines don't apply to it.

*  In **unstable** API (prefixed with ``PyUnstable_``), it is OK to ignore
   these guidelines if there is a reason to do so -- usually for performance.
   Please document the reason in a source comment.

*  In the **general** public API, these guidelines apply in full force.
   The C API working group can grant exceptions.

*  For the **limited** API, always seek explicit approval from the
   C API Working Group.


.. note:: 

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/42 (for limited API)


One header
==========

All public API should be available after including :file:`Python.h`.

To allow selecting an alternate API, such as a subset,
use feature flags/macros that users define before including the header
(for example ``Py_LIMITED_API``).
Adding such a feature macro needs approval from the C API Working Group.

.. note::

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/34


.. XXX add a PEP number prefix to the anchor:

.. _naming:

Naming
======

[TODO: PEP 7 should be updated to link here once this PEP goes live.]

All newly added public names must be prefixed with ``Py``.

Names that users should not use directly, but need to be visible to the
compiler/linker, should be prefixed with ``_Py``.
(Such names are not considered public API, that is, they should not appear in
third-party source code.)

This applies to all names in a global namespace: functions, macros, variables,
typedefs, structs, enums, etc.; not to parameters or struct fields.

The ``Py_`` prefix is reserved for global service routines like
``Py_FatalError``; specific groups of APIs use a longer prefix,
for example ``PyUnicode_`` for string functions.
Use an existing prefix when applicable. If you want to add a new prefix,
contact the C API Working Group.

The ``Py`` prefix is in mixed case, even in macro names.
For example: ``PyUnicode_AS_STRING``.
(Several existing macros use the upper-case ``PY``; if you need this prefix
for consistency, please get approval from the C API Working Group.)

Unstable API is prefixed with ``PyUnstable_`` instead of ``Py``,
for example ``PyUnstable_Long_IsCompact`` or (hypothetically)
``PyUnstable_String_GET_SIZE``.

.. note::

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/21


Do not reuse names
------------------

If an API's interface or behavior changes in a backwards-incompatible way,
add new API with a new name.
You can deprecate and remove the old version following Python's
:pep:`backwards compatibility policy <387>`.

After API has been removed, do not reuse the old name,
since existing documentation and tutorials will continue to refer to the
old behavior.


Language support
================

C standard and dialect
----------------------

[TODO: PEP 7 should be updated to link here once this PEP goes live.]

Public C API must be compatible with:

- C11, with optional features needed by CPython:

  - IEEE 754 floating point
  - Atomics (``!__STDC_NO_ATOMICS__``, or MSVC)

- C99
- C89 with several select C99 features:

  - ``<stdint.h>`` and ``<inttypes.h>``
  - ``static inline`` functions
  - designated initializers
  - intermingled declarations
  - line comments (``//``)

- C++03

It is OK to use other features -- compiler-specific ones,
optional standard ones, or platform-specific ones -- if:

- the behavior of correct user code is the same as with a standard compiler,
- the feature is detected using appropriate preprocessor checks, and
- their use does not produce warnings on any supported compiler,
  including earlier versions of the one it is specific to.

For example, compiler-specific code is often used to improve performance
or compiler diagnostics.

It is also OK to use these other features for platform-specific API,
which needs to be documented as such and have a feature test macro
(for example, ``Py_HAVE_C_COMPLEX``).

.. note::

   The existing API uses a few C11 features which are
   commonly available as compiler extensions to C99.
   In particular, we do use an unnamed union.
   New API should not use these features.

All function declarations and definitions must use full prototypes,
that is, the types of all arguments must be specified.

.. note::

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/22


Portable subset of C
--------------------

While the C API is defined in terms of the C language, it supports building
wrappers for languages other than C, such as Rust, Java, assembly, or
Python with ``ctypes``.
These wrappers cannot realisically use a full-featured C parser.
To make the public API easier to describe and wrap, it should
avoid some of C's features:

*  Avoid ``static inline`` functions and macros.
   All functions must be exported as actual library symbols.
*  Avoid variadic functions.
*  Avoid C-specific types like ``long long``, ``enum`` or bit fields
   (see :ref:`types`).

Once you add API that conforms to this “portable subset”,
you can add additional C/C++-specific API. Usually, the additional API
will be either more performant, or easier to use from C.
For example:

*  A function may be shadowed by a ``static inline`` function or macro with
   the same behavior. Typically, this allows better performance for C/C++.
   See `shadowing example`_.
*  A function that uses C-specific types, such as ``PyLong_AsLongLong``,
   is OK if equivalent functions are provided for the preferred types.
*  A variadic function is OK if there's an equivalent function
   that takes an array.

Macros can be used in the following cases:

*  Feature flags (e.g. ``HAVE_FORK``, ``Py_LIMITED_API``)
*  Simple constants (e.g. ``Py_TPFLAGS_BASETYPE``, ``PY_VERSION_HEX``).
*  Shortcuts for functionality that can be accomplished trivially,
   but perhaps tediously, without macros (e.g. ``Py_VISIT``,
   ``Py_BEGIN_ALLOW_THREADS``, ``Py_RETURN_RICHCOMPARE``).
   In this case, the macro-less equivalent should be clear from documentation:
   consider adding the macro's expansion to the docs.
   Non-C wrappers are expected re-implement these macros.
*  Features that aren't needed in non-C languages (e.g. ``Py_MAX``,
   ``Py_STRINGIFY``).
*  Macros used to define the API (e.g. ``PyAPI_FUNC``, ``Py_ALWAYS_INLINE``,
   ``Py_OBJECT_H``).
*  As an implementation detail (for example, when shadowing a function).

As always, new exceptions can be added with approval from the C API
working group.

.. note::

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/11 (Treat the ABI as an API)
    - https://github.com/capi-workgroup/api-evolution/issues/18 (Avoid macros and static inline functions)
    - https://github.com/capi-workgroup/api-evolution/issues/12 (variadics)


.. _types:

Types
=====

Arithmetic types
----------------

Avoid types with compiler-/platform-specific sizes, such as  ``long``
or ``unsigned short``.

Instead, use:

*  ``int32_t`` and other C99+ fixed width integer types
*  ``Py_ssize_t``, ``intptr_t``, ``ptrdiff_t`` for values of the appropriate
   platform-specific types
*  ``char``
*  ``double`` (IEEE 754 ``binary64``)

The only exception is ``int``, which should be used for small ranges
(typically, as a replacement for enum).
If the 16-bit limit is relevant, and for all unsigned values, prefer
explicit fixed width types over ``int``.

For memory sizes and byte counts, use the signed ``Py_ssize_t``,
not the unsigned ``size_t``.


Enums and bitfields
-------------------

Avoid ``enum``, which have compiler-/platform-dependent size.
(CPython can not yet use C23's fixed underlying `enum` types.)
Instead, use ``int`` with defined constants.

Also avoid bitfields, which have compiler-/platform-dependent memory layout.
Instead, use fixed width integer types and bitmask constants.

To be clear: it is fine to use ``enum`` and bitfields outside the
public headers.


Objects
-------

Use ``PyObject*`` for all Python objects.
Avoid using concrete types (e.g. ``PyDictObject*``).

Public API should type-check all objects passed to it.
When it gets an object of an unexpected type, public API should fail with
``TypeError`` rather than crash.

As an exception, with approval from the C API Working Group you can use concrete types,
such as ``PyTypeObject*``, ``PyCodeObject*`` & ``PyFrameObject*``,
for consistency with existing API.
These objects should be type-checked as if they were ``PyObject*``.


.. note:: 

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/29
    - https://github.com/capi-workgroup/decisions/issues/19


Return values
=============

The return value of a function must indicate whether an exception was set.
It must not be necessary to use ``PyErr_Occurred`` to disambiguate.
(Recall that these guidelines apply to *new* API; existing API does not
necessarily follow this.)

Generally, API functions can return one of:

*  An integral value, where ``-1`` is returned if and only if an exception was
   set, and other values signal an absence of exception.
*  A pointer, where ``NULL`` is returned if and only if an exception was set.
*  A few special cases:

   -  Functions that never return, or always set an exception, should use
      the :c:expr:`void` return type.
   -  Functions used when the runtime might not be initialized
      should either:

      *  return ``PyStatus``, or
      *  return ``-1``/``NULL`` to signal failure, but have an alternate way
         of reporting error details.

In cases where ``-1`` or ``NULL`` is a valid result, use an
:ref:`output argument <output argument>` to provide that result.

See `return schemes`_ for concrete examples.

.. note::

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/13
    - https://github.com/capi-workgroup/decisions/issues/19


Exceptions for infallible functions
-----------------------------------

Some functions can not fail, and callers cannot check for exceptions:

* Deallocators and reference sinks like ``PyMem_Free`` and ``Py_DECREF``,
  which use ``void`` as the return type.

Other functions cannot fail, and users may *optionally* skip error
checking:

* Refcounting operations, like ``Py_NewRef``.
* Operations on native types that cannot have exceptional cases
  (e.g. overflow), like ``Py_HashPointer``.
* Subtype-checking functions, like ``PyTuple_Check`` and ``PyTuple_CheckExact``,
  which return either ``0`` or ``1``.
  (This does not extend to *subclass* checking, like ``PyObject_IsSubclass``,
  which can call Python code.)

Even in these cases, ``-1`` and ``NULL`` are reserved for errors;
infallible functions must never return these values.
(This allows auto-generated API wrappers to avoid unnecessary special cases.)

As always, other exceptions can be added here with approval from the
C API working group.

Mind that infallibility is very often an implementation detail
that should not be exposed in the API. That is, some functions'
*current CPython implementations* cannot fail,
but we may want the function to fail (or warn) in the future.
Users should only skip error checking if the function's documentation
explicitly allows it.


.. _output argument:

Output arguments
================

.. (There's nothing to say for *arguments* in general, it's all under
   types or reference conting.
   If that changes, "Output arguments" should be a subsection of "Arguments")

Output arguments are pointers to memory that a function fills in.
Use these when a result from a function cannot be returned as the return value,
for example:

* ``-1`` or ``NULL`` is a valid (non-exceptional) result: for example,
  in ``PyDict_GetItemRef``.
* There are multiple results: for example, in ``PyUnicode_AsUTF8AndSize``.

Guidelines for output arguments:

* Functions must always fill in the output arguments. If an error
  occurs or the rseult is not available, the output should typically be set
  to NULL or zero.
* When it might be useful for users to call a function but ignore an output,
  allow passing NULL as the output argument.
* Ownership of a ``PyObject*`` result is transferred to the caller,
  as with return values.
  [TODO: Link to guidelines about ownership & borrowing, when those are added]

Since existing API does not necessarily follow these guidelines,
all of the above points should be explicitly mentioned in the
documentation of each function they're relevant to.

.. note::

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/32


.. _shadowing example:

Appendix A. Shadowing example
=============================

To provide a ``static inline`` equivalent to an exported function,
write something like:

Header:

.. code-block:: c

    static inline returntype
    _Py_Foo_impl(ARGS)
    {
        ...
    }

    PyAPI_FUNC(returntype) Py_Foo (ARGS);

    #define Py_Foo _Py_Foo_impl

Code:

.. code-block:: c

    // at the end (after all calls to Py_Foo):
    #undef Py_Foo

    returntype
    Py_Foo(ARGS)
    {
        return _Py_Foo_impl(ARGS);
    }



.. _return schemes:

Appendix B. Return value schemes
================================

Here are common schemes of how to encode return values.

*  Success or failure: return ``int``

   * ``0`` for success
   * ``-1``, with an exception set, for failure

*  Yes or no: return ``int``

   *  ``0`` for ``false``
   *  ``1`` for ``true``
   *  ``-1``, with an exception set, for failure

*  Lookup (“getattr”, “getitem” or “setdefault” style) functions: return
   ``int``; the lookup result is passed via an
   :ref:`output argument <output argument>`):

   *  ``0`` for “not found”
      (*result* is set to ``NULL`` or other zero/empty value)
   *  ``1`` for “found” (*result* is set)
   *  ``-1``, with an exception set, for failure
      (*result* is set to ``NULL`` or other zero/empty value)

*  Enumeration-style: return ``int``

   * ``-1``, with an exception set, for failure
   * Zero and positive numbers for valid results

*  Hashes: return ``Py_hash_t``

   *  ``-1``, with an exception set, for failure
   * All other numbers for result.

*  Objects: return ``PyObject*``

   * ``NULL``, with an exception set, for failure
   * Valid pointer for result
