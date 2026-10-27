# 51684 sc medium unbounded gas consumption in removestakerfromallvalidators leads to denial of service preventing users with large validator counts from removing associations and potentially lock&#x20;

**Submitted on Aug 4th 2025 at 21:28:16 UTC by @drdee for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51684
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeValidatorLogic.sol>
* **Impacts:** Unbounded gas consumption

## Description

### Title

Unbounded Gas Consumption in `removeStakerFromAllValidators` Leads to Denial-of-Service, Preventing Users with Large Validator Counts from Removing Associations and Potentially Locking State.

### Brief/Intro

The `removeStakerFromAllValidators` function in the `PlumeValidatorLogicWrapper` contract suffers from unbounded gas consumption, failing to remove validator associations for users with large counts (e.g., 2000+) due to exceeding gas limits. If exploited on mainnet, this could lead to a denial-of-service (DoS) scenario, locking user state and potentially staked funds, as users cannot clean up associations, rendering the contract unusable for high-stake participants.

### Vulnerability Details

The vulnerability arises in the `removeStakerFromAllValidators` function within `PlumeValidatorLogic.sol`, which iterates over all validators associated with a staker via the `userValidators` array and calls `removeStakerFromValidator` for each with zero stake. The loop’s gas cost scales linearly (or worse) with the number of validators, as each iteration involves storage reads, writes, and array manipulations. Testing on the Plume Testnet with 199 validators used 10.47M gas (transaction `0x76831debabc0b1f12f1b39d5847552d1c8126b8bb904ad944e160a12dd6fff28`), while an attempt with 2000 validators failed with `gas required exceeds allowance (50000000)`.

Vulnerable code snippet:

```solidity
function removeStakerFromAllValidators(PlumeStakingStorage.Layout storage $, address staker) internal {
    uint16[] memory userAssociatedValidators = $.userValidators[staker];
    for (uint256 i = 0; i < userAssociatedValidators.length; i++) {
        uint16 validatorId = userAssociatedValidators[i];
        if ($.userValidatorStakes[staker][validatorId].staked == 0) {
            removeStakerFromValidator($, staker, validatorId);
        }
    }
}
```

No batching or gas-awareness means this can exceed network gas limits (10M–30M on many chains). Extrapolating from 10.47M gas for 199 validators, 2000 validators could require \~105M gas (10.47M × 2000/199), confirming the issue. The `removeStakerFromValidator` function adds array swap/pop costs, increasing total gas. Reproduction done via `forge script` for 2000 validators failed due to gas limits.

### Impact Details

This DoS vulnerability prevents users with very large validator lists (e.g., 2000+) from removing associations, potentially locking state or funds if cleanup is required for withdrawals. On mainnet with a \~30M gas limit, calls requiring \~105M gas will revert. Impact scales with validator count and affects high-value stakers and protocol usability.

### References

* Deployed Contract Address: `0xAcec786f316Fa9B41C241E53310f14EFC072416C`
* Transaction Hashes:
  * `0xe8e2518de183670dac61ce397a32855bf2aebae5b83210e27c137a609eb823aa` (199 validators, 19M gas)
  * `0x76831debabc0b1f12f1b39d5847552d1c8126b8bb904ad944e160a12dd6fff28` (remove 199 validators, 10.47M gas)
  * Latest failure (>50M gas for 2000 validators attempt)
* Code: `PlumeValidatorLogic.sol` (provided in prior context)
* Plume Testnet Explorer: `https://testnet-explorer.plume.org/tx/<TRANSACTION_HASH>`
* RPC Endpoint: `https://testnet-rpc.plume.org`
* Immunefi Plume Network Attackathon: `https://immunefi.com/bug-bounty/plume-network/`

## Mitigation Steps

Suggested modification to add batching and a gas check:

```solidity
function removeStakerFromAllValidators(PlumeStakingStorage.Layout storage $, address staker, uint256 maxIterations) internal {
    uint16[] memory userAssociatedValidators = $.userValidators[staker];
    uint256 limit = maxIterations < userAssociatedValidators.length ? maxIterations : userAssociatedValidators.length;
    require(gasleft() > 100_000, "Insufficient gas for operation");
    for (uint256 i = 0; i < limit; i++) {
        uint16 validatorId = userAssociatedValidators[i];
        if ($.userValidatorStakes[staker][validatorId].staked == 0) {
            removeStakerFromValidator($, staker, validatorId);
        }
    }
}
```

Recommendations:

* Batching: Add `maxIterations` (e.g., 100) to limit processed validators per call and allow incremental cleanup.
* Gas Check: Include `require(gasleft() > 100_000)` to avoid out-of-gas mid-execution.
* External Exposure: Expose the batched function publicly in the wrapper (e.g., `removeStakerFromAllValidators(address, uint256)`), enabling users to manage large validator sets incrementally.
* Verification: Test with 2000 validators in batches (e.g., 100), ensuring each call stays below acceptable gas limits.

## Link to Proof of Concept

<https://gist.github.com/DeepakDubeyCS/8bcdbb65aae3708f824f3945127583d6>

## Proof of Concept

### Executive Summary

The Plume Validator Logic protocol on Plume Testnet manages validator associations via the `PlumeValidatorLogicWrapper` contract. Manual testing from July 29, 2025 to August 5, 2025 identifies an unbounded gas consumption issue in `removeStakerFromAllValidators`. This poses a denial-of-service (DoS) risk for users with very large validator lists, demonstrated using the helper contract in the linked gist.

### Audit Scope and Objectives

