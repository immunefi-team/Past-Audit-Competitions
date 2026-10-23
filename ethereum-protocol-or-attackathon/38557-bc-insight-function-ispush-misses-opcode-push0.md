# #38557 \[BC-Insight] Function \`IsPush()\` Misses Opcode PUSH0

**Submitted on Jan 6th 2025 at 19:51:52 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38557
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

The opcode PUSH0 is introduced in EIP-3855 (<https://eips.ethereum.org/EIPS/eip-3855> ) that pushes the constant value 0 onto the stack.

In Erigon (<https://github.com/erigontech/erigon> ), the function `IsPush()` is used to check if an opcode is a push opcode, which does not include the `PUSH0` opcode. In the current implementation, this function `IsPush()` is mostly utilized in the execution tracer and external tools, such asm and goja. Missing the opcode `PUSH0` would mess up the trace generated in the upstream applications.

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/tree/v2.61.0>

The function `IsPush()` is intended to check if an opcode is a push opcode, that is, the opcodes from `PUSH0` to `PUSH32`:

<https://github.com/erigontech/erigon/blob/v2.61.0/core/vm/opcodes.go#L27>

```
// IsPush specifies if an opcode is a PUSH opcode.
func (op OpCode) IsPush() bool {
	switch op {
	case PUSH1, PUSH2, PUSH3, PUSH4, PUSH5, PUSH6, PUSH7, PUSH8, PUSH9, PUSH10, PUSH11, PUSH12, PUSH13, PUSH14, PUSH15, PUSH16, PUSH17, PUSH18, PUSH19, PUSH20, PUSH21, PUSH22, PUSH23, PUSH24, PUSH25, PUSH26, PUSH27, PUSH28, PUSH29, PUSH30, PUSH31, PUSH32:
		return true
	}
	return false
}
```

However, it misses the `PUSH0` opcode introduced in the EIP-3855 (<https://eips.ethereum.org/EIPS/eip-3855> ). This function `IsPush()` is currently utilized in the upstream execution tracer and external application. For example, the `CaptureState()` in

<https://github.com/erigontech/erigon/blob/v2.61.0/cmd/state/commands/opcode\\_tracer.go#L355>

```
func (ot *opcodeTracer) CaptureState(pc uint64, op vm.OpCode, gas, cost uint64, scope *vm.ScopeContext, rData []byte, opDepth int, err error) {
	//CaptureState sees the system as it is before the opcode is run. It seems to never get an error.
	contract := scope.Contract

	//sanity check
	if pc > uint64(MaxUint16) {
		panic(fmt.Sprintf("PC is bigger than uint16! pc=%d=0x%x", pc, pc))
	}

	pc16 := uint16(pc)
	currentTxHash := ot.env.TxHash
	currentTxDepth := opDepth - 1

	ls := len(ot.stack)
	currentEntry := ot.stack[ls-1]

	//sanity check
	if currentEntry.Depth != currentTxDepth {
		panic(fmt.Sprintf("Depth should be the same but isn't: current tx's %d, current entry's %d", currentTxDepth, currentEntry.Depth))
	}

	// is the Tx entry still not fully initialized?
	if currentEntry.TxHash == nil {
		// CaptureStart creates the entry for a new Tx, but doesn't have access to EVM data, like the Tx Hash
		// here we ASSUME that the tx entry was recently created by CaptureStart
		// AND that this is the first CaptureState that has happened since then
		// AND that both Captures are for the same transaction
		// AND that we can't go into another depth without executing at least 1 opcode
		// Note that the only connection between CaptureStart and CaptureState that we can notice is that the current op's depth should be lastTxEntry.Depth+1

		// fill in the missing data in the entry
		currentEntry.TxHash = new(libcommon.Hash)
		currentEntry.TxHash.SetBytes(currentTxHash.Bytes())
		currentEntry.CodeHash = new(libcommon.Hash)
		currentEntry.CodeHash.SetBytes(contract.CodeHash.Bytes())
		currentEntry.CodeSize = len(contract.Code)
		if ot.saveOpcodes {
			currentEntry.Opcodes = make([]opcode, 0, 200)
		}
		//fmt.Fprintf(ot.w, "%sFilled in TxHash\n", strings.Repeat("\t",depth))

		if ot.saveBblocks {
			currentEntry.Bblocks = make(sliceBblocks, 0, 10)
		}
	}

	// prepare the opcode's stack for saving
	//stackTop := &stack.Stack{Data: make([]uint256.Int, 0, 7)}//stack.New()
	// the most stack positions consumed by any opcode is 7
	//for i:= min(7, st.Len()-1); i>=0; i-- {
	//	stackTop.Push(st.Back(i))
	//}
	//THIS VERSION SHOULD BE FASTER BUT IS UNTESTED
	//stackTop := make([]uint256.Int, 7, 7)
	//sl := st.Len()
	//minl := min(7, sl)
	//startcopy := sl-minl
	//stackTop := &stack.Stack{Data: make([]uint256.Int, minl, minl)}//stack.New()
	//copy(stackTop.Data, st.Data[startcopy:sl])

	//sanity check
	if currentEntry.OpcodeFault != "" {
		panic(fmt.Sprintf("Running opcodes but fault is already set. txFault=%s, opFault=%v, op=%s",
			currentEntry.OpcodeFault, err, op.String()))
	}

	// if it is a Fault, check whether we already have a record of the opcode. If so, just add the flag to it
	errstr := ""
	if err != nil {
		errstr = err.Error()
		currentEntry.OpcodeFault = errstr
	}

	faultAndRepeated := false

	if pc16 == currentEntry.lastPc16 && op == currentEntry.lastOp {
		//it's a repeated opcode. We assume this only happens when it's a Fault.
		if err == nil {
			panic(fmt.Sprintf("Duplicate opcode with no fault. bn=%d txaddr=%s pc=%x op=%s",
				ot.blockNumber, currentEntry.TxAddr, pc, op.String()))
		}
		faultAndRepeated = true
		//ot.fsumWriter.WriteString("Fault for EXISTING opcode\n")
		//ot.fsumWriter.Flush()
		if ot.saveOpcodes {
			lo := len(currentEntry.Opcodes)
			currentEntry.Opcodes[lo-1].Fault = errstr
		}
	} else {
		// it's a new opcode
		if ot.saveOpcodes {
			newOpcode := opcode{pc16, op, errstr}
			currentEntry.Opcodes = append(currentEntry.Opcodes, newOpcode)
		}
	}

	// detect and store bblocks
	if ot.saveBblocks {
		// PC discontinuities can only happen because of a PUSH (which is followed by the data to be pushed) or a JUMP (which lands into a JUMPDEST)
		// Therefore, after a PC discontinuity we either have op==JUMPDEST or lastOp==PUSH
		// Only the JUMPDEST case is a real control flow discontinuity and therefore starts a new bblock

		lseg := len(currentEntry.Bblocks)
		isFirstBblock := lseg == 0
		isContinuous := pc16 == currentEntry.lastPc16+1 || currentEntry.lastOp.IsPush()

...
```

Though it does not affect the state transition of the Ethereum, the execution trace and other external applications dependent on it would be incorrect.

## Impact Details

The function `IsPush()` is only utilized in transaction execution trace and other external applications and does not impact the state transition of Ethereum at this moment, it would mess up the trace and produce incorrect execution trace for the upstream application.

## References

* <https://github.com/erigontech/erigon/tree/v2.61.0>
* <https://eips.ethereum.org/EIPS/eip-3855>

## Proof of Concept

## Proof of Concept

For simplicity, we create the following simple test cases:

1. Check if PUSH0, PUSH1 and PUSH32 are push opcodes:

```
package vm

import (
	"fmt"
	"testing"
)

func TestIsPush(t *testing.T) {
	fmt.Printf("Is PUSH0 a push opcode: %t\n", PUSH0.IsPush())
	fmt.Printf("Is PUSH1 a push opcode: %t\n", PUSH1.IsPush())
	fmt.Printf("Is PUSH32 a push opcode: %t\n", PUSH32.IsPush())
}
```

2. The test result shows that the `PUSH0` is not push opcode:

```
=== RUN   TestIsPush
Is PUSH0 a push opcode: false
Is PUSH1 a push opcode: true
Is PUSH32 a push opcode: true
--- PASS: TestIsPush (0.00s)
PASS


Process finished with the exit code 0
```
