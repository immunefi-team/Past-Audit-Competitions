# #37153 \[BC-Insight] Malicious validator can bring down honest nodes

**Submitted on Nov 26th 2024 at 21:40:06 UTC by @br0nz3p1ck4x3 for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37153
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/prysmaticlabs/prysm>
* **Impacts:**
  * Shutdown of less than 10% of network processing nodes without brute force actions, but does not shut down the network

## Description

Hey friends,

We stumbled upon the following during our research & hope this will add value to the Prysm team.

## Description

If we take a look at `validateBlsToExecutionChange()`, we can see the following:

```go
func (s *Service) validateBlsToExecutionChange(ctx context.Context, pid peer.ID, msg *pubsub.Message) (pubsub.ValidationResult, error) {
//..
	// Validate that the execution change object is valid.
	_, err = blocks.ValidateBLSToExecutionChange(st, blsChange)
//..
}
```

`ValidateBLSToExecutionChange()` is called.

Now, let's take a look at `ValidateBLSToExecutionChange()`:

```go
func ValidateBLSToExecutionChange(st state.ReadOnlyBeaconState, signed *ethpb.SignedBLSToExecutionChange) (*ethpb.Validator, error) {
	if signed == nil {
		return nil, errNilSignedWithdrawalMessage
	}
	message := signed.Message
	if message == nil {
		return nil, errNilWithdrawalMessage
	}

	val, err := st.ValidatorAtIndex(message.ValidatorIndex)
	if err != nil {
		return nil, err
	}
	cred := val.WithdrawalCredentials
	if cred[0] != params.BeaconConfig().BLSWithdrawalPrefixByte {
		return nil, errInvalidBLSPrefix
	}

	// hash the public key and verify it matches the withdrawal credentials
	fromPubkey := message.FromBlsPubkey
	hashFn := ssz.NewHasherFunc(hash.CustomSHA256Hasher())
	digest := hashFn.Hash(fromPubkey)
	if !bytes.Equal(digest[1:], cred[1:]) {
		return nil, errInvalidWithdrawalCredentials
	}
	return val, nil
}
```

There is an out of bounds read here that can be triggered by a malicious validator:

```go
func ValidateBLSToExecutionChange(st state.ReadOnlyBeaconState, signed *ethpb.SignedBLSToExecutionChange) (*ethpb.Validator, error) {
//..
	cred := val.WithdrawalCredentials
	if cred[0] != params.BeaconConfig().BLSWithdrawalPrefixByte {
		return nil, errInvalidBLSPrefix
	}
//..
}
```

If the malicious validator populates the `WithdrawalCredentials` with a `nil` value, an out of bounds read happens which will lead to a panic.

In psuedo-code, the malicious validator should:

* Send a `ethpb.SignedBLSToExecutionChange` message to the honest validator
* Make sure the `WithdrawalCredentials` is of a`nil` value
* Out of bounds will be triggered

Please check the Proof of Concept below.

## Severity Rationale

This attack is limited to the `p2p` nodes that a node is connected too thus, according to the Immunefi Severity Classification we decided to submit this as a Medium Severity issue.

## Recommended Patch

Apply the following patch to `ValidateBLSToExeuctionChange()`:

````diff
```go
func ValidateBLSToExecutionChange(st state.ReadOnlyBeaconState, signed *ethpb.SignedBLSToExecutionChange) (*ethpb.Validator, error) {
//..
	cred := val.WithdrawalCredentials
+	if cred == nil {
+		return nil, errCredIsNil
+	}
	if cred[0] != params.BeaconConfig().BLSWithdrawalPrefixByte {
		return nil, errInvalidBLSPrefix
	}
//..
}
````

## Proof of Concept

## Proof of Concept

Apply the following patch to`validate_bls_to_execution_change_test.go`:

```diff
301:				newVal := val.Copy()
-					newVal.WithdrawalCredentials = newCreds
+					newVal.WithdrawalCredentials = nil
```

Now, do the following:

* `cd beacon-chain/sync`
* `go test -timeout 30s -run ^TestService_ValidateBlsToExecutionChange`

The output will be the following:

```zsh
--- FAIL: TestService_ValidateBlsToExecutionChange (0.38s)
    --- FAIL: TestService_ValidateBlsToExecutionChange/Invalid_Credentials_in_State (0.06s)
panic: runtime error: index out of range [0] with length 0 [recovered]
	panic: runtime error: index out of range [0] with length 0

goroutine 1210 [running]:
testing.tRunner.func1.2({0x1042ad3a0, 0x140015125e8})
	/opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1632 +0x1bc
testing.tRunner.func1()
	/opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1635 +0x334
panic({0x1042ad3a0?, 0x140015125e8?})
	/opt/homebrew/Cellar/go/1.23.2/libexec/src/runtime/panic.go:785 +0x124
github.com/prysmaticlabs/prysm/v5/beacon-chain/core/blocks.ValidateBLSToExecutionChange({0x10442bc18?, 0x14002791188?}, 0x10392d96a?)
	/Users/bronze_pickaxe/x/prysm/beacon-chain/core/blocks/withdrawals.go:104 +0x1f8
github.com/prysmaticlabs/prysm/v5/beacon-chain/sync.(*Service).validateBlsToExecutionChange(0x14001c9e508, {0x104403c70, 0x1400152ecd0}, {0x1038f08d3, 0x6}, 0x14004a6bea8)
	/Users/bronze_pickaxe/x/prysm/beacon-chain/sync/validate_bls_to_execution_change.go:52 +0x178
github.com/prysmaticlabs/prysm/v5/beacon-chain/sync.TestService_ValidateBlsToExecutionChange.func10(0x14001ec8680)
	/Users/bronze_pickaxe/x/prysm/beacon-chain/sync/validate_bls_to_execution_change_test.go:449 +0x44c
testing.tRunner(0x14001ec8680, 0x1400139a0c0)
	/opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1690 +0xe4
created by testing.(*T).Run in goroutine 207
	/opt/homebrew/Cellar/go/1.23.2/libexec/src/testing/testing.go:1743 +0x314
exit status 2
FAIL	github.com/prysmaticlabs/prysm/v5/beacon-chain/sync	1.048s
```

Out of bounds error leading to panic.
