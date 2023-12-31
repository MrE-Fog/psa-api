.. SPDX-FileCopyrightText: Copyright 2018-2022 Arm Limited and/or its affiliates <open-source-office@arm.com>
.. SPDX-License-Identifier: CC-BY-SA-4.0 AND LicenseRef-Patent-license

.. _architectures:

Sample architectures
--------------------

This section describes some example architectures that can be used for
implementations of the interface described in this specification. This list is
not exhaustive and the section is entirely non-normative.

Single-partition architecture
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In the single-partition architecture, there is no security boundary inside the system. The
application code can access all the system memory, including the memory used by
the cryptographic services described in this specification. Thus, the
architecture provides :term:`no isolation`.

This architecture does not conform to the Arm *Platform Security Architecture
Security Model*. However, it is useful for providing cryptographic services
that use the same interface, even on devices that cannot support any security
boundary. So, while this architecture is not the primary design goal of the API
defined in the present specification, it is supported.

The functions in this specification simply execute the underlying algorithmic
code. Security checks can be kept to a minimum, since the cryptoprocessor cannot
defend against a malicious application. Key import and export copy data inside
the same memory space.

This architecture also describes a subset of some larger systems, where the
cryptographic services are implemented inside a high-security partition,
separate from the code of the main application, though it shares this
high-security partition with other platform security services.

.. _isolated-cryptoprocessor:

Cryptographic token and single-application processor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This system is composed of two partitions: one is a cryptoprocessor and the
other partition runs an application. There is a security boundary between the
two partitions, so that the application cannot access the cryptoprocessor,
except through its public interface. Thus, the architecture provides
:term:`cryptoprocessor isolation`. The cryptoprocessor has
some non-volatile storage, a TRNG, and possibly, some cryptographic accelerators.

There are a number of potential physical realizations: the cryptoprocessor might
be a separate chip, a separate processor on the same chip, or a logical
partition using a combination of hardware and software to provide the isolation.
These realizations are functionally equivalent in terms of the offered software
interface, but they would typically offer different levels of security
guarantees.

The |API| in the application processor consists of a thin layer of code
that translates function calls to remote procedure calls in the cryptoprocessor.
All cryptographic computations are, therefore, performed inside the
cryptoprocessor. Non-volatile keys are stored inside the cryptoprocessor.

Cryptoprocessor with no key storage
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

As in the :secref:`isolated-cryptoprocessor` architecture, this system
is also composed of two partitions separated by a security boundary and also
provides :term:`cryptoprocessor isolation`.
However, unlike the previous architecture, in this system, the cryptoprocessor
does not have any secure, persistent storage that could be used to store
application keys.

If the cryptoprocessor is not capable of storing cryptographic material, then
there is little use for a separate cryptoprocessor, since all data would have to
be imported by the application.

The cryptoprocessor can provide useful services if it is able to store at least
one key. This might be a hardware unique key that is burnt to one-time
programmable memory during the manufacturing of the device. This key can be used
for one or more purposes:

*   Encrypt and authenticate data stored in the application processor.
*   Communicate with a paired device.
*   Allow the application to perform operations with keys that are derived from
    the hardware unique key.

Multi-client cryptoprocessor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is an expanded variant of
:secref:`isolated-cryptoprocessor`. In this
variant, the cryptoprocessor serves multiple applications that are mutually
untrustworthy. This architecture provides :term:`caller isolation`.

In this architecture, API calls are translated to remote procedure calls, which
encode the identity of the client application. The cryptoprocessor carefully
segments its internal storage to ensure that a client’s data is never leaked to
another client.

Multi-cryptoprocessor architecture
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This system includes multiple cryptoprocessors. There are several reasons to
have multiple cryptoprocessors:

*   Different compromises between security and performance for different keys.
    Typically, this means a cryptoprocessor that runs on the same hardware as the
    main application and processes short-term secrets, a secure element or a
    similar separate chip that retains long-term secrets.
*   Independent provisioning of certain secrets.
*   A combination of a non-removable cryptoprocessor and removable ones, for
    example, a smartcard or HSM.
*   Cryptoprocessors managed by different stakeholders who do not trust each
    other.

The keystore implementation needs to dispatch each request to the correct
processor. For example:

*   All requests involving a non-extractable key must be processed in the
    cryptoprocessor that holds that key.
*   Requests involving a persistent key must be processed in the cryptoprocessor
    that corresponds to the key’s lifetime value.
*   Requests involving a volatile key might target a cryptoprocessor based on
    parameters supplied by the application, or based on considerations such as
    performance inside the implementation.
