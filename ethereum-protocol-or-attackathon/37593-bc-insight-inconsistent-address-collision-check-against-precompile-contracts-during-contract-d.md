# #37593 \[BC-Insight] Inconsistent Address Collision Check Against Precompile Contracts During Contract Deployment

**Submitted on Dec 10th 2024 at 04:35:52 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37593
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/paradigmxyz/reth>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

Within the contract deployment in the Revm (utilized in Ethereum execution client, Reth), it has the check that the newly created contract address does not belong to the precompile contract addresses, while the Ethereum execution specs and other Ethereum execution clients do not have such check. Though it is unlikely to happen due to the hardness of hash collision, it could lead to consensus issues once it’s triggered.

## Vulnerability Details

Affected Codebase:\
<https://github.com/bluealloy/revm/tree/v50>

The function `make_create_frame()` is utilized to create a contract via contract deployment transaction, opcode Create and Create2.

<https://github.com/bluealloy/revm/blob/v50/crates/revm/src/context/evm\\_context.rs#L267>

```
pub fn make_create_frame(
        &mut self,
        spec_id: SpecId,
        inputs: &CreateInputs,
    ) -> Result<FrameOrResult, EVMError<DB::Error>> {
        let return_error = |e| {
            Ok(FrameOrResult::new_create_result(
                InterpreterResult {
                    result: e,
                    gas: Gas::new(inputs.gas_limit),
                    output: Bytes::new(),
                },
                None,
            ))
        };

        // Check depth
        if self.journaled_state.depth() > CALL_STACK_LIMIT {
            return return_error(InstructionResult::CallTooDeep);
        }

        // Prague EOF
        if spec_id.is_enabled_in(OSAKA) && inputs.init_code.starts_with(&EOF_MAGIC_BYTES) {
            return return_error(InstructionResult::CreateInitCodeStartingEF00);
        }

        // Fetch balance of caller.
        let caller_balance = self.balance(inputs.caller)?;

        // Check if caller has enough balance to send to the created contract.
        if caller_balance.data < inputs.value {
            return return_error(InstructionResult::OutOfFunds);
        }

        // Increase nonce of caller and check if it overflows
        let old_nonce;
        if let Some(nonce) = self.journaled_state.inc_nonce(inputs.caller) {
            old_nonce = nonce - 1;
        } else {
            return return_error(InstructionResult::Return);
        }

        // Create address
        let mut init_code_hash = B256::ZERO;
        let created_address = match inputs.scheme {
            CreateScheme::Create => inputs.caller.create(old_nonce),
            CreateScheme::Create2 { salt } => {
                init_code_hash = keccak256(&inputs.init_code);
                inputs.caller.create2(salt.to_be_bytes(), init_code_hash)
            }
        };

        // created address is not allowed to be a precompile.
        if self.precompiles.contains(&created_address) {
            return return_error(InstructionResult::CreateCollision);
        }

        // warm load account.
        self.load_account(created_address)?;

        // create account, transfer funds and make the journal checkpoint.
        let checkpoint = match self.journaled_state.create_account_checkpoint(
            inputs.caller,
            created_address,
            inputs.value,
            spec_id,
        ) {
            Ok(checkpoint) => checkpoint,
            Err(e) => {
                return return_error(e);
            }
        };

        let bytecode = Bytecode::new_legacy(inputs.init_code.clone());

        let contract = Contract::new(
            Bytes::new(),
            bytecode,
            Some(init_code_hash),
            created_address,
            None,
            inputs.caller,
            inputs.value,
        );

        Ok(FrameOrResult::new_create_frame(
            created_address,
            checkpoint,
            Interpreter::new(contract, inputs.gas_limit, false),
        ))
    }
```

After the address is created via create or create2, it checks if the resulting address belongs to the precompile contract addresses or not:

<https://github.com/bluealloy/revm/blob/v50/crates/revm/src/context/evm\\_context.rs#L320>

```
 // created address is not allowed to be a precompile.
        if self.precompiles.contains(&created_address) {
            return return_error(InstructionResult::CreateCollision);
        }
```

Though the precompile contract addresses are supposed to be reserved only for the precompile contract, and it indeed should have one. However, this check does not align with Ethereum execution specs and other Ethereum execution clients (We reported it to the Go Ethereum team but they refused to fix it as it aligns with the Ethereum specs and is unlikely to happen due to hash collision.)

For example,

Ethereum execution spaces

After creating the address (contract\_address) with create or create2, the function `generic_create()` is invoked without validation of the newly created address in the precompile contract addresses.

