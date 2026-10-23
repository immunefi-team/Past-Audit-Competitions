# 57021 bc insight lack of panic recovery in housekeeping goroutine creates potential for denial of service

**Submitted on Oct 22nd 2025 at 18:01:35 UTC by @rionnaldi for** [**Attackathon | VeChain Hayabusa Upgrade**](https://immunefi.com/audit-competition/vechain-hayabusa-upgrade-attackathon)

* **Report ID:** #57021
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/vechain/thor/compare/v2.3.2...release/hayabusa>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

### Brief/Intro

The lack of a `defer recover()` statement at the beginning of the `housekeeping` goroutine is a critical fragility. Long-running goroutines responsible for core node functions, especially those processing potentially malicious external input, must be resilient to panics. The `wash` function is a prime candidate for panics because it involves many complex interactions with the state and untrusted transaction data.

### Vulnerability Details

The `housekeeping` function runs as a long-lived goroutine, performing the critical task of periodically cleaning ("washing") the transaction pool. This `wash` operation is complex, involving the creation of new state objects and the validation of transactions against the state by calling `txObj.Executable()`. The `Executable` method in turn calls `resolved.BuyGas()`, which interacts with multiple native smart contracts.

The `housekeeping` goroutine does not implement a `recover` mechanism. If any part of the code within the `for` loop panics (e.g., due to a nil pointer dereference, an out-of-bounds slice access, or a bug in a native contract triggered by a malformed transaction), the panic will be unhandled. This will cause the `housekeeping` goroutine to crash and exit permanently, leaving the node without its transaction pool cleaning mechanism.

Vulnerable Code (in `txpool/tx_pool.go`):

```go
func (p *TxPool) housekeeping() {
	logger.Debug("enter housekeeping")
	defer logger.Debug("leave housekeeping")

	ticker := time.NewTicker(time.Second * 1)
	defer ticker.Stop()

	headSummary := p.repo.BestBlockSummary()

	for {
		// NO RECOVER MECHANISM HERE
		select {
		case <-p.ctx.Done():
			return
		case <-ticker.C:
            // ... complex logic that calls wash() ...
            // A panic inside wash() will crash this goroutine.
				executables, removedLegacy, removedDynamicFee, err := p.wash(headSummary)
        }
    }
}
```

### Impact Details

* Permanent Denial of Service on Transaction Processing: As demonstrated in the PoC, once the `housekeeping` goroutine crashes, the node permanently loses its ability to clean the transaction pool. Expired transactions will accumulate indefinitely. The pool will fill with these "zombie" transactions, eventually reaching its capacity (`options.Limit`). At this point, the node will reject all new incoming transactions, effectively taking it offline in terms of transaction processing and propagation. The only remediation is to restart the node.
* Resource Exhaustion: The accumulation of unswept transactions in memory will lead to a gradual increase in the node's memory footprint, which can eventually lead to an Out-Of-Memory (OOM) crash, causing the entire node process to terminate.
* Network Degradation: If an attacker can successfully execute this attack against multiple nodes, particularly Authority Masternodes, it could significantly impair the transaction processing capability of the entire network, leading to increased confirmation times and reduced throughput.

## References

<https://github.com/vechain/thor/blob/release/hayabusa/txpool/tx\\_pool.go>

## Link to Proof of Concept

<https://gist.github.com/rionnaldi/fb50d3b13e3aad311cf37d1f63c89451>

## Proof of Concept

{% stepper %}
{% step %}

### Attack overview

1. State Poisoning: The attacker first submits a transaction that writes a malformed RLP value (specifically `0x01`, which is invalid as a vp-encoded string) to a storage slot associated with a contract. The PoC simulates this by directly modifying the state database to place this value in the `user` credit slot of the `prototype` contract.
2. Triggering the Panic: The attacker submits a second transaction to the target node's `txpool`. This transaction is crafted to trigger a read of the poisoned storage slot during validation (e.g., by calling a function on the contract that checks the user's credit).
3. Goroutine Crash: When the `housekeeping` goroutine's `wash` function processes this second transaction, it calls `txObj.Executable()`. This leads to the `prototype` contract attempting to decode the malformed RLP data from storage. The RLP parser panics due to a "slice bounds out of range" error.
4. Permanent DoS: The panic is unhandled, crashing the `housekeeping` goroutine. The node's transaction pool is no longer cleaned of expired, invalid, or successfully processed transactions.
   {% endstep %}

