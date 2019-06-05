4. Bitcoin Compatible: Using T-address and Scripting Opcodes
-------------
We assume the following features are present:

(a) ``OP_CLTV`` - absolute lock time
(b) ``OP_CSV`` - relative lock time
(c) 2-of-2 multi-sig transparent address support
(d) Transaction non-malleability for t-addresses
(e) ``OP_BOLT`` opcode: takes two inputs as argument (a mode and a serialized token) and outputs a `True` or `False` on the stack. Same description from Section 2.

**Note**: We assume P2WSH as it enforces transaction non-malleability and allows unconfirmed transaction dependency chains. Another approach to transaction non-malleability would be acceptable.

**Privacy Limitations**. With T-addresses, we give up the ability to hide the initial balance for the funding transaction and final balances when closing the channel. Channel opening will be distinguishable on the network due to use of ``OP_BOLT`` opcodes.

**Channel Opening**. A channel is established when two parties successfully lock up funds in a multi-sig transparent address on the blockchain. The funds remain spendable by the customer in a commitment transaction that closes the channel and splits the funds as indicated by the last invocation of the (off-chain) pay protocol. The merchant can close the channel using their own commitment transaction, which claims the entire channel balance while giving the customer time to post the appropriate commitment transaction for closing.

The customer and merchant first initialize the channel by generating their respective keypairs and computing the channel tokens for the initial wallet commitment.

The customer then creates a funding transaction that deposits ZEC to a 2-of-2 multi-signature transparent address using a pay-to-witness-script-hash (P2WSH) output (alternatively, a P2WPKH nested in a P2SH could work). The customer obtains a signature for the funding transaction and commitment transaction from the merchant. The customer can then post the funding transaction to the blockchain.

4.1 Funding Transaction
-------------
The funding transaction is by default funded by only one participant, the customer. This transaction is a P2WSH SegWit transaction. Here is a high-level of what the funding transaction would look like:

	witness: 0 <opbolt-mode> <<channel-token> <closing token>> <cust-sig> <merch-sig> <2 <cust-pubkey> <merch-pubkey> 2 OP_CHECKMULTISIGVERIFY OP_DUP OP_HASH160 <hash-of-channel-token> OP_EQUALVERIFY OP_BOLT>
	
	scriptSig: (empty)	
	scriptPubKey: 0 <32-byte-hash>

This is a standard SegWit P2WSH transaction. Note that the witness and empty ``scriptSig`` are provided by a subsequent transaction that spends the funding transaction output. The ``scriptPubKey`` of the funding transaction indicates that a witness script should be provided with a given hash; the ``witnessScript`` (≤ 10,000 bytes) is popped off the initial witness stack of a spending transaction and the SHA256 of witnessScript must match the 32-byte hash of the following:

	2 <cust-pubkey> <merch-pubkey> 2 OP_CHECKMULTISIGVERIFY	
	OP_DUP OP_HASH160 <hash-of-channel-token> OP_EQUALVERIFY OP_BOLT
	
4.2 Initial Wallet Commitment
-------------
This wallet commitement below is created first during channel initialization, but the customer does not broadcast to the network.

* ``version``: specify version number
* ``groupid``: specify group id
* ``locktime``: should be set so that the commitment can be included in current block 
* ``txin`` count: 1

  - ``txin[0]`` outpoint: ``txid`` and ``outpoint_index`` of the funding transaction
  - ``txin[0]`` script bytes: 0
  - ``txin[0]`` witness: ``0 <opbolt-mode> <<channel-token> <closing token> or <rev-token>> <cust-sig> <merch-sig> <2 <cust_fund_pubkey> <merch_fund_pubkey> 2 OP_CHECKMULTISIGVERIFY OP_DUP OP_HASH160 <hash-of-channel-token> OP_EQUALVERIFY OP_BOLT>``

* ``txouts``: 
* ``to_customer``: a timelocked (using ``OP_CSV``) version-0 P2WSH output sending funds back to the customer. So scriptPubKey is of the form ``0 <32-byte-hash>``. A customer node may create a transaction spending this output with:

  - ``nSequence: <time-delay>``
  - ``witness: <closing-token> <cust-sig> 0 <witnessScript>``
  - ``witness script:``
  
	OP_IF
	  # Merchant can spend if revocation token available
	  OP_2 <rev-pubkey> <merch-pubkey> OP_2
	OP_ELSE
	  # Customer must wait 
	  <time-delay> OP_CSV OP_DROP <cust-pubkey>
	OP_ENDIF
	OP_CHECKSIGVERIFY 

* ``to_merchant``: A P2WPKH to merch-pubkey output (sending funds back to the merchant), i.e.
   * ``scriptPubKey``: ``0 <20-byte-key-hash of merch-pubkey>``

Or, if a revoked commitment transaction is available, the merchant may spend the ``to_customer`` output with the above witness script and witness stack:

	3 <rev-token> 1 <witnessScript>
			
To spend ``to_merchant`` output, the merchant publishes a transaction with:
	
	witness: <merch-sig> <merch-pubkey> <witnessScript>

The merchant can create their own initial commitment transaction as follows.

* ``version``: specify version number
* ``groupid``: specify group id
* ``locktime``: should be set so that the commitment can be included in current block 
* ``txin`` count: 1

  - ``txin[0]`` outpoint: ``txid`` and ``outpoint_index`` of the funding transaction
  - ``txin[0]`` script bytes: 0
  - ``txin[0]`` witness: ``0 <opbolt-mode> <<channel-token> <closing token> or <rev-token>> <cust-sig> <merch-sig> <2 <cust_fund_pubkey> <merch_fund_pubkey> 2 OP_CHECKMULTISIGVERIFY OP_DUP OP_HASH160 <hash-of-channel-token> OP_EQUALVERIFY OP_BOLT>``

* ``txout`` count: 1
* ``txouts``: 

  * ``to_merchant``: a timelocked (using ``OP_CSV``) P2WSH output sending all the funds back to the merchant. So ``scriptPubKey`` is of the form ``0 <32-byte-hash>``.  
      - ``amount``: balance paid back to merchant
      - ``nSequence: <time-delay>``
      - ``witness: 1 <merch-sig> 0 <witnessScript>``
      - ``witnessScript``:
      
		OP_IF
	  	  OP_2 <closing-token> <cust-pubkey> OP_2
		OP_ELSE
		  <time-delay> OP_CSV OP_DROP <merchant-pubkey>
		OP_ENDIF
		OP_CHECKSIGVERIFY


4.3 Channel Closing
-------------
The customer initiates channel closing by posting a closing transaction that spends from the multi-signature address with a witness that satisfies the witnessScript and the ``OP_BOLT`` opcode in mode 1. This consists of a closing token (i.e., merchant signature on the wallet state) or an opening of the initial wallet commitment (if there were no payments on the channel via mode 2). 

Once the timeout has been reached, the customer can post a transaction that claims the output of the customer closing transaction to another output. Before the timeout, the merchant can claim the funds from the ``to_customer`` output by posting a revocation token (via mode 3), if they have one. The merchant can immediately claim the ``to_merchant`` output from the customer closing transaction by presenting their P2WPKH address.

The merchant can initiate closing by posting the initial commitment transaction (in Section 4.3) from establishing the channel that pays the merchant the full balance of the channel with a time lock that allows for customer dispute.
