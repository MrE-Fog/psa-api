.. SPDX-FileCopyrightText: Copyright 2018-2023 Arm Limited and/or its affiliates <open-source-office@arm.com>
.. SPDX-License-Identifier: CC-BY-SA-4.0 AND LicenseRef-Patent-license

.. _implementation-considerations:

Implementation considerations
-----------------------------

Implementation-specific aspects of the interface
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Implementation profile
^^^^^^^^^^^^^^^^^^^^^^

Implementations can implement a subset of the API and a subset of the available
algorithms. The implemented subset is known as the implementation’s profile. The
documentation for each implementation must describe the profile that it
implements. This specification’s companion documents also define a number of
standard profiles.

.. _implementation-defined-type:

Implementation-specific types
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This specification defines a number of implementation-specific types, which
represent objects whose content depends on the implementation. These are defined
as C ``typedef`` types in this specification, with a comment
:code:`/* implementation-defined type */` in place of the underlying type
definition. For some types the specification constrains the type, for example,
by requiring that the type is a ``struct``, or that it is convertible to and
from an unsigned integer. In the implementation's version of :file:`psa/crypto.h`,
these types need to be defined as complete C types so that objects of these
types can be instantiated by application code.

Applications that rely on the implementation specific definition of any of these
types might not be portable to other implementations of this specification.

.. _implementation-specific-macro:

Implementation-specific macros
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Some macro constants and function-like macros are precisely defined by this
specification. The use of an exact definition is essential if the definition can
appear in more than one header file within a compilation.

Other macros that are defined by this specification have a macro body that is
implementation-specific. The description of an implementation-specific macro can
optionally specify each of the following requirements:

*   Input domains: the macro must be valid for arguments within the input domain.
*   A return type: the macro result must be compatible with this type.
*   Output range: the macro result must lie in the output range.
*   Computed value: A precise mapping of valid input to output values.

Each implementation-specific macro is in one of following categories:

.. _specification-defined-value:

*Specification-defined value*
    The result type and computed value of the macro expression is defined by
    this specification, but the definition of the macro body is provided by the
    implementation.

    These macros are indicated in this specification using the comment:

    .. code-block:: xref

        /* specification-defined value */

    .. TODO!!
        Change this text when we have provided pseudo-code implementations of
        all the relevant macro expressions.

    For function-like macros with specification-defined values:

    *   Example implementations are provided in an appendix to this specification.
        See :secref:`appendix-specdef-values`.

    *   The expected computation for valid and supported input arguments will be
        defined as pseudo-code in a future version of this specification.

.. _implementation-defined-value:

*Implementation-defined value*
    The value of the macro expression is implementation-defined.

    For some macros, the computed value is derived from the specification of one
    or more cryptographic algorithms. In these cases, the result must exactly
    match the value in those external specifications.

    These macros are indicated in this specification using the comment:

    .. code-block:: xref

        /* implementation-defined value */


Some of these macros compute a result based on an algorithm or key type.
If an implementation defines vendor-specific algorithms or
key types, then it must provide an implementation for such macros that takes all
relevant algorithms and types into account. Conversely, an implementation that
does not support a certain algorithm or key type can define such macros in a
simpler way that does not take unsupported argument values into account.

Some macros define the minimum sufficient output buffer size for certain
functions. In some cases, an implementation is permitted to require a buffer size
that is larger than the theoretical minimum. An implementation must define
minimum-size macros in such a way that it guarantees that the buffer of the
resulting size is sufficient for the output of the corresponding function. Refer
to each macro’s documentation for the applicable requirements.

Porting to a platform
~~~~~~~~~~~~~~~~~~~~~

Platform assumptions
^^^^^^^^^^^^^^^^^^^^

This specification is designed for a C99 platform. The interface is defined in
terms of C macros, functions and objects.

The specification assumes 8-bit bytes, and “byte” and “octet” are used
synonymously.

Platform-specific types
^^^^^^^^^^^^^^^^^^^^^^^

The specification makes use of some types defined in C99. These types must be
defined in the implementation version of :file:`psa/crypto.h` or by a header
included in this file. The following C99 types are used:

``uint8_t``, ``uint16_t``, ``uint32_t``
    Unsigned integer types with 8, 16 and 32 value bits respectively.
    These types are defined by the C99 header :file:`stdint.h`.

Cryptographic hardware support
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Implementations are encouraged to make use of hardware accelerators where
available. A future version of this specification will define a function
interface that calls drivers for hardware accelerators and external
cryptographic hardware.

Security requirements and recommendations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Error detection
^^^^^^^^^^^^^^^

Implementations that provide :term:`isolation` between the caller and the cryptography
processing environment must validate parameters to ensure that the cryptography
processing environment is protected from attacks caused by passing invalid
parameters.

Even implementations that do not provide isolation are recommended to detect bad
parameters and fail-safe where possible.

Indirect object references
^^^^^^^^^^^^^^^^^^^^^^^^^^

Implementations can use different strategies for allocating key identifiers,
and other types of indirect object reference.

