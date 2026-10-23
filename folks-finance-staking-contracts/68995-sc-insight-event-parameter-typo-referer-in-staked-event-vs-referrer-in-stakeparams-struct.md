# 68995 sc insight event parameter typo referer in staked event vs referrer in stakeparams struct

**Submitted on Mar 12th 2026 at 09:30:12 UTC by @godwinudo for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #68995
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/interfaces/IStakingV1.sol>
* **Impacts:**

## Description

### Brief/Intro

The `Staked` event in `IStakingV1.sol` uses the parameter name `referer` (single `r`), while the `StakeParams` struct uses `referrer` (double `r`). This naming inconsistency is baked into the contract's ABI and cannot be corrected post-deployment, causing permanent confusion for off-chain integrations.

### Vulnerability Details

The `IStakingV1.sol` interface defines the `StakeParams` struct with the field name `referrer`:

```solidity
struct StakeParams {
    uint64 maxStakingDurationSeconds;
    uint64 maxUnlockDurationSeconds;
    uint32 minAprBps;
    address referrer;       // <-- double 'r'
}
```

However, the `Staked` event in the same interface uses the spelling `referer`:

```solidity
event Staked(
    address indexed user,
    uint8 indexed periodIndex,
    address indexed referer,   // <-- single 'r'
    uint8 stakeIndex,
    uint256 amount
);
```

The event is emitted in `_stake()` in `Staking.sol`, passing the struct field into the event parameter:

```solidity
emit Staked(msg.sender, periodIndex, params.referrer, stakeIndex, amount);
```

The code compiles and functions correctly because event parameter names are metadata, and the EVM only cares about the types and indexing for topic hashing. The event signature hash is computed from the types (`address, uint8, address, uint8, uint256`), not the parameter names. So no on-chain behavior is affected.

However, the parameter name `referer` becomes part of the contract's ABI JSON output. Every tool that generates typed bindings from the ABI inherits this inconsistency. A subgraph developer defining their schema sees `referer` in the event ABI but `referrer` in the struct ABI and documentation.

Since the event signature is encoded in the deployed bytecode, this cannot be fixed without redeploying the contract.

### Impact Details

Off-chain systems that parse the contract's ABI subgraphs, analytics dashboards, frontend applications, and SDK wrappers will encounter different spellings for the same conceptual field depending on whether they read it from the `StakeParams` struct or the `Staked` event.

## Proof of Concept

{% stepper %}
{% step %}

### Step 1

The Staking contract is deployed. The `Staked` event is now part of the on-chain ABI with the parameter name `referer` (single `r`).
{% endstep %}

{% step %}

### Step 2

A developer reads the `StakeParams` struct in `IStakingV1.sol` and sees the field is called `referrer` (double `r`). They build their staking UI using this spelling throughout their codebase.
{% endstep %}

{% step %}

### Step 3

After a user stakes, the frontend listens for the `Staked` event to confirm the transaction and display details.

The decoded event args use the ABI parameter name `referer`, not `referrer`. The variable `referrerAddress` is `undefined`. The frontend silently fails to display or track the referral.
{% endstep %}

{% step %}

### Step 4

A subgraph developer building an indexer for the protocol defines their schema based on the struct documentation:

The actual event parameter is `event.params.referer`. The subgraph either fails to compile or silently stores null referrer addresses.
{% endstep %}

{% step %}

### Step 5

The protocol's referral reward system queries the subgraph for all stakes with a given referrer address. Because the referrer was stored as null due to the spelling mismatch, no referral rewards are attributed. Referrers who brought users to the protocol receive nothing.
{% endstep %}
{% endstepper %}
