::

  ZIP: XXX
  Title: Enabling Blind Off-chain Lightweight Transactions (Bolt) using shielded/transparent features
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

Layer 2 protocols like Lightning enable scalable payments for Bitcoin. This specification describes an approach to enable a similar, privacy-preserving layer 2 protocol on top of Zcash.

Specification
=============

Based on a mix of shielded and transparent transactions, we need the following capabilities:

(1) Ability to create a funding transaction such that the transaction inputs originate from the shielded pool.
(2) Ability to do relative time locks for commitment transactions to support unilateral channel closing.
(3) Ability to do absolute time locks to support multi-hop payments.
(4) Ability to validate Bolt-specific commitment opening message and closing signature:

    - check the validity of the commitment opening
    - check the validity of blinded signature on the wallet commitment in closure token
    - check the validity of revocation token signature in the event of a channel dispute by merchant

(5) Ability to verify the transaction output such that:

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

We assume the following specific features are present:

(1) ``OP_CLTV`` - absolute lock time
(2) ``OP_CSV`` - relative lock time
(3) Can specify shielded inputs and outputs
(4) P2SH support - to build a 2-of-2 multi-sig style transaction
(5) ``OP_BOLT`` opcode: takes one argument (the first byte represents the mode followed by a serialized token of hex encoded bytes) and outputs a ``True`` or ``False`` on the stack:

    * Mode 1 (for customer-initiated close). This mode expects a channel token and a customer closure token of one of the following types:

       (a) An opening of the channel's initial wallet commitment. This type of closure token is to be used when no payments have been made on the specified channel. The opcode verifies that the provided commitment opening is valid with respect to the specified channel.

       (b) A signature under the merchant's longterm keypair on the customer's current wallet state, together with the wallet state. This type of closure token is to be used when one or more payment have been made on the channel. The opcode validates the merchant signature on the closure token first. Then, the opcode verifies two additional constraints: (1) there are two outputs in the closing transaction: one paying the merchant his balance and the other paying the customer, and (2) the customer’s payout is timelocked (to allow for merchant dispute).

    * Mode 2 (for merchant dispute of customer closure token). This mode is used in a merchant closing transaction to dispute a customer's closure token. The opcode expects a merchant revocation token. It validates the revocation token with respect to the wallet pub key posted by the customer in the customer's closing transaction. If valid, the customer's closure token will be invalidated and the merchant's closing transaction will be deemed valid.

**Privacy Limitations**. The aggregate balance of the channel will be revealed in the 2-of-2 multi-sig transparent address. Similarly, the final spliting of funds will be revealed to the network. However, for channel opening and closing, the identity of the participants remain hidden. Channel opening and closing will also be distinguishable on the network due to use of ``OP_BOLT`` opcodes.

**Channel Opening**. The customer creates a funding transaction that spends ZEC from a shielded address to a 2-of-2 multi-sig transparent address using a pay-to-script-hash (P2SH) output with a `pay-to-public-key-hash (P2PKH)` embedded inside the script. Here is what the funding transaction looks like when opening the channel.

**Token Descriptions**. There are three types of tokens described in this section: (1) channel token, (2) closure token, and (3) revocation token.

(a) *Channel token*: this consists of public key from the customer and merchant for the channel and a long-lived public key for the merchant. It also includes the initial customer balance and optionally, the wallet commitment.
(b) *Closure token*: for the customer, this consists of the wallet (i.e., the channel public key, wallet public key, current channel balance, total channel balance), and a closure signature (i.e., blinded sig) on the wallet.
(c) *Revocation token*: this consists of a wallet public key and a corresponding revocation signature.

1.2 Funding Transaction
-------------
The funding transaction is by default funded by only one participant, the customer. We will be extending the protocol to allow for dual-funded channels.

This transaction has 2 shielded inputs (but can be up to some N) and 1 output to a P2SH address (to a 2-of-2 multi-sig address) with the merchant public key. Note that the customer can specify as many shielded inputs as necessary to fund the channel sufficiently (limited only by the overall transaction size).

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

  - ``scriptPubKey`` must have the form ``0 <32-byte hash>``, where the latter is the hash of the script needed to spend the output.

To redeem this output as the customer, the redeeming transaction must use the following ``scriptSig``:

	1 <<channel-token> <closing-token>> <cust-sig> <serializedScript>,

or as the merchant:

	1 <cust-sig> <merch-sig> <serializedScript>,

where ``serializedScript`` is as follows:

	OP_IF
	  2 <cust-pubkey> <merch-pubkey> 2 OP_CHECKMULTISIG
	OP_ELSE
	  <cust-pubkey> OP_CHECKSIGVERIFY 1 OP_BOLT
	OP_ENDIF

* ``bindingSig``: a signature that proves that (1) the total value spent by Spend transfers - Output transfers = value balance field.

The customer (in collaboration with the merchant) creates their initial closing transaction before sending the funding transaction to the network (since  the customer needs to know they can get their money back). Once both customer and merchant closing transactions have been created, the customer should broadcast the funding transaction and waits for the network to confirm the transaction. After the transaction has been confirmed, the payment channel is established.

1.3 Closing Transactions
-------------
1.3.1 Customer closing transaction
----
The customer closing transaction is generated by the customer during the channel establishment but is not broadcast to the network. The customer's closing transaction (below) contains two outputs: (1) an output that can be spent immediately by the merchant and (2) another output that can be spent by either the customer after a relative timeout or the merchant with a revocation token. This approach allows the merchant to see the customer's closing transaction and spend the output with a revocation token if the customer posted an outdated closure token.

The customer's closing transaction is described below.