Implementations that provide isolation between the caller and the cryptography
processing environment must consider the threats relating to abuse and misuse
of key identifiers and other indirect resource references. For example,
multi-part operations can be implemented as backend state to which the client
only maintains an indirect reference in the application's multi-part operation
object.

An implementation that supports multiple callers must implement strict isolation
of API resources between different callers. For example, a client must not be
able to obtain a reference to another client's key by guessing the key
identifier value. Isolation of key identifiers can be achieved in several ways.
For example:

*   There is a single identifier namespace for all clients, and the
    implementation verifies that the client is the owner of the identifier when
    looking up the key.
*   Each client has an independent identifier namespace, and the implementation
    uses a client specific identifier-to-key mapping when looking up the key.

After a volatile key identifier is destroyed, it is recommended that the
implementation does not immediately reuse the same identifier value for a
different key. This reduces the risk of an attack that is able to exploit a key
identifier reuse vulnerability within an application.

.. _memory-cleanup:

Memory cleanup
^^^^^^^^^^^^^^

Implementations must wipe all sensitive data from memory when it is no longer
used. It is recommended that they wipe this sensitive data as soon as possible. All
temporary data used during the execution of a function, such as stack buffers,
must be wiped before the function returns. All data associated with an object,
such as a multi-part operation, must be wiped, at the latest, when the object
becomes inactive, for example, when a multi-part operation is aborted.

The rationale for this non-functional requirement is to minimize impact if the
system is compromised. If sensitive data is wiped immediately after use, only
data that is currently in use can be leaked. It does not compromise past data.

.. _key-material:

Managing key material
^^^^^^^^^^^^^^^^^^^^^

In implementations that have limited volatile memory for keys, the
implementation is permitted to store a :term:`volatile key` to a
temporary location in non-volatile memory. The implementation must delete any
non-volatile copies when the key is destroyed, and it is recommended that these copies
are deleted as soon as the key is reloaded into volatile memory. An
implementation that uses this method must clear any stored volatile key material
on startup.

Implementing the memory cleanup rule (see :secref:`memory-cleanup`) for a :term:`persistent key`
can result in inefficiencies when the same persistent key is used sequentially
in multiple cryptographic operations. The inefficiency stems from loading the
key from non-volatile storage on each use of the key. The `PSA_KEY_USAGE_CACHE`
usage flag in a key policy allows an application to request that the implementation does not cleanup
non-essential copies of persistent key material, effectively suspending the
cleanup rules for that key. The effects of this policy depend on the
implementation and the key, for example:

*   For volatile keys or keys in a secure element with no open/close mechanism,
    this is likely to have no effect.
*   For persistent keys that are not in a secure element, this allows the
    implementation to keep the key in a memory cache outside of the memory used
    by ongoing operations.
*   For keys in a secure element with an open/close mechanism, this is a hint to
    keep the key open in the secure element.

The application can indicate when it has finished using the key by calling
`psa_purge_key()`, to request that the key material is cleaned from memory.

Safe outputs on error
^^^^^^^^^^^^^^^^^^^^^

Implementations must ensure that confidential data is not written to output
parameters before validating that the disclosure of this confidential data is
authorized. This requirement is particularly important for implementations where
the caller can share memory with another security context, as described in
:secref:`stability-of-parameters`.

In most cases, the specification does not define the content of output
parameters when an error occurs. It is recommended that implementations try to
ensure that the content of output parameters is as safe as possible, in case an
application flaw or a data leak causes it to be used. In particular, Arm
recommends that implementations avoid placing partial output in output buffers
when an action is interrupted. The meaning of “safe as possible” depends on the
implementation, as different environments require different compromises between
implementation complexity, overall robustness and performance. Some common
strategies are to leave output parameters unchanged, in case of errors, or
zeroing them out.

Attack resistance
^^^^^^^^^^^^^^^^^

Cryptographic code tends to manipulate high-value secrets, from which other
secrets can be unlocked. As such, it is a high-value target for attacks. There
is a vast body of literature on attack types, such as side channel attacks and
glitch attacks. Typical side channels include timing, cache access patterns,
branch-prediction access patterns, power consumption, radio emissions and more.

This specification does not specify particular requirements for attack
resistance. Implementers are encouraged to consider the attack resistance
desired in each use case and design their implementation accordingly. Security
standards for attack resistance for particular targets might be applicable in
certain use cases.

Other implementation considerations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Philosophy of resource management
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The specification allows most functions to return
:code:`PSA_ERROR_INSUFFICIENT_MEMORY`. This gives implementations the freedom to
manage memory as they please.

Alternatively, the interface is also designed for conservative strategies of
memory management. An implementation can avoid dynamic memory allocation
altogether by obeying certain restrictions:

*   Pre-allocate memory for a predefined number of keys, each with sufficient
    memory for all key types that can be stored.
*   For multi-part operations, in an implementation with :term:`no isolation`, place all
    the data that needs to be carried over from one step to the next in the
    operation object. The application is then fully in control of how memory is
    allocated for the operation.
*   In an implementation with :term:`isolation`, pre-allocate memory for a predefined
    number of operations inside the cryptoprocessor.

.. Inclusion of algorithms

    Inline algorithm-generic functions into specialized functions at compile/link time
