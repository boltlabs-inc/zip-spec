::

  ZIP: XXX
  Title: Non-private scripting for private transactions
  Authors: J. Ayo Akinyele <ayo@boltlabs.io>
           Colleen Swanson <swan@boltlabs.io>
           James Prestwich <james@summa.one>
  Credits: Ian Miers <imiers@z.cash>
  Category: Consensus
  Created: 2019-07-02
  License: MIT


Terminology
===========

The key words "MUST" and "MUST NOT" in this document are to be interpreted as described in RFC 2119.

Abstract
========

This proposal describes a new opcode for the Bitcoin scripting system that restricts the execution of a strict in such a way that we can enable non-private scripting for shielded transactions and fix transaction malleability with respect to transparent inputs.

Motivation
==========

The ability to add general programmability to shielded transactions is crucial for several use cases using Zcash which include the Blind Off-chain Lightweight Transactions (BOLT) protocol at layer 2 and support for Hashed-time Lock contracts (HTLCs) for better cross-chain interoperability with other chains.
A solution that leverages Bitcoin script system in combination with shielded transactions is complicated by a lack of transaction malleability fix. Adopting a SegWit-style solution would break consensus rules that enable shielded transactions today and therefore incompatible with Zcash.

Instead, we are proposing a solution ...

Specification
=============

``ESCAPE`` redefines an existing NOP4 opcode such that when it is executed, the script interpreter will stop execution and
execute logic based on a message handler opcode CONTRACT that will complete checking the correctness of the transaction.

The ``CONTRACT`` message handler opcode will then either be a BOLT or HTLC type and then the ``CONTRACT`` opcode will also ensure
that the transaction is not malleable by checking that there are only shielded inputs specified in the transaction.

This solution would not break compatibility with existing clients as the opcodes will be ignored if they don't have BOLT or HTLC support.
And these additional opcodes would be a minimal addition to the Bitcoin script language for the purposes of enabling non-private scripting on
shielded transactions.

Transaction Examples
-------

* ``lock_time``: 0
* ``nExpiryHeight``: 0
* ``valueBalance``: funding amount + transaction fee
* ``nShieldedSpend``: 1 or N (if funded by both customer and merchant)
* ``vShieldedSpend[0]``: tx for customer’s note commitment and nullifier for the coins

  - ``cv``: commitment for the input note
  - ``root``: root hash of note commitment tree at some block height
  - ``nullifier``: unique serial number of the input note
  - ``rk``: randomized pubkey for spendAuthSig
  - ``zkproof``: zero-knowledge proof for the note
  - ``spendAuthSig``: signature authorizing the spend

* ``vShieldedSpend[1..N]``: additional tx for customer's note commitment and nullifier for the coins

  - ``cv``: commitment for the input note
  - ``root``: root hash of note commitment tree at some block height
  - ``nullifier``: unique serial number of the input note
  - ``rk``: randomized pubkey for spendAuthSig
  - ``zkproof``: zero-knowledge proof for the note
  - ``spendAuthSig``: signature authorizing the spend
* ``tx_out_count``: 1
* ``tx_out``: (using a P2SH address)

  - ``scriptPubKey``: ``OP_ESCAPE OP_BOLT <version> <type> <binary-data>

(1) Ability to validate Bolt-specific commitment opening message and closing signature:

    - check the validity of the commitment opening
    - check the validity of blinded signature on the wallet commitment in closure token
    - check the validity of revocation token signature in the event of a channel dispute by merchant

(2) Ability to verify the transaction output such that:

    - if customer initiated closing, first output pays out to customer with a time lock (to allow merchant to dispute customer balance) and second output pays out to merchant immediately
    - if merchant initiated closing, a single output that pays the merchant the full balance of the channel with a time lock that allows for customer dispute
    - if mutual closing, an output to both parties with the appropriate balances and no time lock

**Channel Operation Assumptions.**
 - Single-funded channel by customer with a minimum fee paid to the merchant.
 - Either the customer or the merchant can initiate channel closing.
 - If the customer initiates closing, then the merchant can dispute the closing transaction if they disagrees with the closure token in the closing transaction.
 - If the merchant initiates closing, the customer has the opportunity to post their own valid closing transaction. In this case, the merchant has an additional opportunity to validate this closing transaction and can dispute if necessary.

Conditions for Opening Channel
-------------

To open a channel, a customer picks a channel-specific public key, commits to an initial wallet, and receives a signature from the merchant (using their long-term keypair) on that wallet. A wallet consists of a wallet-specific public key, a customer balance, and a total channel balance, and is linked to the customer's channel-specific public key. The channel specific public key, initial customer balance, total channel balance, and initial wallet commitment comprise the customer's channel token.

The keypairs used by both the merchant and the customer must support a blind signature scheme.

Conditions for Closing Channel
-------------

A customer should be able to close the channel by either opening the initial wallet commitment (if no payments made) or posting a closing token.

A merchant should be able to close the channel by either posting their closing token or, if the customer posts an outdated version of their closure token (or opens the initial wallet commitment for the channel after one or more payments have been made), a revocation token.

Reference Implementation
========================

Reference implementation will go here and will include message handler implementation for BOLT as an example: https://github.com/boltlabs-inc/zcash.

References
==========

.. [#RFC2119] `Key words for use in RFCs to Indicate Requirement Levels <https://tools.ietf.org/html/rfc2119>`_
