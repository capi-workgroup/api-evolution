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

