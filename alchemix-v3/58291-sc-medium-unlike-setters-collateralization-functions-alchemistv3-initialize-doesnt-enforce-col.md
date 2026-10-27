# 58291 sc medium unlike setters collateralization functions alchemistv3 initialize doesnt enforce collateralization invariants allowing to break them&#x20;

**Submitted on Nov 1st 2025 at 01:26:10 UTC by @hunter0xweb3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58291
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

AlchemistV3 defines multiple collateralization requirements: MinimumCollateralization, GlobalMinimumCollateralization, CollateralizationLowerBound and a relation between them:

```
globalMinimumCollateralization >= minimumCollateralization >= collateralizationLowerBound >= FIXED_POINT_SCALAR
```

This relationship is enforced in the setter functions for this variables, however AlchemistV3::initialize allows to break the invariant and start AlchemistV3 in an invariant breakage state

## Vulnerability Details

AlchemistV3 defines variables for multiple collateralization requirements:

* MinimumCollateralization,
* GlobalMinimumCollateralization
* CollateralizationLowerBound And also defines its setter functions

```
    function setMinimumCollateralization(uint256 value) external onlyAdmin {
@>      _checkArgument(value >= FIXED_POINT_SCALAR);
        minimumCollateralization = value;
        emit MinimumCollateralizationUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setGlobalMinimumCollateralization(uint256 value) external onlyAdmin {
@>      _checkArgument(value >= minimumCollateralization);
        globalMinimumCollateralization = value;
        emit GlobalMinimumCollateralizationUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
    function setCollateralizationLowerBound(uint256 value) external onlyAdmin {
@>      _checkArgument(value <= minimumCollateralization);
@>      _checkArgument(value >= FIXED_POINT_SCALAR);
        collateralizationLowerBound = value;
        emit CollateralizationLowerBoundUpdated(value);
    }
```

From the setter function it can be seen that the invariant enforced by AlchemistV3 is the following

```
globalMinimumCollateralization >= minimumCollateralization >= collateralizationLowerBound >= FIXED_POINT_SCALAR
```

However this invariant is not enforced in AlchemistV3::initialize allowing to use it to break and start contract in an non compliant state:

```solidity
    function initialize(AlchemistInitializationParams memory params) external initializer {
        //...snippet...
@>      minimumCollateralization = params.minimumCollateralization;
@>      globalMinimumCollateralization = params.globalMinimumCollateralization;
@>      collateralizationLowerBound = params.collateralizationLowerBound;
        //...snippet... 
```

## Impact Details

* Collateralization requirements invariant breakage
* AlchemistV3 is unable to perform liquidations
* Contract failed to fullfil premises

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L174-L176\\>
<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L292-L312>

## Proof of Concept

## Proof of Concept

The following PoC shows AlchemistV3::initialize() can be used to break the AlchemistV3 collateralization's invariant:

```
globalMinimumCollateralization >= minimumCollateralization >= collateralizationLowerBound >= FIXED_POINT_SCALAR
```

enforced by setMinimumCollateralization, setGlobalMinimumCollateralization, and setCollateralizationLowerBound AlchemistV3's functions

To do so, perform the following:

In deployCoreContracts function in src/test/AlchemistV3.t.sol modify AlchemistInitializationParams struct, only the minimumCollateralization, collateralizationLowerBound and globalMinimumCollateralization fields:

```solidity
    function deployCoreContracts(uint256 alchemistUnderlyingTokenDecimals) public {
        // ...
        AlchemistInitializationParams memory params = AlchemistInitializationParams({
            admin: alOwner,
            debtToken: address(alToken),
            underlyingToken: address(vault.asset()),
            depositCap: type(uint256).max,
            minimumCollateralization: 2_111_111_111_111_111_111, //2.1
            collateralizationLowerBound: 3_111_111_111_111_111_111, // 3.1 collateralization
            globalMinimumCollateralization: 1_111_111_111_111_111_111, // 1.1
```

Leave the rest of struct with the original values

Create in the same file (src/test/AlchemistV3.t.sol) the following test

```solidity
    function testInitializeInvariantBreakage() public {
        // Check invariant was break using initialize
        assert(
            alchemist.collateralizationLowerBound() > alchemist.minimumCollateralization() 
        );
        assert(
            alchemist.minimumCollateralization() > alchemist.globalMinimumCollateralization() 
        );
    }
```

Exec test

```bash
forge test --mt testInitializeInvariantBreakage
```

Observe the AlchemistV3 invariant is bypassed
