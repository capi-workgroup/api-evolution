PEP: 9999
Title: Guidelines for Python's C API
Author: Petr Viktorin <encukou@gmail.com>,
Discussions-To: https://discuss.python.org/c/c-api/30
Status: Draft
Type: Process
Requires: 731
Created: 20-Nov-2023
Post-History: XXX

.. pep-banner::

   This is a **incomplete draft**, published for initial discussion.
   It is likely to be amended substantially.

.. Authors to be added:

        Guido van Rossum <guido@python.org>,
        Victor Stinner <vstinner@python.org>,
        Steve Dower <steve.dower@python.org>,
        Irit Katriel <irit@python.org>


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

All public names must be prefixed with ``Py``.

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

The ``Py_`` prefix is in mixed case, even in macro names.
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

If an API's behavior changes in a backwards-incompatible way,
for example when a function's signature changes,
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
optional standard ones, or platform-specific ones -- if the behavior of
correct user code is the same as with a standard compiler.
For example, compiler-specific code is often used to improve performance
or compiler diagnostics.

It is also OK to use these other features for platform-specific API,
which needs to be documented as such and have a feature test macro
(for example, ``PY_HAVE_THREAD_NATIVE_ID``).

.. note::

   The existing API uses a few C11 features which are
   commonly available as compiler extensions to C99.
   In particular, we do use an unnamed union.
   New API should not use these features.

All function declarations and definitions must use full prototypes,
that is, the types of all arguments must be specified.

Argument *names* should generally be included in declarations,
but may be omitted if the meaning is obvious.

.. note::

    For background and discussions, see:

    - https://github.com/capi-workgroup/api-evolution/issues/22


Types
=====


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

