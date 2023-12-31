.. SPDX-FileCopyrightText: Copyright 2018-2023 Arm Limited and/or its affiliates <open-source-office@arm.com>
.. SPDX-License-Identifier: CC-BY-SA-4.0 AND LicenseRef-Patent-license

.. header:: psa/crypto
    :seq: 28

.. _key-agreement:

Key agreement
=============

Two functions are provided for a Diffie-Hellman-style key agreement where each party combines its own private key with the peer’s public key.

*   The recommended approach is to use a :ref:`key derivation operation <kdf>` with the `psa_key_derivation_key_agreement()` input function, which calculates a shared secret for the key derivation function.

*   Where an application needs direct access to the shared secret, it can call `psa_raw_key_agreement()` instead. Note that in general the shared secret is not directly suitable for use as a key because it is biased.

.. _key-agreement-algorithms:

Key agreement algorithms
------------------------

.. macro:: PSA_ALG_FFDH
    :definition: ((psa_algorithm_t)0x09010000)

    .. summary::
        The finite-field Diffie-Hellman (DH) key agreement algorithm.

    This algorithm can be used directly in a call to `psa_raw_key_agreement()`, or combined with a key derivation operation using `PSA_ALG_KEY_AGREEMENT()` for use with `psa_key_derivation_key_agreement()`.

    When used as a key's permitted-algorithm policy, the following uses are permitted:

    *   In a call to `psa_raw_key_agreement()`, with algorithm `PSA_ALG_FFDH`.
    *   In a call to `psa_key_derivation_key_agreement()`, with any combined key agreement and key derivation algorithm constructed with `PSA_ALG_FFDH`.

    When used as part of a multi-part key derivation operation, this implements a Diffie-Hellman key agreement scheme using a single Diffie-Hellman key-pair for each participant. This includes the *dhEphem*, *dhOneFlow*, and *dhStatic* schemes. The input step `PSA_KEY_DERIVATION_INPUT_SECRET` is used when providing the secret and peer keys to the operation.

    The shared secret produced by this key agreement algorithm is ``g^{ab}`` in big-endian format. It is ``ceiling(m / 8)`` bytes long where ``m`` is the size of the prime ``p`` in bits.

    This key agreement scheme is defined by :cite-title:`SP800-56A` §5.7.1.1 under the name FFC DH.

    .. subsection:: Compatible key types

        | `PSA_KEY_TYPE_DH_KEY_PAIR()`

