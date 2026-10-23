# 60149 sc insight revised missing input validation in addlevels can break multiple staking tier invariant in startgatenft&#x20;

**Submitted on Nov 19th 2025 at 09:04:53 UTC by @blackgrease for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60149
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/libraries/Levels.sol>

## Summary

Affected files: `StargateNFT.sol` and `Levels.sol`

This is reported as an Insight under "Code Optimizations and Enhancements / Security Best Practices". The core issue: the `Levels::addLevel` function lacks sufficient input validation to enforce several invariants implied by the StargateNFT documentation and intended protocol behavior. As of StargateNFT v3 the `updateLevel` function is no longer available, so any incorrect configuration added via `addLevel` is effectively permanent unless the contract is upgraded.

Invariants (from documentation and intended behavior)

* Each staking tier must have a unique id.
* Each staking tier must have a unique name.
* Each staking tier has its own VET staking amount.
* Each staking tier must have its own/unique reward factor. (Docs: "Each NFT tier is associated with a reward multiplier that enhances the user's share of rewards")
* Each staking tier must have a different maturity period. (Docs: "Every NFT level has a different maturity period,")
* A maturity period cannot be 0. (Docs: "Every staking NFT is subject to a maturity period, which is determined by its node tier.")
* All staking tiers must have a cap. (Docs: "All NFT levels have a cap...")

## What is missing

`Levels::addLevel` only validates:

* level.name is not empty
* vetAmountRequiredToStake is not 0 (for non-zero level)
* cap vs circulatingSupply consistency (circulatingSupply must not exceed cap)

It does NOT validate:

* vetAmountRequiredToStake uniqueness across levels
* scaledRewardFactor (non-zero, uniqueness, or reasonable bounds)
* maturityBlocks (non-zero, uniqueness across levels, or reasonable upper bound)
* optionally: uniqueness of level names (affects UI/UX)

Because `updateLevel` is removed in v3, these misconfigurations cannot be corrected through on-chain functions after submission of a bad level.

## Breaking the invariants — Examples & Effects

Below are concrete cases that illustrate how lack of validation can break protocol expectations:

Case #1: Staking tiers with same VET staking amount

* addLevel does not check if the new level's VET stake amount matches an existing level.
* If a new level requires the same VET amount but has a higher reward factor, stakers can earn greater rewards for the same stake — undermining tier semantics and fairness.

Case #2: No validation for level.scaledRewardFactor

* Duplicate reward factor across different stake amounts: cheaper tiers could offer same rewards as more expensive tiers.
* Zero reward factor: a tier could be created that grants zero rewards.

Case #3: No validation for level.maturityBlocks

* Zero maturity: maturityBlocks could be 0, allowing immediate delegation/minting contrary to docs.
* Excessively large maturity: no upper bound, allowing unintentionally huge maturity periods (e.g., an extra zero turns 60 days into \~600 days).
* Duplicate maturity periods: multiple tiers could share the same maturity period, contradicting "Every NFT level has a different maturity period".

## Relevant code excerpts

StargateNFT::addLevel (simplified):

```solidity
function addLevel(
    DataTypes.StargateNFTStorage storage $,
    DataTypes.LevelAndSupply memory _levelAndSupply
) external { 
    // Increment MAX_LEVEL_ID
    $.MAX_LEVEL_ID++;

    // Override level ID to be the new MAX_LEVEL_ID (We do not care about the level id in the input)
    _levelAndSupply.level.id = $.MAX_LEVEL_ID;

    // Validate level fields
    _validateLevel(_levelAndSupply.level); //@audit: only checks name and amount to stake is not the same. Could also have a duplicate name check but optional

    // Validate supply
    if (_levelAndSupply.circulatingSupply > _levelAndSupply.cap) { //@audit-info: checked in `MintingLogic`
        revert Errors.CirculatingSupplyGreaterThanCap();
    } // THIS IS A NEW LEVEL THEREFORE CAP SHOULD BE 0

    // Add new level to storage
    $.levels[_levelAndSupply.level.id] = _levelAndSupply.level;
    _checkpointLevelCirculatingSupply(
        $,
        _levelAndSupply.level.id,
        _levelAndSupply.circulatingSupply
    );
    $.cap[_levelAndSupply.level.id] = _levelAndSupply.cap;

    //@audit-issue: insufficient validation. Does not check if other levels have the same VET stake amount (duplicate levels)
    //@audit-issue: insufficient validation. No checks on maturityBlocks
    //@audit-issue: insufficient validation. No checks on the scaledRewardFactor

    emit LevelUpdated(
        _levelAndSupply.level.id,
        _levelAndSupply.level.name,
        _levelAndSupply.level.isX,
        _levelAndSupply.level.maturityBlocks,
        _levelAndSupply.level.scaledRewardFactor, 
        _levelAndSupply.level.vetAmountRequiredToStake
    ); 
    emit LevelCirculatingSupplyUpdated($.MAX_LEVEL_ID, 0, _levelAndSupply.circulatingSupply);
    emit LevelCapUpdated($.MAX_LEVEL_ID, 0, _levelAndSupply.cap);
}
```