* ``version``: specify version number
* ``groupid``: specify group id
* ``locktime``: should be set such that closing transactions can be included in a current block.
* ``txin`` count: 1

   - ``txin[0]`` outpoint: references the funding transaction txid and output_index
   - ``txin[0]`` script bytes: 0
   - ``txin[0]`` script sig: 0 <<channel-token> <closing-token>> <cust-sig> <OP_IF 2 <cust-pubkey> <merch-pubkey> 2 OP_CHECKMULTISIG OP_ELSE <cust-pubkey> OP_CHECKSIGVERIFY 1 OP_BOLT OP_ENDIF>

* ``txout`` count: 2
* ``txouts``:

  * ``to_customer``: a timelocked (using ``OP_CSV``) P2SH output sending funds back to the customer.
      - ``amount``: balance paid back to customer
      - ``nSequence: <time-delay>``
      - ``scriptPubKey: 0 <32-byte-hash>``
      - ``scriptSig: (empty)``

  * ``to_merchant``: A P2PKH to merch-pubkey output (sending funds back to the merchant), i.e.
      * ``scriptPubKey``: ``0 <20-byte-key-hash of merch-pubkey>``

To redeem the ``to_customer`` output, the customer presents a ``scriptSig`` with the customer signature after a time delay as follows:

	``1 <cust-sig> 0 <serializedScript>``

where the ``serializedScript`` is as follows

	``OP_IF``
	  ``<revocation-pubkey> <merch-pubkey> 2 OP_BOLT``
	``OP_ELSE``
	  ``<time-delay> OP_CSV OP_DROP <cust-pubkey> OP_CHECKSIGVERIFY``
	``OP_ENDIF``

In the event of a dispute, the merchant can redeem the ``to_customer`` by posting a transaction ``scriptSig`` as follows:

	``<revocation-token> <merch-sig> 1``

1.3.2 Merchant closing transaction
----
The merchant can create their own initial closing transaction as follows.

* ``version``: specify version number
* ``groupid``: specify group id
* ``locktime``: should be set such that closing transactions can be included in a current block.
* ``txin`` count: 1

   - ``txin[0]`` outpoint: references the funding transaction txid and output_index
   - ``txin[0]`` script bytes: 0
   - ``txin[0]`` script sig: 0 <cust-sig> <merch-sig> <2 <cust-pubkey> <merch-pubkey> 2 OP_CHECKMULTISIG>

* ``txout`` count: 1
* ``txouts``:

  * ``to_merchant``: a timelocked (using ``OP_CSV``) P2SH output sending all the funds back to the merchant. So ``scriptPubKey`` is of the form ``0 <32-byte-hash>``.
      - ``amount``: balance paid back to merchant
      - ``nSequence: <time-delay>``
      - ``script sig: 1 <merch-sig> 0 <serializedScript>``
      - ``serializedScript``:

		OP_IF
	  	  <cust-pubkey> OP_CHECKSIGVERIFY 1 OP_BOLT
		OP_ELSE
		  <time-delay> OP_CSV OP_DROP <merch-pubkey> OP_CHECKSIGVERIFY
		OP_ENDIF

After each payment on the channel, the customer obtains a closing token for the updated channel balance and provides the merchant a revocation token for the previous state along with the associated wallet public key (this invalidates the pub key). If the customer initiated closing, the merchant can use the revocation token to spend the funds of the channel if the customer posts an outdated closing transaction.

1.4 Channel Closing
-------------
To close the channel, the customer can initiate by posting the most recent closing transaction (in Section 2.3) that spends from the multi-signature transparent address with inputs that satisfies the script and the ``OP_BOLT`` opcode in mode 1. This consists of a closing token (i.e., merchant signature on the wallet state) or an opening of the initial wallet commitment (if there were no payments on the channel via mode 2).

Once the timeout has been reached, the customer can post a transaction that claims the output of the customer closing transaction to a shielded output (see below for an example). Before the timeout, the merchant can claim the funds from the ``to_customer`` output by posting a revocation token, if they have one.

The merchant can immediately claim the ``to_merchant`` output from the customer closing transaction to a shielded address by presenting their P2PKH address.

Because we do not know how to encumber the outputs of shielded outputs right now, we will rely on a standard transaction to move funds from the closing transaction into a shielded address as follows:

* ``version``: 2
* ``groupid``: specify group id
* ``locktime``: 0
* ``txin`` count: 1
   * ``txin[0]`` outpoint: ``txid`` and ``output_index``
   * ``txin[0]`` sequence: 0xFFFFFFFF
   * ``txin[0]`` script bytes: 0
   * ``txin[0]`` script sig: ``0 <cust-sig> <merch-sig>``
* ``nShieldedOutput``: 1
* ``vShieldedOutput[0]``:
   - ``cv``: commitment for the output note
   - ``cmu``: ...
   - ``ephemeralKey``:ephemeral public key
   - ``encCiphertext``: encrypted output note (part 1)
   - ``outCiphertext``: encrypted output note (part 2)
   - ``zkproof``: zero-knowledge proof for the note

The merchant can initiate closing by posting the initial closing transaction (in Section 2.3) from establishing the channel that pays the merchant the full balance of the channel with a time lock that allows for customer dispute. The merchant can then post a separate standard transaction that moves those funds to a shielded address.

Reference Implementation
========================

We are currently working on a reference implementation based on section 2 in a fork of Zcash here: https://github.com/boltlabs-inc/zcash.

References
==========

.. [#RFC2119] `Key words for use in RFCs to Indicate Requirement Levels <https://tools.ietf.org/html/rfc2119>`_