.. macro:: PSA_ALG_ECDH
    :definition: ((psa_algorithm_t)0x09020000)

    .. summary::
        The elliptic curve Diffie-Hellman (ECDH) key agreement algorithm.

    This algorithm can be used directly in a call to `psa_raw_key_agreement()`, or combined with a key derivation operation using `PSA_ALG_KEY_AGREEMENT()` for use with `psa_key_derivation_key_agreement()`.

    When used as a key's permitted-algorithm policy, the following uses are permitted:

    *   In a call to `psa_raw_key_agreement()`, with algorithm `PSA_ALG_ECDH`.
    *   In a call to `psa_key_derivation_key_agreement()`, with any combined key agreement and key derivation algorithm constructed with `PSA_ALG_ECDH`.

    When used as part of a multi-part key derivation operation, this implements a Diffie-Hellman key agreement scheme using a single elliptic curve key-pair for each participant. This includes the *Ephemeral unified model*, the *Static unified model*, and the *One-pass Diffie-Hellman* schemes. The input step `PSA_KEY_DERIVATION_INPUT_SECRET` is used when providing the secret and peer keys to the operation.

    The shared secret produced by key agreement is the x-coordinate of the shared secret point. It is always ``ceiling(m / 8)`` bytes long where ``m`` is the bit size associated with the curve, i.e. the bit size of the order of the curve's coordinate field. When ``m`` is not a multiple of 8, the byte containing the most significant bit of the shared secret is padded with zero bits. The byte order is either little-endian or big-endian depending on the curve type.

    *   For Montgomery curves (curve family `PSA_ECC_FAMILY_MONTGOMERY`), the shared secret is the x-coordinate of ``Z = d_A Q_B = d_B Q_A`` in little-endian byte order.

        -   For Curve25519, this is the X25519 function defined in :cite-title:`Curve25519`. The bit size ``m`` is 255.
        -   For Curve448, this is the X448 function defined in :cite-title:`Curve448`. The bit size ``m`` is 448.

    *   For Weierstrass curves (curve families ``PSA_ECC_FAMILY_SECP_XX``, ``PSA_ECC_FAMILY_SECT_XX``, `PSA_ECC_FAMILY_BRAINPOOL_P_R1` and `PSA_ECC_FAMILY_FRP`) the shared secret is the x-coordinate of ``Z = h d_A Q_B = h d_B Q_A`` in big-endian byte order. This is the Elliptic Curve Cryptography Cofactor Diffie-Hellman primitive defined by :cite-title:`SEC1` §3.3.2 as, and also as ECC CDH by :cite-title:`SP800-56A` §5.7.1.2.

        -   Over prime fields (curve families ``PSA_ECC_FAMILY_SECP_XX``, `PSA_ECC_FAMILY_BRAINPOOL_P_R1` and `PSA_ECC_FAMILY_FRP`), the bit size is ``m = ceiling(log_2(p))`` for the field ``F_p``.
        -   Over binary fields (curve families ``PSA_ECC_FAMILY_SECT_XX``), the bit size is ``m`` for the field ``F_{2^m}``.

        .. note::

            The cofactor Diffie-Hellman primitive is equivalent to the standard elliptic curve Diffie-Hellman calculation ``Z = d_A Q_B = d_B Q_A`` (`[SEC1]` §3.3.1) for curves where the cofactor ``h`` is ``1``. This is true for all curves in the ``PSA_ECC_FAMILY_SECP_XX``, `PSA_ECC_FAMILY_BRAINPOOL_P_R1`, and `PSA_ECC_FAMILY_FRP` families.

    .. subsection:: Compatible key types

        | :code:`PSA_KEY_TYPE_ECC_KEY_PAIR(family)`

        where ``family`` is a Weierstrass or Montgomery Elliptic curve family. That is, one of the following values:

        *   ``PSA_ECC_FAMILY_SECT_XX``
        *   ``PSA_ECC_FAMILY_SECP_XX``
        *   `PSA_ECC_FAMILY_FRP`
        *   `PSA_ECC_FAMILY_BRAINPOOL_P_R1`
        *   `PSA_ECC_FAMILY_MONTGOMERY`

.. macro:: PSA_ALG_KEY_AGREEMENT
    :definition: /* specification-defined value */

    .. summary::
        Macro to build a combined algorithm that chains a key agreement with a key derivation.

    .. param:: ka_alg
        A key agreement algorithm: a value of type `psa_algorithm_t` such that :code:`PSA_ALG_IS_KEY_AGREEMENT(ka_alg)` is true.
    .. param:: kdf_alg
        A key derivation algorithm: a value of type `psa_algorithm_t` such that :code:`PSA_ALG_IS_KEY_DERIVATION(kdf_alg)` is true.

    .. return::
        The corresponding key agreement and derivation algorithm.

        Unspecified if ``ka_alg`` is not a supported key agreement algorithm or ``kdf_alg`` is not a supported key derivation algorithm.

    A combined key agreement algorithm is used with a multi-part key derivation operation, using a call to `psa_key_derivation_key_agreement()`.

    The component parts of a key agreement algorithm can be extracted using `PSA_ALG_KEY_AGREEMENT_GET_BASE()` and `PSA_ALG_KEY_AGREEMENT_GET_KDF()`.

    .. subsection:: Compatible key types

        The resulting combined key agreement algorithm is compatible with the same key types as the raw key agreement algorithm used to construct it.


Standalone key agreement
------------------------

