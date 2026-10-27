# VeChain Hayabusa Upgrade | Attackathon

## Reports by Severity

<details>

<summary>Critical</summary>

* \#55632 \[BC-Critical] Delegation submitted in the same period before a validator exit will be permanently frozen

</details>

<details>

<summary>Medium</summary>

* \#55957 \[SC-Medium] `checkStake` does not check for uint64 overflow
* \#56611 \[BC-Medium] Remote P2P Crash During Sync (Thor default configuration)
* \#57055 \[BC-Medium] DOS via P2P during block header validation using bad proof

</details>

<details>

<summary>Low</summary>

* \#57136 \[BC-Low] TxPool priority cache lets base-fee swings reduce proposers tips

</details>

<details>

<summary>Insight</summary>

* \#57021 \[BC-Insight] Lack of Panic Recovery in `housekeeping` Goroutine Creates Potential for Denial of Service
* \#57412 \[SC-Insight] Gas Optimization Insight: Improve Gas Cost Efficiency by the Use of Custom Errors in `Staker.sol` Contract
* \#55711 \[SC-Insight] Redundant Gas Charge in `native_addValidation` Function Leads to Unnecessary Gas Costs
* \#56626 \[BC-Insight] Trivial renewalList Bloat Attack Exploits Unmetered Database Writes to Increase Block Processing Time, Risking BFT Disruption
* \#56629 \[BC-Insight] There is an issue in Mapping Gas Undercharge and is Enables ≥30% Extra Node Work Per Unit Gas
* \#55806 \[BC-Insight] Critical: Missing Input Validation in Governance Parameter Allows Malicious Underflow, Leading to Permanent Freeze of All DPoS Rewards
* \#56454 \[BC-Insight] Gas Undercharging threatens HAYABUSA network upgrade
* \#57468 \[BC-Insight] There is an issue about Zero VTHO Generation During Hayabusa Transition Period
* \#56362 \[BC-Insight] During `addValidation` if PoS not active, `Authority.Native(env.State()).Get()` should consume double the gas
* \#55524 \[BC-Insight] Null body transaction submission crashes RPC handler
* \#55926 \[BC-Insight] `totalSupply()` overstates circulating VTHO
* \#56256 \[BC-Insight] Redundant SLOAD for Global Endorsement Parameter
* \#56513 \[BC-Insight] During the call to `native_issuance`, there's a missing gas charge before call to `CalculateRewards`
* \#56367 \[SC-Insight] Staker Gas Optimization: Public to External Visibility
* \#56187 \[BC-Insight] Brittle, Hardcoded Gas Metering Model
* \#56403 \[BC-Insight] There is a problem in the DPoS Threshold Switch Undercounts Votes at Hayabusa Activation
* \#56045 \[BC-Insight] Block Packing Starvation via Oversized Priority Transactions
* \#57179 \[BC-Insight] During the call to `native_totalSupply`, there's missing gas charges
* \#55925 \[BC-Insight] Underpriced supply queries enable cheap CPU DoS
* \#56345 \[BC-Insight] There is an issue related to strict threshold Breaks Exact 2/3, and is Causing Finality Freeze
* \#56761 \[BC-Insight] The check for integer overflow in the function `staker.go#checkStake()` is incorrect
* \#56657 \[BC-Insight] Inactive Validator Scheduling Bypass in VeChain Thor PoS Consensus Mechanism
* \#56946 \[BC-Insight] The code comparing two big.In pointers for equality, not their numeric values

</details>

***

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#57412 \[SC-Insight] Gas Optimization Insight: Improve Gas Cost Efficiency by the Use of Custom Errors in `Staker.sol` Contract
* \#55711 \[SC-Insight] Redundant Gas Charge in `native_addValidation` Function Leads to Unnecessary Gas Costs
* \#55957 \[SC-Medium] `checkStake` does not check for uint64 overflow
* \#56367 \[SC-Insight] Staker Gas Optimization: Public to External Visibility

</details>

<details>

<summary>Blockchain/DLT</summary>

* \#57021 \[BC-Insight] Lack of Panic Recovery in `housekeeping` Goroutine Creates Potential for Denial of Service
* \#56626 \[BC-Insight] Trivial renewalList Bloat Attack Exploits Unmetered Database Writes to Increase Block Processing Time, Risking BFT Disruption
* \#56629 \[BC-Insight] There is an issue in Mapping Gas Undercharge and is Enables ≥30% Extra Node Work Per Unit Gas
* \#55806 \[BC-Insight] Critical: Missing Input Validation in Governance Parameter Allows Malicious Underflow, Leading to Permanent Freeze of All DPoS Rewards
* \#56454 \[BC-Insight] Gas Undercharging threatens HAYABUSA network upgrade
* \#57468 \[BC-Insight] There is an issue about Zero VTHO Generation During Hayabusa Transition Period
* \#56362 \[BC-Insight] During `addValidation` if PoS not active, `Authority.Native(env.State()).Get()` should consume double the gas
* \#55524 \[BC-Insight] Null body transaction submission crashes RPC handler
* \#55926 \[BC-Insight] `totalSupply()` overstates circulating VTHO
* \#56256 \[BC-Insight] Redundant SLOAD for Global Endorsement Parameter
* \#56513 \[BC-Insight] During the call to `native_issuance`, there's a missing gas charge before call to `CalculateRewards`
* \#56187 \[BC-Insight] Brittle, Hardcoded Gas Metering Model
* \#56611 \[BC-Medium] Remote P2P Crash During Sync (Thor default configuration)
* \#56403 \[BC-Insight] There is a problem in the DPoS Threshold Switch Undercounts Votes at Hayabusa Activation
* \#56045 \[BC-Insight] Block Packing Starvation via Oversized Priority Transactions
* \#57179 \[BC-Insight] During the call to `native_totalSupply`, there's missing gas charges
* \#55925 \[BC-Insight] Underpriced supply queries enable cheap CPU DoS
* \#56345 \[BC-Insight] There is an issue related to strict threshold Breaks Exact 2/3, and is Causing Finality Freeze
* \#56761 \[BC-Insight] The check for integer overflow in the function `staker.go#checkStake()` is incorrect
* \#56657 \[BC-Insight] Inactive Validator Scheduling Bypass in VeChain Thor PoS Consensus Mechanism
* \#57055 \[BC-Medium] DOS via P2P during block header validation using bad proof
* \#56946 \[BC-Insight] The code comparing two big.In pointers for equality, not their numeric values
* \#55632 \[BC-Critical] Delegation submitted in the same period before a validator exit will be permanently frozen
* \#57136 \[BC-Low] TxPool priority cache lets base-fee swings reduce proposers tips

</details>
