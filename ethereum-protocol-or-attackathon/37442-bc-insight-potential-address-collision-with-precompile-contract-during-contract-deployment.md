# #37442 \[BC-Insight] Potential Address Collision with Precompile Contract During Contract Deployment

**Submitted on Dec 4th 2024 at 20:27:45 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37442
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

Within the contract deployment in the Erigon, it misses the check that the newly created contract address does not belong to the precompile contract addresses, which leads to the address collision.

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/tree/v2.61.0-beta1>

The function `create()` is utilized to create a contract via contract deployment transaction, opcode Create and Create2.

<https://github.com/erigontech/erigon/blob/v2.61.0-beta1/core/vm/evm.go#L353>

```
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gasRemaining uint64, value *uint256.Int, address libcommon.Address, typ OpCode, incrementNonce bool, bailout bool) ([]byte, libcommon.Address, uint64, error) {
	var ret []byte
	var err error
	var gasConsumption uint64
	depth := evm.interpreter.Depth()


	if evm.config.Debug {
		if depth == 0 {
			evm.config.Tracer.CaptureStart(evm, caller.Address(), address, false /* precompile */, true /* create */, codeAndHash.code, gasRemaining, value, nil)
			defer func() {
				evm.config.Tracer.CaptureEnd(ret, gasConsumption, err)
			}()
		} else {
			evm.config.Tracer.CaptureEnter(typ, caller.Address(), address, false /* precompile */, true /* create */, codeAndHash.code, gasRemaining, value, nil)
			defer func() {
				evm.config.Tracer.CaptureExit(ret, gasConsumption, err)
			}()
		}
	}


	// Depth check execution. Fail if we're trying to execute above the
	// limit.
	if depth > int(params.CallCreateDepth) {
		err = ErrDepth
		return nil, libcommon.Address{}, gasRemaining, err
	}
	if !evm.Context.CanTransfer(evm.intraBlockState, caller.Address(), value) {
		if !bailout {
			err = ErrInsufficientBalance
			return nil, libcommon.Address{}, gasRemaining, err
		}
	}
	if incrementNonce {
		nonce := evm.intraBlockState.GetNonce(caller.Address())
		if nonce+1 < nonce {
			err = ErrNonceUintOverflow
			return nil, libcommon.Address{}, gasRemaining, err
		}
		evm.intraBlockState.SetNonce(caller.Address(), nonce+1)
	}
	// We add this to the access list _before_ taking a snapshot. Even if the creation fails,
	// the access-list change should not be rolled back
	if evm.chainRules.IsBerlin {
		evm.intraBlockState.AddAddressToAccessList(address)
	}
	// Ensure there's no existing contract already at the designated address
	contractHash := evm.intraBlockState.ResolveCodeHash(address)
	if evm.intraBlockState.GetNonce(address) != 0 || (contractHash != (libcommon.Hash{}) && contractHash != emptyCodeHash) {
		err = ErrContractAddressCollision
		return nil, libcommon.Address{}, 0, err
	}
	// Create a new account on the state
	snapshot := evm.intraBlockState.Snapshot()
	evm.intraBlockState.CreateAccount(address, true)
	if evm.chainRules.IsSpuriousDragon {
		evm.intraBlockState.SetNonce(address, 1)
	}
	evm.Context.Transfer(evm.intraBlockState, caller.Address(), address, value, bailout)


	// Initialise a new contract and set the code that is to be used by the EVM.
	// The contract is a scoped environment for this execution context only.
	contract := NewContract(caller, address, value, gasRemaining, evm.config.SkipAnalysis)
	contract.SetCodeOptionalHash(&address, codeAndHash)


	if evm.config.NoRecursion && depth > 0 {
		return nil, address, gasRemaining, nil
	}


	ret, err = run(evm, contract, nil, false)


	// EIP-170: Contract code size limit
	if err == nil && evm.chainRules.IsSpuriousDragon && len(ret) > evm.maxCodeSize() {
		// Gnosis Chain prior to Shanghai didn't have EIP-170 enabled,
		// but EIP-3860 (part of Shanghai) requires EIP-170.
		if !evm.chainRules.IsAura || evm.config.HasEip3860(evm.chainRules) {
			err = ErrMaxCodeSizeExceeded
		}
	}


	// Reject code starting with 0xEF if EIP-3541 is enabled.
	if err == nil && evm.chainRules.IsLondon && len(ret) >= 1 && ret[0] == 0xEF {
		err = ErrInvalidCode
	}
	// if the contract creation ran successfully and no errors were returned
	// calculate the gas required to store the code. If the code could not
	// be stored due to not enough gas set an error and let it be handled
	// by the error checking condition below.
	if err == nil {
		createDataGas := uint64(len(ret)) * params.CreateDataGas
		if contract.UseGas(createDataGas) {
			evm.intraBlockState.SetCode(address, ret)
		} else if evm.chainRules.IsHomestead {
			err = ErrCodeStoreOutOfGas
		}
	}


	// When an error was returned by the EVM or when setting the creation code
	// above we revert to the snapshot and consume any gas remaining. Additionally
	// when we're in homestead this also counts for code storage gas errors.
	if err != nil && (evm.chainRules.IsHomestead || err != ErrCodeStoreOutOfGas) {
		evm.intraBlockState.RevertToSnapshot(snapshot)
		if err != ErrExecutionReverted {
			contract.UseGas(contract.Gas)
		}
	}


	// calculate gasConsumption for deferred captures
	gasConsumption = gasRemaining - contract.Gas


	return ret, address, contract.Gas, err
}
```

However, there is no check to ensure the newly created `address` passed into the function is not one of the precompile contract addresses. The precompile contract addresses are supposed to be reserved only for the precompile contract.

The newly created address is created either via `Create()` or `Create2()`:

<https://github.com/erigontech/erigon/blob/v2.61.0-beta1/core/vm/evm.go#L474>

```
// Create creates a new contract using code as deployment code.
// DESCRIBED: docs/programmers_guide/guide.md#nonce
func (evm *EVM) Create(caller ContractRef, code []byte, gasRemaining uint64, endowment *uint256.Int, bailout bool) (ret []byte, contractAddr libcommon.Address, leftOverGas uint64, err error) {
	contractAddr = crypto.CreateAddress(caller.Address(), evm.intraBlockState.GetNonce(caller.Address()))
	return evm.create(caller, &codeAndHash{code: code}, gasRemaining, endowment, contractAddr, CREATE, true /* incrementNonce */, bailout)
}


// Create2 creates a new contract using code as deployment code.
//
// The different between Create2 with Create is Create2 uses keccak256(0xff ++ msg.sender ++ salt ++ keccak256(init_code))[12:]
// instead of the usual sender-and-nonce-hash as the address where the contract is initialized at.
// DESCRIBED: docs/programmers_guide/guide.md#nonce
func (evm *EVM) Create2(caller ContractRef, code []byte, gasRemaining uint64, endowment *uint256.Int, salt *uint256.Int, bailout bool) (ret []byte, contractAddr libcommon.Address, leftOverGas uint64, err error) {
	codeAndHash := &codeAndHash{code: code}
	contractAddr = crypto.CreateAddress2(caller.Address(), salt.Bytes32(), codeAndHash.Hash().Bytes())
	return evm.create(caller, codeAndHash, gasRemaining, endowment, contractAddr, CREATE2, true /* incrementNonce */, bailout)
}
```

Though this check does not seem to be specified in the execution specification and it is unlikely to occur in a short term due to the hardness of hash collision, it is necessary to ensure the logic correctness of the execution. Note that the revm (used in Ethereum client Reth) has such check:

<https://github.com/bluealloy/revm/blob/v50/crates/revm/src/context/evm\\_context.rs#L320>

```
        // created address is not allowed to be a precompile.
        if self.precompiles.contains(&created_address) {
            return return_error(InstructionResult::CreateCollision);
        }
```

## Impact Details

In case that the newly created contract address belongs to the precompile contract addresses, the contract will never be functional.

## References

* <https://github.com/erigontech/erigon/tree/v2.61.0-beta1>
* <https://github.com/bluealloy/revm/tree/v50>

## Proof of Concept

## Proof of Concept

For simplicity, we modify the code inside the `Create()` to mimic that the newly created address is `0x08` (a precommplile contract address) and test on the `Create()` .

```
func (evm *EVM) Create(caller ContractRef, code []byte, gasRemaining uint64, endowment *uint256.Int, bailout bool) (ret []byte, contractAddr libcommon.Address, leftOverGas uint64, err error) {
	//contractAddr = crypto.CreateAddress(caller.Address(), evm.intraBlockState.GetNonce(caller.Address()))
	contractAddr = libcommon.BytesToAddress([]byte{0x8}) // assume the newly created address is 0x08 precompile contract address
	return evm.create(caller, &codeAndHash{code: code}, gasRemaining, endowment, contractAddr, CREATE, true /* incrementNonce */, bailout)
}
```

Run the following test case to show that the `0x08` address is allowed to be deployed with contract:

```
package vm


import (
	"fmt"
	"github.com/ledgerwatch/erigon-lib/kv/memdb"
	"testing"


	libcommon "github.com/ledgerwatch/erigon-lib/common"
	"github.com/ledgerwatch/erigon/core/vm/evmtypes"
	"github.com/ledgerwatch/erigon/params"


	"github.com/holiman/uint256"
	"github.com/ledgerwatch/erigon/core/state"
	"pgregory.net/rapid"
)


func TestCreateTx(t *testing.T) {
	t.Parallel()
	code := []byte{byte(PUSH1), 0, byte(PUSH1), 0, byte(RETURN)}
	_, tx := memdb.NewTestTx(t)
	r, _ := state.NewPlainStateReader(tx), state.NewPlainStateWriter(tx, nil, 0)


	address := libcommon.BytesToAddress([]byte("01"))
	statedb := state.New(r)
	statedb.CreateAccount(address, false)
	statedb.SetCode(address, code)
	statedb.SetState(address, &libcommon.Hash{}, *uint256.NewInt(0))
	statedb.AddBalance(address, uint256.NewInt(10000))
	fmt.Printf("balance is %d\n", statedb.GetBalance(address))


	vmctx := evmtypes.BlockContext{
		CanTransfer: func(evmtypes.IntraBlockState, libcommon.Address, *uint256.Int) bool { return true },
		Transfer:    func(evmtypes.IntraBlockState, libcommon.Address, libcommon.Address, *uint256.Int, bool) {},
	}


	vmenv := NewEVM(vmctx, evmtypes.TxContext{}, statedb, params.TestChainConfig, Config{})


	ret, addr, gas, err := vmenv.Create(AccountRef(address), code, 10000, uint256.NewInt(0), true)


	if err != nil {
		fmt.Printf("deployment err: %v\n", err)
	} else {
		fmt.Printf("Return is %x, address is %x, and gas used is %d\n\n", ret, addr, gas)
	}
	contractHash := vmenv.intraBlockState.ResolveCodeHash(addr)
	fmt.Printf("The contract hash is %x\n", contractHash)
}
```

The test result shows the contract deployment is successful:

```
=== RUN   TestCreateTx
=== PAUSE TestCreateTx
=== CONT  TestCreateTx
balance is 10000
Return is , address is 0000000000000000000000000000000000000008, and gas used is 9994

The contract hash is c5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470
--- PASS: TestCreateTx (0.04s)
PASS
```