.. function:: psa_raw_key_agreement

    .. summary::
        Perform a key agreement and return the raw shared secret.

    .. param:: psa_algorithm_t alg
        The key agreement algorithm to compute: a value of type `psa_algorithm_t` such that :code:`PSA_ALG_IS_RAW_KEY_AGREEMENT(alg)` is true.
    .. param:: psa_key_id_t private_key
        Identifier of the private key to use.
        It must permit the usage `PSA_KEY_USAGE_DERIVE`.
    .. param:: const uint8_t * peer_key
        Public key of the peer. The peer key must be in the same format that `psa_import_key()` accepts for the public key type corresponding to the type of ``private_key``. That is, this function performs the equivalent of :code:`psa_import_key(..., peer_key, peer_key_length)`, with key attributes indicating the public key type corresponding to the type of ``private_key``. For example, for ECC keys, this means that peer_key is interpreted as a point on the curve that the private key is on. The standard formats for public keys are documented in the documentation of `psa_export_public_key()`.
    .. param:: size_t peer_key_length
        Size of ``peer_key`` in bytes.
    .. param:: uint8_t * output
        Buffer where the raw shared secret is to be written.
    .. param:: size_t output_size
        Size of the ``output`` buffer in bytes.
        This must be appropriate for the keys:

        *   The required output size is :code:`PSA_RAW_KEY_AGREEMENT_OUTPUT_SIZE(type, bits)` where ``type`` is the type of ``private_key`` and ``bits`` is the bit-size of either ``private_key`` or the ``peer_key``.
        *   `PSA_RAW_KEY_AGREEMENT_OUTPUT_MAX_SIZE` evaluates to the maximum output size of any supported raw key agreement algorithm.

    .. param:: size_t * output_length
        On success, the number of bytes that make up the returned output.

    .. return:: psa_status_t
    .. retval:: PSA_SUCCESS
        Success.
        The first ``(*output_length)`` bytes of ``output`` contain the raw shared secret.
    .. retval:: PSA_ERROR_INVALID_HANDLE
        ``private_key`` is not a valid key identifier.
    .. retval:: PSA_ERROR_NOT_PERMITTED
        ``private_key`` does not have the `PSA_KEY_USAGE_DERIVE` flag, or it does not permit the requested algorithm.
    .. retval:: PSA_ERROR_INVALID_ARGUMENT
        The following conditions can result in this error:

        *   ``alg`` is not a key agreement algorithm.
        *   ``private_key`` is not compatible with ``alg``.
        *   ``peer_key`` is not a valid public key corresponding to ``private_key``.
    .. retval:: PSA_ERROR_BUFFER_TOO_SMALL
        The size of the ``output`` buffer is too small.
        `PSA_RAW_KEY_AGREEMENT_OUTPUT_SIZE()` or `PSA_RAW_KEY_AGREEMENT_OUTPUT_MAX_SIZE` can be used to determine a sufficient buffer size.
    .. retval:: PSA_ERROR_NOT_SUPPORTED
        The following conditions can result in this error:

        *   ``alg`` is not supported or is not a key agreement algorithm.
        *   ``private_key`` is not supported for use with ``alg``.
    .. retval:: PSA_ERROR_INSUFFICIENT_MEMORY
    .. retval:: PSA_ERROR_COMMUNICATION_FAILURE
    .. retval:: PSA_ERROR_CORRUPTION_DETECTED
    .. retval:: PSA_ERROR_STORAGE_FAILURE
    .. retval:: PSA_ERROR_DATA_CORRUPT
    .. retval:: PSA_ERROR_DATA_INVALID
    .. retval:: PSA_ERROR_BAD_STATE
        The library requires initializing by a call to `psa_crypto_init()`.

    .. warning::
        The raw result of a key agreement algorithm such as finite-field Diffie-Hellman or elliptic curve Diffie-Hellman has biases, and is not suitable for use as key material. Instead it is recommended that the result is used as input to a key derivation algorithm. To chain a key agreement with a key derivation, use `psa_key_derivation_key_agreement()` and other functions from the key derivation interface.

Combining key agreement and key derivation
------------------------------------------