```
def generic_create(
    evm: Evm,
    endowment: U256,
    contract_address: Address,
    memory_start_position: U256,
    memory_size: U256,
    init_code_gas: Uint,
) -> None:
    """
    Core logic used by the `CREATE*` family of opcodes.
    """
    # This import causes a circular import error
    # if it's not moved inside this method
    from ...vm.interpreter import (
        MAX_CODE_SIZE,
        STACK_DEPTH_LIMIT,
        process_create_message,
    )


    call_data = memory_read_bytes(
        evm.memory, memory_start_position, memory_size
    )
    if len(call_data) > 2 * MAX_CODE_SIZE:
        raise OutOfGasError


    evm.accessed_addresses.add(contract_address)


    create_message_gas = max_message_call_gas(Uint(evm.gas_left))
    evm.gas_left -= create_message_gas
    if evm.message.is_static:
        raise WriteInStaticContext
    evm.return_data = b""


    sender_address = evm.message.current_target
    sender = get_account(evm.env.state, sender_address)


    if (
        sender.balance < endowment
        or sender.nonce == Uint(2**64 - 1)
        or evm.message.depth + Uint(1) > STACK_DEPTH_LIMIT
    ):
        evm.gas_left += create_message_gas
        push(evm.stack, U256(0))
        return


    if account_has_code_or_nonce(evm.env.state, contract_address):
        increment_nonce(evm.env.state, evm.message.current_target)
        push(evm.stack, U256(0))
        return


    increment_nonce(evm.env.state, evm.message.current_target)


    child_message = Message(
        caller=evm.message.current_target,
        target=Bytes0(),
        gas=create_message_gas,
        value=endowment,
        data=b"",
        code=call_data,
        current_target=contract_address,
        depth=evm.message.depth + Uint(1),
        code_address=None,
        should_transfer_value=True,
        is_static=False,
        accessed_addresses=evm.accessed_addresses.copy(),
        accessed_storage_keys=evm.accessed_storage_keys.copy(),
        parent_evm=evm,
    )
    child_evm = process_create_message(child_message, evm.env)


    if child_evm.error:
        incorporate_child_on_error(evm, child_evm)
        evm.return_data = child_evm.output
        push(evm.stack, U256(0))
    else:
        incorporate_child_on_success(evm, child_evm)
        evm.return_data = b""
        push(evm.stack, U256.from_be_bytes(child_evm.message.current_target))
```

Go Ethereum

There is no check to ensure the newly created `address` passed into the function is not one of the precompile contract addresses.

<https://github.com/ethereum/go-ethereum/blob/293a300d64be3d9a1c2cc92c26fcff4089deadcd/core/vm/evm.go#L418>

```
func (evm *EVM) create(caller ContractRef, codeAndHash *codeAndHash, gas uint64, value *big.Int, address common.Address, typ OpCode) ([]byte, common.Address, uint64, error) {
	// Depth check execution. Fail if we're trying to execute above the
	// limit.
	if evm.depth > int(params.CallCreateDepth) {
		return nil, common.Address{}, gas, ErrDepth
	}
	if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
		return nil, common.Address{}, gas, ErrInsufficientBalance
	}
	nonce := evm.StateDB.GetNonce(caller.Address())
	if nonce+1 < nonce {
		return nil, common.Address{}, gas, ErrNonceUintOverflow
	}
	evm.StateDB.SetNonce(caller.Address(), nonce+1)
	// We add this to the access list _before_ taking a snapshot. Even if the creation fails,
	// the access-list change should not be rolled back
	if evm.chainRules.IsBerlin {
		evm.StateDB.AddAddressToAccessList(address)
	}
	// Ensure there's no existing contract already at the designated address
	contractHash := evm.StateDB.GetKeccakCodeHash(address)
	if evm.StateDB.GetNonce(address) != 0 || (contractHash != (common.Hash{}) && contractHash != emptyKeccakCodeHash) {
		return nil, common.Address{}, 0, ErrContractAddressCollision
	}
	// Create a new account on the state
	snapshot := evm.StateDB.Snapshot()
	evm.StateDB.CreateAccount(address)
	if evm.chainRules.IsEIP158 {
		evm.StateDB.SetNonce(address, 1)
	}
	evm.Context.Transfer(evm.StateDB, caller.Address(), address, value)


	// Initialise a new contract and set the code that is to be used by the EVM.
	// The contract is a scoped environment for this execution context only.
	contract := NewContract(caller, AccountRef(address), value, gas)
	contract.SetCodeOptionalHash(&address, codeAndHash)


	if evm.Config.NoRecursion && evm.depth > 0 {
		return nil, address, gas, nil
	}


	if evm.Config.Debug {
		if evm.depth == 0 {
			evm.Config.Tracer.CaptureStart(evm, caller.Address(), address, true, codeAndHash.code, gas, value)
		} else {
			evm.Config.Tracer.CaptureEnter(typ, caller.Address(), address, codeAndHash.code, gas, value)
		}
	}


	start := time.Now()


	ret, err := evm.interpreter.Run(contract, nil, false)


	// Check whether the max code size has been exceeded, assign err if the case.
	if err == nil && evm.chainRules.IsEIP158 && len(ret) > params.MaxCodeSize {
		err = ErrMaxCodeSizeExceeded
	}


	// Reject code starting with 0xEF if EIP-3541 is enabled.
	if err == nil && len(ret) >= 1 && ret[0] == 0xEF && evm.chainRules.IsLondon {
		err = ErrInvalidCode
	}


	// if the contract creation ran successfully and no errors were returned
	// calculate the gas required to store the code. If the code could not
	// be stored due to not enough gas set an error and let it be handled
	// by the error checking condition below.
	if err == nil {
		createDataGas := uint64(len(ret)) * params.CreateDataGas
		if contract.UseGas(createDataGas) {
			evm.StateDB.SetCode(address, ret)
		} else {
			err = ErrCodeStoreOutOfGas
		}
	}


	// When an error was returned by the EVM or when setting the creation code
	// above we revert to the snapshot and consume any gas remaining. Additionally
	// when we're in homestead this also counts for code storage gas errors.
	if err != nil && (evm.chainRules.IsHomestead || err != ErrCodeStoreOutOfGas) {
		evm.StateDB.RevertToSnapshot(snapshot)
		if err != ErrExecutionReverted {
			contract.UseGas(contract.Gas)
		}
	}


	if evm.Config.Debug {
		if evm.depth == 0 {
			evm.Config.Tracer.CaptureEnd(ret, gas-contract.Gas, time.Since(start), err)
		} else {
			evm.Config.Tracer.CaptureExit(ret, gas-contract.Gas, err)
		}
	}
	return ret, address, contract.Gas, err
}
```

