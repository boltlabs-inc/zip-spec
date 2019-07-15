::

  ZIP: XXX
  Title: Add support for Blind Off-chain Lightweight Transactions (Bolt) protocol
  Authors: J. Ayo Akinyele <ayo@boltlabs.io>
           Colleen Swanson <swan@boltlabs.io>
  Credits: Ian Miers <imiers@z.cash>
           Matthew Green <mgreen@z.cash>
  Category: Consensus
  Created: 2019-07-04
  License: MIT


Terminology
===========

The key words "MUST" and "MUST NOT" in this document are to be interpreted as described in RFC 2119. [#RFC2119]_

Abstract
========

This proposal specifies three possible approaches for integrating the blind off-chain lightweight transaction (Bolt) protocol [#bolt-paper]_ into Zcash.

Motivation
==========

Layer 2 protocols like Lightning enable scalable payments for Bitcoin but lack the mechanisms to provide strong privacy guarantees on the payment network. Zcash offers private transactions but currently lacks features that would enable a lightning-style payment channel. This proposal specifies an integration of the Bolt privacy-preserving Lightning protocol on top of Zcash [#bolt-paper]_.

Specification
=============

This specification details an initial approach to integrating the features of Bolt into Zcash in a future network upgrade and depends on the WTP ZIP [#wtp-programs]_ that introduces Whitelisted Transparent Programs (WTPs).

1. General requirements for Bolt protocol
--------------------------

Bolt private payment channels require the following capabilities to provide anonymity properties for users on a payment network:

(1) Ability to create a escrow transaction such that the transaction inputs are anonymous.
(2) Ability to escrow funds to a multi-signature style address via non-malleable transactions.
(3) Ability to specify relative time locks for commitment transactions to support unilateral channel closing.
(4) Ability to specify absolute and relative time locks to support HTLCs for multi-hop payments.
(5) Ability to validate Bolt-specific opening and closing transactions:

    - check the validity of randomized/blinded signature on the wallet commitment in closing token
    - check the validity of revocation token in the event of a channel dispute by merchant

(6) Ability to verify transaction outputs using WTPs such that:

    - if customer-initiated closing, one output pays out to customer with a time lock (to allow merchant to dispute customer balance) and one output pays out to merchant immediately
    - if merchant-initiated closing, a single output pays the merchant the full balance of the channel with a time lock that allows for customer dispute

**Channel Operation Assumptions.**
 - Channels funded by the customer alone and dual-funded channels are both supported.
 - Either the customer or the merchant can initiate channel closing.
 - If the customer initiates closing, then the merchant can dispute the closing transaction if they disagrees with the closing token in the closing transaction.
 - If the merchant initiates closing, the merchant posts a transaction claiming all the funds in the channel for themselves with a timelock. This gives the customer the opportunity to post their own valid closing transaction with the current channel balances. If the customer posts their own closing transaction, the merchant has an additional opportunity to dispute if necessary.

1.1 Customer and Merchant Signing Keys
-------------

The customer and the merchant both have key pairs from a suitable signature scheme. These are denoted as:
``<cust-pk>, <cust-sk>`` and 
``<merch-pk>, <merch-sk>``, respectively, where ``pk`` stands for "public key" and ``sk`` stands for the corresponding "secret key".

The merchant must be able to issue blind signatures, so they have an additional keypair; this keypair is denoted as:
``<MERCH-PK>, <MERCH-SK>``.

The customer key pair is specific to the channel and must not be reused. The merchant key pair is long term and should be used for all customer channels. 

1.2 Wallets
-------------
A Bolt channel allows a customer to make or receive a sequence of payments off chain. These payments are tracked and validated using a sequence of *wallets*. A wallet consists of the customer's public key (which ties the wallet to the channel), a wallet-specific public key (which can be from any suitable signature scheme), denoted ``<wpk>``, and the current customer and merchant balances.

After each payment, the customer receives an updated wallet and blind signatures from the merchant on the wallet contents allowing channel close as specified below.

1.2 Opening a Channel: Overview
-------------
To open a channel, the customer and merchant exchange key information and set the channel token ``<channel-token> = <<cust-pk> <merch-pk> <MERCH-PK>>``. 

They agree on their respective initial balances ``initial-cust-balance`` and ``initial-merch-balance``.

The customer picks an inital wallet public key ``<wpk>``.

The customer and merchant escrow the necessary funds in a funding transaction, denoted ``escrow-tx``. 

1.3 Closing a Channel: Overview
-------------

A customer should be able to close the channel by posting a *closing token* ``close-token``, which is a blind signature from the merchant under ``<MERCH-PK>`` on a special closing wallet that contains ``<<cust-pk>, <wpk>, <balance-cust>, <balance-merch>, CLOSE>``. We use ``cust-close-tx`` to denote the transaction posted by the customer to initiate channel closure.

A merchant should be able to close the channel by either posting a special closing transaction ``merch-close-tx`` (detailed in Section 2.3.2) or, if the customer posts an outdated version of their closing token, a signed revocation token, ``rev-token`` as detailed below. The revocation token ``rev-token`` is a signature under the wallet public key ``<wpk>`` on the special revocation message ``<<wpk> || REVOKED>``. The transaction posted by the merchant to dispute is denoted ``dispute-tx``.

The customer and merchant may also negotiate off-chain to form a *mutual close transaction*, ``mutual-close-tx``. Off-chain collaboration to create ``mutual-close-tx`` reduces the required number of on-chain transactions and eliminates the time delays.

2. Transparent/Shielded Tx: Using T/Z-addresses and WTPs
-------------

We assume the following specific features are present:

(1) Support for whitelisted transparent programs (WTPs) that enables 2-of-2 multi-sig style transactions
(2) Can specify absolute lock time in transaction
(3) Can specify relative lock time in transparent program
(4) Can specify shielded inputs and outputs
(5) A non-SegWit approach that fixes transaction malleability
(6) ``OP_BOLT`` logic expressed as WTPs. We will use the Bolt WTPs defined in section 2.1: ``open-channel``, ``cust-close``, and ``merch-close``.

**Privacy Limitations**. The aggregate balance of the channel will be revealed in the funding transaction ``escrow-tx``. Similarly, the final splitting of funds will be revealed to the network. However, for channel opening and closing, the identity of the participants remains hidden. Channel opening and closing will also be distinguishable on the network due to use of WTPs.

**Channel Opening**. The funding transaction ``escrow-tx`` spends ZEC from one or more shielded addresses to a transparent output that is encumbered by a Bolt transparent program. See Section 2.1 for what the funding transaction looks like when instantiated using WTPs.

2.1 Bolt WTPs
--------------

Transparent programs take as input a ``predicate``, ``witness``, and ``context`` and then output a ``True`` or ``False`` on the stack. Bolt-specific transparent programs are deterministic and any malleation of the ``witness`` will result in a ``False`` output. The WTPs are as follows:

1. ``open-channel`` program. The purpose of this WTP is to encumber the funding transaction such that either party may initiate channel closing as detailed above in Section 1.3. The program is structured as follows:
	a. ``predicate``: The predicate consists of ``<<channel-token> <merch-close-address>>``, where ``<channel-token> = <<cust-pk> <merch-pk> <MERCH-PK>>`` contains three public keys, one for the customer and two for the merchant, and an address ``<merch-close-address>`` for the merchant at which to receive funds from a customer-initiated close.
	
	b. ``witness``: The witness is defined as follows:
	
		1. ``<balance-cust> <balance-merch> <cust-sig> <merch-sig>``
 		2. ``<balance-cust> <balance-merch> <cust-sig> <wpk> <closing-token>``
 	c. ``verify_program`` behaves as follows:
	
		1. If witness is of the first type, check that 2 new outputs are created, with the specified balances (unless one of the balances is zero), and that the signatures verify.
		2. If witness is of second type, check that 2 new outputs are created (unless one of the balances is zero), with the specified balances:
		
			+ one paying ``<balance-merch>`` to ``<merch-close-address>`` 
			+ one paying a cust_close WTP containing ``<wallet> = <<wpk> <balance-cust> <balance-merch>>``  and ``<channel-token>`` 
			Also check that ``<cust-sig>`` is a valid signature and that ``<closing-token>`` contains a valid signature under ``<MERCH-PK>`` on ``<<cust-pk> <wpk> <balance-cust> <balance-merch> CLOSE>``.

2. ``cust-close`` program. The purpose of this WTP is to allow the customer to initiate channel closure as specified in Section 1.3. The program is specified as follows:

	a. ``predicate``: ``<wallet> <channel-token>``, where
	
		1. ``<wallet> = <<wpk> <balance-cust> <balance-merch>>``, and 
 		2. ``<channel-token> = <<cust-pk> <merch-pk> <MERCH-PK>>``.
	b. ``witness``: The witness is defined as one of the following:
	
		1. ``<cust-sig>``
		2. ``<merch-sig> <address> <rev-token>``
	c. ``verify_program`` behaves as follows:
	
		1. If witness is of the first type, check that ``<cust-sig>`` is valid and a relative timeout has been met
		2. If witness is of second type, check that 1 output is created paying ``<balance-merch + balance-cust>`` to ``<address>``. Also check that ``<merch-sig>`` is a valid signature on ``<address> <rev-token>`` and that ``<rev-token>`` contains a valid signature under ``<wpk>`` on ``<<wpk> || REVOKED>``.

3. ``merch-close``. The purpose of this WTP is to allow a merchant to initiate channel closure as specified in Section 1.3.

	a. ``predicate``: ``<channel-token> <merch-close-address>``.
	b. ``witness`` is defined as one of the following:
	
		1. ``<merch-sig>``
		2. ``<cust-sig> <<wpk> <balance-cust> <balance-merch>> <closing-token>``
		3. ``verify_program`` behaves as follows:
		
			1. If witness is of the first type, check that <merch-sig> is valid and a relative timeout has been met
			2. If witness is of second type, check that 2 new outputs are created (unless one of the balances is zero), with the specified balances:
			
				+ one paying ``<balance-merch>`` to ``<merch-close-address>`` 
 				+ one paying a ``cust_close`` WTP containing ``<wallet> = <<wpk> <balance-cust> <balance-merch>>``  and ``<channel-token>`` 
				
				Also check that ``<cust-sig>`` is a valid signature and that ``<closing-token>`` contains a valid signature under ``<MERCH-PK>`` on ``<<cust-pk> <wpk> <balance-cust> <balance-merch> CLOSE>``.


2.2 Funding Transaction
-------------
The funding transaction is by default funded by only one participant, the customer. We will be extending the protocol to allow for dual-funded channels.

This transaction has 2 shielded inputs (but can be up to some N) and 1 transparent output with a WTP and the predicate is the customer and merchant public keys. Note that the customer can specify as many shielded inputs as necessary to fund the channel sufficiently (limited only by the overall transaction size).

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
* ``tx_out``: (via a transparent program)

  - ``scriptPubKey``: ``PROGRAM PUSHDATA( <open-channel> || <<channel-token> || <merch-close-addr>> )``

where the ``<open-channel>`` type corresponds to the following logic (expressed in ``Script`` for convenience):

	OP_IF
	  2 <cust-pubkey> <merch-pubkey> 2 OP_CHECKMULTISIG
	OP_ELSE
	  <cust-pubkey> OP_CHECKSIGVERIFY 1 OP_BOLT
	OP_ENDIF

* ``bindingSig``: a signature that proves that (1) the total value spent by Spend transfers - Output transfers = value balance field.

The customer (in collaboration with the merchant) creates their initial closing transaction before sending the funding transaction to the network (since  the customer needs to know they can get their money back). Once both customer and merchant closing transactions have been created, the customer should broadcast the funding transaction and waits for the network to confirm the transaction. After the transaction has been confirmed, the payment channel is established.

2.3 Closing Transactions
-------------
2.3.1 Customer closing transaction
----
The customer closing transaction is generated by the customer during the channel establishment but is not broadcast to the network. The customer's closing transaction (below) contains two outputs: (1) an output that can be spent immediately by the merchant and (2) another output that can be spent by either the customer after a relative timeout or the merchant with a revocation token. This approach allows the merchant to see the customer's closing transaction and spend the output with a revocation token if the customer posted an outdated closure token.

The customer's closing transaction is described below.

* ``version``: specify version number
* ``groupid``: specify group id
* ``locktime``: should be set such that closing transactions can be included in a current block.
* ``txin`` count: 1

   - ``txin[0]`` outpoint: references the funding transaction txid and output_index
   - ``txin[0]`` script bytes: 0
   - ``txin[0]`` script sig: ``PROGRAM PUSHDATA( <open-channel> || <<customer> || <close-token> || <cust-sig>> )``

* ``txout`` count: 2
* ``txouts``:

  * ``to_customer``: a timelocked WTP output sending funds back to the customer with a time delay.
      - ``amount``: balance paid back to customer
      - ``nSequence: <time-delay>``
      - ``scriptPubKey``: ``PROGRAM PUSHDATA( <close-channel> || <<cust-pubkey> || <merch-pubkey> || <revocation-pubkey>>  )``

  * ``to_merchant``: a P2PKH to merch-pubkey output (sending funds back to the merchant), i.e.
      * ``scriptPubKey``: ``0 <20-byte-key-hash of merch-pubkey>``
      * ``amount``: balance paid to merchant
      * ``nSequence``: 0

To redeem the ``to_customer`` output, the customer presents a ``scriptSig`` with the customer signature after a time delay as follows:

	``PROGRAM PUSHDATA( <close-channel> || <<customer> || <cust-sig> || <block-height>> )``

where the ``witness`` consists of a first byte ``0x0`` to indicate customer spend followed by the customer signature and the current block height (used to ensure that timeout reached) and where the ``<cust-close>`` type corresponds to the following logic (expressed in ``Script`` for convenience):

	``OP_IF``
	  ``<revocation-pubkey> <merch-pubkey> 2 OP_BOLT``
	``OP_ELSE``
	  ``<time-delay> OP_CSV OP_DROP <cust-pubkey> OP_CHECKSIGVERIFY``
	``OP_ENDIF``

If the customer posted an outdated closing token, the merchant can redeem the ``to_customer`` output by posting a transaction with the following ``scriptSig``:

	``PROGRAM PUSHDATA( <close-channel> || <<merchant> || <merch-sig> || <revocation-token>> )``

where the ``witness`` consists of a first byte ``0x1`` to indicate merchant spend followed by the merchant signature and the revocation token.

2.3.2 Merchant closing transaction
----
The merchant can create their own initial closing transaction as follows and obtain the customer signature during the establishment phase.

* ``version``: specify version number
* ``groupid``: specify group id
* ``locktime``: should be set such that closing transactions can be included in a current block.
* ``txin`` count: 1

   - ``txin[0]`` outpoint: references the funding transaction txid and output_index
   - ``txin[0]`` script bytes: 0
   - ``txin[0]`` script sig: ``PROGRAM PUSHDATA( <open-channel> || <<merchant> || <cust-sig> || <merch-sig>> )``

* ``txout`` count: 1
* ``txouts``:

  * ``to_merchant``: a timelocked WTP output sending all the funds in the channel back to the merchant with a time delay
      - ``amount``: full balance paid back to merchant
      - ``nSequence: <time-delay>``
      - ``scriptPubKey``: ``PROGRAM PUSHDATA( <close-channel> || <<merch-pubkey> || <cust-pubkey>> )``

where the ``<close-channel>`` type corresponds to the following logic (expressed in ``Script`` for convenience):

		OP_IF
	  	  <cust-pubkey> OP_CHECKSIGVERIFY 1 OP_BOLT
		OP_ELSE
		  <time-delay> OP_CSV OP_DROP <merch-pubkey> OP_CHECKSIGVERIFY
		OP_ENDIF

After each payment on the channel, the customer obtains a closing token for the updated channel balance and provides the merchant a revocation token for the previous state along with the associated wallet public key (this invalidates the pub key). If the customer initiated closing, the merchant can use the revocation token to spend the funds of the channel if the customer posts an outdated closing transaction.

2.4 Channel Closing
-------------
To close the channel, the customer can initiate by posting the most recent closing transaction that spends from the multi-signature transparent address with inputs that satisfies the script and the ``OP_BOLT`` opcode in mode 1. This consists of a closing token (i.e., merchant signature on the wallet state) or an opening of the initial wallet commitment (if there were no payments on the channel).

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

The merchant can initiate closing by posting the initial closing transaction from establishing the channel that pays the merchant the full balance of the channel with a time lock that allows for customer dispute. After the time delay, the merchant can then post a separate standard transaction that moves the claimed funds to a shielded address.

Rationale
---------

TODO: include a section explaining how any potential sources of malleability are handled.

Reference Implementation
========================

We are currently working on a reference implementation based on section 2 in a fork of Zcash here: https://github.com/boltlabs-inc/zcash.

References
==========

.. [#RFC2119] `Key words for use in RFCs to Indicate Requirement Levels <https://tools.ietf.org/html/rfc2119>`_
.. [#bolt-paper]  `Bolt: Anonymous Payment Channels for Decentralized Currencies <https://eprint.iacr.org/2016/701>`_
.. [#wtp-programs]  `ZIP XXX: Whitelisted Transparent Programs (Draft) <https://github.com/zcash/zips/pull/248>`_