.. function:: psa_key_derivation_key_agreement

    .. summary::
        Perform a key agreement and use the shared secret as input to a key derivation.

    .. param:: psa_key_derivation_operation_t * operation
        The key derivation operation object to use. It must have been set up with `psa_key_derivation_setup()` with a key agreement and derivation algorithm ``alg``: a value of type `psa_algorithm_t` such that :code:`PSA_ALG_IS_KEY_AGREEMENT(alg)` is true and :code:`PSA_ALG_IS_RAW_KEY_AGREEMENT(alg)` is false.

        The operation must be ready for an input of the type given by ``step``.
    .. param:: psa_key_derivation_step_t step
        Which step the input data is for.
    .. param:: psa_key_id_t private_key
        Identifier of the private key to use.
        It must permit the usage `PSA_KEY_USAGE_DERIVE`.
    .. param:: const uint8_t * peer_key
        Public key of the peer. The peer key must be in the same format that `psa_import_key()` accepts for the public key type corresponding to the type of ``private_key``. That is, this function performs the equivalent of :code:`psa_import_key(..., peer_key, peer_key_length)`, with key attributes indicating the public key type corresponding to the type of ``private_key``. For example, for ECC keys, this means that peer_key is interpreted as a point on the curve that the private key is on. The standard formats for public keys are documented in the documentation of `psa_export_public_key()`.
    .. param:: size_t peer_key_length
        Size of ``peer_key`` in bytes.

    .. return:: psa_status_t
    .. retval:: PSA_SUCCESS
        Success.
    .. retval:: PSA_ERROR_BAD_STATE
        The following conditions can result in this error:

        *   The operation state is not valid for this key agreement ``step``.
        *   The library requires initializing by a call to `psa_crypto_init()`.
    .. retval:: PSA_ERROR_INVALID_HANDLE
        ``private_key`` is not a valid key identifier.
    .. retval:: PSA_ERROR_NOT_PERMITTED
        ``private_key`` does not have the `PSA_KEY_USAGE_DERIVE` flag, or it does not permit the operation's algorithm.
    .. retval:: PSA_ERROR_INVALID_ARGUMENT
        The following conditions can result in this error:

        *   The operation's algorithm is not a key agreement algorithm.
        *   ``step`` does not permit an input resulting from a key agreement.
        *   ``private_key`` is not compatible with the operation's algorithm.
        *   ``peer_key`` is not a valid public key corresponding to ``private_key``.
    .. retval:: PSA_ERROR_NOT_SUPPORTED
        ``private_key`` is not supported for use with the operation's algorithm.
    .. retval:: PSA_ERROR_INSUFFICIENT_MEMORY
    .. retval:: PSA_ERROR_COMMUNICATION_FAILURE
    .. retval:: PSA_ERROR_CORRUPTION_DETECTED
    .. retval:: PSA_ERROR_STORAGE_FAILURE
    .. retval:: PSA_ERROR_DATA_CORRUPT
    .. retval:: PSA_ERROR_DATA_INVALID

    A key agreement algorithm takes two inputs: a private key ``private_key``, and a public key ``peer_key``. The result of this function is passed as input to the key derivation operation. The output of this key derivation can be extracted by reading from the resulting operation to produce keys and other cryptographic material.

    If this function returns an error status, the operation enters an error state and must be aborted by calling `psa_key_derivation_abort()`.

Support macros
--------------

.. macro:: PSA_ALG_KEY_AGREEMENT_GET_BASE
    :definition: /* specification-defined value */

    .. summary::
        Get the raw key agreement algorithm from a full key agreement algorithm.

    .. param:: alg
        A key agreement algorithm: a value of type `psa_algorithm_t` such that :code:`PSA_ALG_IS_KEY_AGREEMENT(alg)` is true.

    .. return::
        The underlying raw key agreement algorithm if ``alg`` is a key agreement algorithm.

        Unspecified if ``alg`` is not a key agreement algorithm or if it is not supported by the implementation.

    See also `PSA_ALG_KEY_AGREEMENT()` and `PSA_ALG_KEY_AGREEMENT_GET_KDF()`.