* Reviewed Contracts: `PlumeValidatorLogic.sol`, `PlumeValidatorLogicWrapper.sol` (wrapper deployed at `0xAcec786f316Fa9B41C241E53310f14EFC072416C`)
* In-Scope Functions: `setupStaker`, `removeStakerFromAllValidators`, `getUserValidators`
* Objective: Demonstrate unbounded gas consumption for large validator counts (2000 validators)

### Methodology

* Manual code review of relevant logic
* Transaction testing with `cast send` and Forge scripts on Plume Testnet
* Threat modeling focusing on gas exhaustion and DoS

### Findings Summary Table

* ID & Title: VULN-001: Unbounded Gas Consumption in `removeStakerFromAllValidators`
* Severity Level: High
* Impact: Prevents removal of associations for very large validator counts, causing DoS/locked state
* Reproduction: Setup large validator list and call `removeStakerFromAllValidators` — call reverts when required gas exceeds allowance
* Affected Components: `PlumeValidatorLogic.removeStakerFromAllValidators`, `removeStakerFromValidator`
* Mitigation: Add batching and gas check (see mitigation code above)
* Status: Unresolved

### Severity Classification & Impact Assessment

* Severity: High (Immunefi scale)
* Rationale: Users with 2000+ validators may be unable to perform cleanup due to network gas caps, potentially locking funds or state.

### Remediation & Verification

* Project Team Response: Not received as of report date.
* Recommended post-remediation tests: run 2000-validator cleanup in batches (e.g., 100 per tx) and verify each tx stays below target gas limits.

## Appendix — Reproduction & Setup

{% stepper %}
{% step %}

### Environment: Install Foundry

Run:

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

{% endstep %}

{% step %}

### Project Setup

Create project and configure:

```bash
mkdir plume-test && cd plume-test
forge init
```

Update `foundry.toml` with:

```toml
[profile.default]
src = "src"
test = "test"
out = "out"
libs = ["lib"]
cache_path = "cache"
gas_estimate_multiplier = 200

[rpc_endpoints]
plume_testnet = "${PLUME_RPC_URL}"

[etherscan]
plume_testnet = { key = "", url = "${VERIFIER_URL}" }
```

Install Plume contracts (use official repo or Gist versions as needed):

```bash
forge install https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeStakingStorage.sol
forge install https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeValidatorLogic.sol
```

{% endstep %}

{% step %}

### Configure Environment Variables

Create `.env`:

```bash
PLUME_RPC_URL=https://testnet-rpc.plume.org
PRIVATE_KEY="0xYour_Private_key"
```

Then source it:

```bash
source .env
```

{% endstep %}

{% step %}

### Deploy Helper Contract

* Save `PlumeValidatorLogicWrapper.sol` from: <https://gist.github.com/DeepakDubeyCS/8bcdbb65aae3708f824f3945127583d6> into: `src/PlumeValidatorLogicWrapper.sol`
* Ensure `src/` contains:
  * `PlumeValidatorLogic.sol`
  * `PlumeStakingStorage.sol`

Deploy:

```bash
forge create src/PlumeValidatorLogicWrapper.sol:PlumeValidatorLogicWrapper \
  --rpc-url $PLUME_RPC_URL \
  --private-key $PRIVATE_KEY \
  --legacy
```

{% endstep %}

{% step %}

### Setup 2000 Validators (example script)

This example splits 2000 validators into 8 batches of 250:

```bash
for i in {0..7}; do
  start=$((i * 250 + 1))
  end=$((start + 249))
  ids=$(seq -s ',' $start $end)

  echo "🔄 Sending validators $start to $end..."

  cast send 0xAcec786f316Fa9B41C241E53310f14EFC072416C \
    "setupStaker(address,uint16[])" \
    0x0000000000000000000000000000000000001239 \
    "[$ids]" \
    --rpc-url $PLUME_RPC_URL \
    --private-key $PRIVATE_KEY \
    --legacy \
    --gas-price 10000000
done
```

This batches setup to stay within gas limits.
{% endstep %}

{% step %}

### Trigger Vulnerability

Run:

```bash
cast send 0xAcec786f316Fa9B41C241E53310f14EFC072416C \
  "removeStakerFromAllValidators(address)" \
  0x0000000000000000000000000000000000001239 \
  --rpc-url $PLUME_RPC_URL \
  --private-key $PRIVATE_KEY \
  --legacy \
  --gas-price 0.3gwei
```

Expect a revert with `gas required exceeds allowance (50000000)` when validator count large enough.
{% endstep %}

{% step %}

### Verify State

Check validators for the staker:

```bash
cast call 0xAcec786f316Fa9B41C241E53310f14EFC072416C \
  "getUserValidators(address)(uint16[])" \
  0x0000000000000000000000000000000000001239 \
  --rpc-url $PLUME_RPC_URL
```

{% endstep %}
{% endstepper %}

## Deploy Helper & Reproduction References

* Plume Testnet Explorer address: <https://testnet-explorer.plume.org/address/0xacec786f316fa9b41c241e53310f14efc072416c?tab=txs>
* Immunefi Attackathon: <https://immunefi.com/bug-bounty/plume-network/>
* Gist with helper: <https://gist.github.com/DeepakDubeyCS/8bcdbb65aae3708f824f3945127583d6>

***

If you want, I can:

* Produce a patch-style diff with the batched function and a new external wrapper function signature for easy integration; or
* Suggest tests (Forge scripts) that call the batched function repeatedly until cleanup completes, measuring gas per batch.