\_validateLevel:

```solidity
function _validateLevel(DataTypes.Level memory _level) internal pure {
    // Name cannot be empty
    if (bytes(_level.name).length == 0) {
        revert Errors.StringCannotBeEmpty();
    }

    // VET amount required to stake must be greater than 0 for all levels except level 0
    if (_level.vetAmountRequiredToStake == 0) {
        revert Errors.ValueCannotBeZero();
    }
}
```

## Impact

This is reported as an Insight (not a formal vulnerability) because the consequence is a misconfiguration that can damage staking semantics, UX, and protocol intent. Missing validation is a security best practice issue: it’s advisable to validate inputs that make permanent changes to contract state, especially when on-chain updates are irreversible in the deployed implementation.

Potential impacts:

* Economic unfairness (lower-tier users earning rewards meant for higher tiers).
* Broken UX due to duplicate names or tiers with unexpected properties.
* Permanently incorrect tier configuration requiring an upgrade to fix.

## Mitigation / Recommendations

* Add additional validation in `addLevel` to enforce the invariants before persisting a new level:
  * Ensure vetAmountRequiredToStake is unique across existing levels (or otherwise ensure tiers are distinctly ordered).
  * Ensure scaledRewardFactor is non-zero and within reasonable bounds; optionally ensure uniqueness.
  * Ensure maturityBlocks is non-zero, within a reasonable upper bound, and unique across levels.
  * Optionally ensure name uniqueness to avoid UI/UX confusion.
* Re-introduce an `updateLevel` function (with appropriate admin controls and constraints) so that legitimate mistakes can be rectified without a full contract upgrade.
* Consider adding events and off-chain checks/tests in deployment tooling to validate tier sets before calling `addLevel`.

## Proof of Concept

A Foundry-compatible PoC is provided in a gist that demonstrates creating levels that break these invariants:

PoC Gist: <https://gist.github.com/blackgrease/87a7070799eaea0aecabf892a4e0d9f0>

Run with:

```
forge test --mt testPotentialForGriefingViaInsufficientLevelInputValidation -C ./packages/contracts
```

PoC description:

* Case\_#1: Demonstrates duplicate stake amounts, duplicate maturity periods and a reward factor of 0 in one level.
* Case\_#2: Demonstrates duplicate stake amounts with different reward factors.
* Case\_#3: Demonstrates lack of maximum maturityBlocks (e.g., an unintended extra zero leads to \~600 days) and duplicate reward factors.

Note: the test suite was originally Hardhat-based; conversion instructions are below.

## Hardhat -> Foundry conversion (how to run PoC)

{% stepper %}
{% step %}

### Step 1

Clone the repository:

```bash
git clone https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa.git
cd audit-comp-vechain-vechain-stargate-hayabusa
```

{% endstep %}

{% step %}

### Step 2

Install forge test dependencies:

```bash
forge install foundry-rs/forge-std
forge install OpenZeppelin/openzeppelin-contracts-upgradeable@v5.0.2
forge install OpenZeppelin/openzeppelin-contracts@v5.0.2
```

(These installs are required for the tests; OZ contracts may be used in tests.)
{% endstep %}

{% step %}

### Step 3

Compile the contracts under the packages folder:

```bash
forge compile ./packages/contracts/
```

(Compilation warnings can be ignored for the PoC run.)
{% endstep %}

{% step %}

### Step 4

Run the specific test:

```bash
forge test --mt testPotentialForGriefingViaInsufficientLevelInputValidation -C ./packages/contracts
```

This will run the test(s) that emit events and demonstrate the missing validations.
{% endstep %}
{% endstepper %}

## Additional notes

* The report treats this as an Insight: the root cause is insufficient input validation on a function that performs a permanent state change (and the removal of an `updateLevel` function exacerbates the risk).
* The suggested mitigations are conservative: enforce invariants in `addLevel` and consider reintroducing controlled update functionality to allow safe correction of accidental misconfigurations.

\-- End of report.