.. macro:: PSA_ALG_KEY_AGREEMENT_GET_KDF
    :definition: /* specification-defined value */

    .. summary::
        Get the key derivation algorithm used in a full key agreement algorithm.

    .. param:: alg
        A key agreement algorithm: a value of type `psa_algorithm_t` such that :code:`PSA_ALG_IS_KEY_AGREEMENT(alg)` is true.

    .. return::
        The underlying key derivation algorithm if ``alg`` is a key agreement algorithm.

        Unspecified if ``alg`` is not a key agreement algorithm or if it is not supported by the implementation.

    See also `PSA_ALG_KEY_AGREEMENT()` and `PSA_ALG_KEY_AGREEMENT_GET_BASE()`.

.. macro:: PSA_ALG_IS_RAW_KEY_AGREEMENT
    :definition: /* specification-defined value */

    .. summary::
        Whether the specified algorithm is a raw key agreement algorithm.

    .. param:: alg
        An algorithm identifier: a value of type `psa_algorithm_t`.

    .. return::
        ``1`` if ``alg`` is a raw key agreement algorithm, ``0`` otherwise. This macro can return either ``0`` or ``1`` if ``alg`` is not a supported algorithm identifier.

    A raw key agreement algorithm is one that does not specify a key derivation function. Usually, raw key agreement algorithms are constructed directly with a ``PSA_ALG_xxx`` macro while non-raw key agreement algorithms are constructed with `PSA_ALG_KEY_AGREEMENT()`.

    The raw key agreement algorithm can be extracted from a full key agreement algorithm identifier using `PSA_ALG_KEY_AGREEMENT_GET_BASE()`.

.. macro:: PSA_ALG_IS_FFDH
    :definition: /* specification-defined value */

    .. summary::
        Whether the specified algorithm is a finite field Diffie-Hellman algorithm.

    .. param:: alg
        An algorithm identifier: a value of type `psa_algorithm_t`.

    .. return::
        ``1`` if ``alg`` is a finite field Diffie-Hellman algorithm, ``0`` otherwise. This macro can return either ``0`` or ``1`` if ``alg`` is not a supported key agreement algorithm identifier.

    This includes the raw finite field Diffie-Hellman algorithm as well as finite-field Diffie-Hellman followed by any supported key derivation algorithm.

.. macro:: PSA_ALG_IS_ECDH
    :definition: /* specification-defined value */

    .. summary::
        Whether the specified algorithm is an elliptic curve Diffie-Hellman algorithm.

    .. param:: alg
        An algorithm identifier: a value of type `psa_algorithm_t`.

    .. return::
        ``1`` if ``alg`` is an elliptic curve Diffie-Hellman algorithm, ``0`` otherwise. This macro can return either ``0`` or ``1`` if ``alg`` is not a supported key agreement algorithm identifier.

    This includes the raw elliptic curve Diffie-Hellman algorithm as well as elliptic curve Diffie-Hellman followed by any supporter key derivation algorithm.

.. macro:: PSA_RAW_KEY_AGREEMENT_OUTPUT_SIZE
    :definition: /* implementation-defined value */

    .. summary::
        Sufficient output buffer size for `psa_raw_key_agreement()`.

    .. param:: key_type
        A supported key type.
    .. param:: key_bits
        The size of the key in bits.

    .. return::
        A sufficient output buffer size for the specified key type and size. An implementation can return either ``0`` or a correct size for a key type and size that it recognizes, but does not support. If the parameters are not valid, the return value is unspecified.

    If the size of the output buffer is at least this large, it is guaranteed that `psa_raw_key_agreement()` will not fail due to an insufficient buffer size. The actual size of the output might be smaller in any given call.

    See also `PSA_RAW_KEY_AGREEMENT_OUTPUT_MAX_SIZE`.

.. macro:: PSA_RAW_KEY_AGREEMENT_OUTPUT_MAX_SIZE
    :definition: /* implementation-defined value */

    .. summary::
        Sufficient output buffer size for `psa_raw_key_agreement()`, for any of the supported key types and key agreement algorithms.

    If the size of the output buffer is at least this large, it is guaranteed that `psa_raw_key_agreement()` will not fail due to an insufficient buffer size.

    See also `PSA_RAW_KEY_AGREEMENT_OUTPUT_SIZE()`.
