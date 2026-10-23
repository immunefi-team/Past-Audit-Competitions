# 58081 sc medium missing check in function alchemistv3 setminimumcollateralization could lead to set minimumcollateralization globalminimumcollateralization&#x20;

**Submitted on Oct 30th 2025 at 13:42:18 UTC by @sol\_4th05 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58081
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `minimumCollateralization` could be set to be `> globalMinimumCollateralization` when it should not be allowed. Therefore, all the functions using these parameters would use values that should not be allowed by the framework of the `AlchemistV3`. Because of the `onlyOwner modifier` it has just low severity.

## Vulnerability Details

In the `AlchemistV3` contract the `globalMinimumCollateralization` should always be `>= minimumCollateralization` as checked by the function `AlchemistV3::setGlobalMinimumCollateralization`.

```solidity
    function setGlobalMinimumCollateralization(uint256 value) external onlyAdmin {
@>      _checkArgument(value >= minimumCollateralization);
        globalMinimumCollateralization = value;
        emit GlobalMinimumCollateralizationUpdated(value);
    }
```

However, the `minimumCollateralization` could be set to a value `>= globalMinimumCollateralization` breaking this way the property above, through the `AlchemistV3::setMinimumCollateralization`.

```solidity
    function setMinimumCollateralization(uint256 value) external onlyAdmin {
@>      _checkArgument(value >= FIXED_POINT_SCALAR);
        minimumCollateralization = value;
        emit MinimumCollateralizationUpdated(value);
    }
```

Indeed the only check made in the `AlchemistV3::setMinimumCollateralization` regards the `value` to be `>= FIXED_POINT_SCALAR`. However, the `Admin` could call this function more than once. In this case the `value` should be checked to be always `<= globalMinimumCollateralization`.

## Impact Details

The `AlchemistV3` contract allows the parameter `minimumCollateralization` to be set by the `Admin` in a way that should not be allowed `minimumCollateralization > globalMinimumCollateralization` through the `setMinimumCollateralization` function. This has an impact on all functions using the collateralization values that do not comply with the already mentioned property. The severity could have been high in case this function could not be called only by the Admin. Therefore, considering that the function has the `onlyAdmin modifier`, its severity is low.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L292-L297>

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L300-L304>

## Mitigation

A possible mitigation action could be the one below:

```diff
function setMinimumCollateralization(uint256 value) external onlyAdmin {
        _checkArgument(value >= FIXED_POINT_SCALAR);

+       if (value > globalMinimumCollateralization)  
+       {
+       setGlobalMinimumCollateralization(value);
+       }

        minimumCollateralization = value;

        emit MinimumCollateralizationUpdated(value);
    }

    /// @inheritdoc IAlchemistV3AdminActions
-    function setGlobalMinimumCollateralization(uint256 value) external onlyAdmin {
+    function setGlobalMinimumCollateralization(uint256 value) public onlyAdmin {

        _checkArgument(value >= minimumCollateralization);
        globalMinimumCollateralization = value;
        emit GlobalMinimumCollateralizationUpdated(value);
    }
```

## Link to Proof of Concept

<https://gist.github.com/0x4th05/a3922b7a322e5656f4a8a38fe3937f01>

## Proof of Concept

## Proof of Concept

Add the following test to the test suite (`AlchemistV3.t.sol`) of the project, and run this: `forge test --mt test_setminimumCollateralizationGreaterThanGlobalMinimumCollateralization -vvvvv` and it should pass

```solidity
function test_setminimumCollateralizationGreaterThanGlobalMinimumCollateralization () external {
    vm.startPrank(alOwner);
//  using collateralization parameter = 2e10 > 1_111_111_111_111_111_111 (globalMinimumCollateralization)
    alchemist.setMinimumCollateralization(2e20);
    vm.assertGt(alchemist.minimumCollateralization(), alchemist.globalMinimumCollateralization(), "Test failed");
    console.log("minimumCollateralization:", alchemist.minimumCollateralization());
    console.log("globalMinimumCollateralization:", alchemist.globalMinimumCollateralization());
    vm.stopPrank(); 
    }
```

The result is the following:

```
  [48468] AlchemistV3Test::test_setminimumCollateralizationGreaterThanGlobalMinimumCollateralization()
    ├─ [0] VM::startPrank(0x000000000000000000000000000000000000dEaD)
    │   └─ ← [Return] 
    ├─ [14482] TransparentUpgradeableProxy::fallback(200000000000000000000 [2e20])
    │   ├─ [9270] AlchemistV3::setMinimumCollateralization(200000000000000000000 [2e20]) [delegatecall]
    │   │   ├─ emit MinimumCollateralizationUpdated(minimumCollateralization: 200000000000000000000 [2e20])
    │   │   ├─  storage changes:
    │   │   │   @ 10: 0x0000000000000000000000000000000000000000000000000f6b75ab2bc471c7 → 0x00000000000000000000000000000000000000000000000ad78ebc5ac6200000
    │   │   └─ ← [Stop] 
    │   └─ ← [Return] 
    ├─ [2637] TransparentUpgradeableProxy::fallback() [staticcall]
    │   ├─ [1925] AlchemistV3::minimumCollateralization() [delegatecall]
    │   │   └─ ← [Return] 200000000000000000000 [2e20]
    │   └─ ← [Return] 200000000000000000000 [2e20]
    ├─ [3933] TransparentUpgradeableProxy::fallback() [staticcall]
    │   ├─ [3221] AlchemistV3::globalMinimumCollateralization() [delegatecall]
    │   │   └─ ← [Return] 1111111111111111111 [1.111e18]
    │   └─ ← [Return] 1111111111111111111 [1.111e18]
    ├─ [0] VM::assertGt(200000000000000000000 [2e20], 1111111111111111111 [1.111e18], "Test failed") [staticcall]
    │   └─ ← [Return] 
    ├─ [2637] TransparentUpgradeableProxy::fallback() [staticcall]
    │   ├─ [1925] AlchemistV3::minimumCollateralization() [delegatecall]
    │   │   └─ ← [Return] 200000000000000000000 [2e20]
    │   └─ ← [Return] 200000000000000000000 [2e20]
    ├─ [0] console::log("minimumCollateralization:", 200000000000000000000 [2e20]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [1933] TransparentUpgradeableProxy::fallback() [staticcall]
    │   ├─ [1221] AlchemistV3::globalMinimumCollateralization() [delegatecall]
    │   │   └─ ← [Return] 1111111111111111111 [1.111e18]
    │   └─ ← [Return] 1111111111111111111 [1.111e18]
    ├─ [0] console::log("globalMinimumCollateralization:", 1111111111111111111 [1.111e18]) [staticcall]
    │   └─ ← [Stop] 
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return] 
    ├─  storage changes:
    │   @ 10: 0x0000000000000000000000000000000000000000000000000f6b75ab2bc471c7 → 0x00000000000000000000000000000000000000000000000ad78ebc5ac6200000
    └─ ← [Return] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 31.16ms (6.21ms CPU time)
```