{% step %}

### Steps to reproduce

1. Copy the code from provided gist.
2. Create a file named `h01_poc_test.go` inside the `txpool` folder, then paste the code.
3. Run the code with:

```bash
go test ./txpool -run TestPanicRecoveryInHousekeeping -v
```

{% endstep %}

{% step %}

### Observed behavior (Expected vs Actual)

Expected: Housekeeping continues running and cleans expired transactions after processing the malicious transaction.

Actual: Housekeeping crashes due to an unhandled panic, leaving expired transactions in the pool and causing progressive resource depletion and eventual denial of service until node restart.
{% endstep %}
{% endstepper %}

<details>

<summary>Expected test output from PoC</summary>

```
=== RUN   TestPanicRecoveryInHousekeeping
    h01_poc_test.go:38: Initial pool size: 0 transactions
    h01_poc_test.go:51: Malicious RLP payload: 01 (triggers slice bounds panic)
    h01_poc_test.go:59: Target contract: 0x435933c8064b4ae76be665428e0307ef2ccfbd68
    h01_poc_test.go:60: User credit storage key: 307837383965353533653563323866303261663631633533333834633232636663303134653633343533643461623964383662643432323732323937616433336364
    h01_poc_test.go:85: Block 1 mined with poisoned state root: 307839343161363365646531636330613339646433356666663263303266623833613766663639353364373362396135653462396334333633323436376565616463
    h01_poc_test.go:107: Exploit transaction ID: 0x50172a3ffdaa1e6ab0cfd549516e8ba82a889fa59a0a26509325922c1824c156
    h01_poc_test.go:112: Waiting for housekeeping to process exploit transaction and crash...
    h01_poc_test.go:122: Adding test transactions that should be cleaned up by housekeeping...
    h01_poc_test.go:139: Added test transaction 1: 0xb8e95546bcae7691a3498e648b68b193440b4d151a9a5e6b5ef4ccc88bece998
    h01_poc_test.go:139: Added test transaction 2: 0x39c29c05267f8e110350aa5b1b5c46ba8dfb7454f8a3bfd3803acc7661b8ac4e
    h01_poc_test.go:139: Added test transaction 3: 0xacef64fffe2fd8f0889bce699bf71efef6c3e1f1f8ac2f53562f3594efb22391
    h01_poc_test.go:139: Added test transaction 4: 0xb6143a21d889d8e3a3b265ee33bfc38e0ec10190c67b97d95338f81265fa8a89
    h01_poc_test.go:139: Added test transaction 5: 0xf0523eebc071786edaaf68044c16e47b4f86af2e47e6c552752d611b3ad0e688
    h01_poc_test.go:143: Pool size before expiry: 6 transactions
    h01_poc_test.go:146: Mining block to expire test transactions...
    h01_poc_test.go:149: Block advanced - test transactions should now be expired
    h01_poc_test.go:152: Waiting for housekeeping to process expired transactions...
    h01_poc_test.go:160: Pool size after expiry + wait: 6 transactions
    h01_poc_test.go:176: Expired transactions still in pool: 5/5
    h01_poc_test.go:178: Zombie transactions: [0x39c29c... 0xacef64... 0xb6143a... 0xf0523e... 0xb8e955...]
--- PASS: TestPanicRecoveryInHousekeeping (6.01s)
PASS
```

</details>