## Impact Details

In case that the newly created contract address belongs to the precompile contract addresses, the contract deployment fails but it succeeds in other clients. However, this is unlikely to happen due to the hardness of the hash collision.

## References

* <https://github.com/bluealloy/revm/tree/v50>
* <https://github.com/ethereum/execution-specs>
* <https://github.com/ethereum/go-ethereum/tree/v1.14.12>

## Proof of Concept

## Proof of Concept

For simplicity, we modify the code inside the `make_create_frame( )` to mimic that the newly created address is `0x08` (a precommplile contract address) and perform the testing.

```
      // Create address
       let mut init_code_hash = B256::ZERO;
       /*
       let created_address = match inputs.scheme {
           CreateScheme::Create => inputs.caller.create(old_nonce),
           CreateScheme::Create2 { salt } => {
               init_code_hash = keccak256(&inputs.init_code);
               inputs.caller.create2(salt.to_be_bytes(), init_code_hash)
           }
       };
       */
       let created_address =Address::with_last_byte(0x08);  // assume it is a precompile address
       if self.precompiles.contains(&created_address) {
           return return_error(InstructionResult::CreateCollision);
       }
```

Run the following test case to show that the `0x08` address is not allowed to be deployed with contract:

```
  fn test_make_create_frame_fails_if_precompile() {
       let env = Env::default();
       let mut cdb = CacheDB::new(EmptyDB::default());
       let bal = U256::from(3_000_000_000_u128);
       let by = Bytecode::new_raw(Bytes::from(vec![0x60, 0x00, 0x60, 0x00]));
       let contract = address!("dead10000000000000000000000000000001dead"); 
       cdb.insert_account_info(
           contract,
           crate::primitives::AccountInfo {
               nonce: 0,
               balance: bal,
               code_hash: by.clone().hash_slow(),
               code: Some(by),
           },
       );
       let mut evm_context = create_cache_db_evm_context_with_balance(Box::new(env), cdb, bal);
      
       let precompiles = ContextPrecompiles::<CacheDB<EmptyDB>>::new(PrecompileSpecId::CANCUN);
       evm_context.set_precompiles(precompiles);
       let create_inputs = test_utils::create_mock_create_inputs(contract);
       let res = evm_context.make_create_frame(SpecId::CANCUN, &create_inputs);
       eprintln!("Error occurred: {:?}", res);
   }
```

The test result shows the create collision error occurs:

```
---- context::evm_context::tests::test_make_create_frame_fails_if_precompile stdout ----
Error occurred: Ok(Result(Create(CreateOutcome { result: InterpreterResult { result: CreateCollision, output: 0x, gas: Gas { limit: 0, remaining: 0, refunded: 0 } }, address: None })))
```
