::

  ZIP: XXX
  Title: Enabling Blind Off-chain Lightweight Transactions (Bolt) using shielded-only features
  Authors: J. Ayo Akinyele <ayo@boltlabs.io>
           Colleen Swanson <swan@boltlabs.io>
  Credits: Ian Miers <imiers@z.cash>
           Matthew Green <mgreen@z.cash>
  Category: Consensus
  Created: 2019-06-24
  License: MIT


Terminology
===========

The key words "MUST" and "MUST NOT" in this document are to be interpreted as described in RFC 2119.

Abstract
========

This proposal specifies three possible approaches for integrating the blind off-chain lightweight transaction (Bolt) protocol into Zcash.

Motivation
==========

Layer 2 protocols like Lightning enable scalable payments for Bitcoin. We want to implement a similar, privacy-preserving layer 2 protocol on top of Zcash.

Specification
=============

Using shielded-only features of Zcash, we need the following capabilities:

(1) Ability to verify additional fields from the shielded transaction as part of signature verification.
(2) Ability to do relative time locks for commitment transactions to support unilateral channel closing.
(3) Ability to do absolute time locks to support multi-hop payments.
(4) Ability to validate Bolt-specific commitment opening message and closing signature:

    - check the validity of the commitment opening
    - check the validity of blinded signature on the wallet commitment in closure token
    - check the validity of revocation token signature in the event of a channel dispute by merchant

(5) Ability to encumber the transaction output such that:

    - if customer initiated closing, first output pays out to customer with a time lock (to allow merchant to dispute customer balance) and second output pays out to merchant immediately
    - if merchant initiated closing, a single output that pays the merchant the full balance of the channel with a time lock that allows for customer dispute

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

1.1 Specific Features
-------------
We assume the following features are present:

(a) ``lock_time`` for absolute lock time
(b) A field to enforce relative lock time
(c) 2-of-2 multi-sig shielded address support
(d) All inputs/outputs are specified from/to shielded pool
(e) A method to encumber the outputs of a shielded transaction
(f) An extension to the transaction format to include BOLT tokens (e.g., like ``vBoltDescription``)
(g) Add new ``SIGHASH`` flags to cover the extended field

The goal here is to perform all the same validation steps for channel opening/closing without relying on the scripting system, as well as allowing for relative timelocks (the equivalent of ``OP_CSV``). In order to support multi-hop payments, we need absolute timelocks as well (the equivalent of ``OP_CLTV``). We also want to ensure that transactions are non-malleable in order to allow for unconfirmed dependency transaction chains.

**Limitations/Notes**. With extensions to shielded transaction format, it may be evident whenever parties are establishing private payment channels. We appreciate feedback on the feasibility of what is proposed for each aspect of the Bolt protocol.

**Channel Opening**. The customer creates a funding transaction that spends ZEC from a shielded address to a 2-of-2 multi-sig shielded address. Here is the flow (1) creating a multi-sig shielded address specifying both parties keys and (2) generating channel tokens.

1.2 Funding Transaction
-------------
This transaction has 2 shielded inputs (but can be up to some N) and 1 output to a 2-of-2 shielded address. If a ``vBoltDescription`` field is added, then we could use it to store the channel parameters and the channel token for opening the channel.

1.3 Closing Transaction
-------------
The initial wallet commitment will spend from the shielded address to two shielded outputs.  The first shielded output pays the customer with a timelock (or the merchant with a revocation token) and the second shielded output allows the merchant to spend immediately. It is not clear to us whether it will be possible to encumber the outputs of shielded outputs directly.

Feedback from @Str4d on how we could encumber shielded outputs:

* The encumbered output would contain a commitment to the various Bolt parameters (the timelock, the revocation token, etc).
     * Without changing the Sapling circuit, the commitment would be added to a global Merkle tree in parallel to the current Sapling Merkle tree (meaning that they don't have a shared privacy set).
     * If the Sapling circuit was altered, the privacy sets could potentially be shared, at the cost of requiring all Sapling users to be aware of Bolt semantics. IMHO this probably isn't worth the cost of doing such a change, but we could consider it during a later general programmability solution.
     * The parameters themselves would probably also be included directly in the transaction in an encrypted field (as we do for shielded notes).

* The spend using that output would contain a proof using the Bolt circuit, and the necessary public inputs such as the "time" at which the proof was created (perhaps stored in the locktime field).
     * The circuit would enforce the equivalent of the OP_BOLT logic, allowing a valid proof to be created if the prover had knowledge of the revocation key and merchant key, OR the prover had knowledge of the customer key AND the public time input was past the committed timelock. It would also enforce all the necessary peripherial checks (the parameters match the original commitment, there exists a Merkle path from the original commitment to a specified public anchor, etc.).
     * Network nodes would validate the Bolt-specific proof, and also validate the public inputs (if necessary, e.g. the locktime field is already enforced by the network).

1.4 Channel Closing
-------------
The channel closing consists of the customer broadcasting the most recent commitment transaction and requires that they present the closure token necessary to claim the funds. Similarly, the merchant would be able to claim the funds with the appropriate revocation token as well.

Reference Implementation
========================

We are currently working on a reference implementation based on section 2 in a fork of Zcash here: https://github.com/boltlabs-inc/zcash.

References
==========

.. [#RFC2119] `Key words for use in RFCs to Indicate Requirement Levels <https://tools.ietf.org/html/rfc2119>`_
