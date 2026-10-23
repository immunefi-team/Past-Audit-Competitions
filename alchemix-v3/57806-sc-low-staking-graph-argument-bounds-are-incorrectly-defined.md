# 57806 sc low staking graph argument bounds are incorrectly defined

**Submitted on Oct 29th 2025 at 00:42:27 UTC by @a16 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57806
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/libraries/StakingGraph.sol>
* **Impacts:**

## Description

## Brief/Intro

The DELTA\_MAX, DELTA\_MIN, PRODUCT\_MAX and PRODUCT\_MIN constants are incorrectly defined.

## Vulnerability Details

The following constants are incorrectly defined:

```
int256 private constant DELTA_MAX = int256(2**DELTA_BITS - 1)-1;
int256 private constant DELTA_MIN = -int256(2**DELTA_BITS - 1);
int256 private constant PRODUCT_MAX = int256(2**PRODUCT_BITS - 1)-1;
int256 private constant PRODUCT_MIN = -int256(2**PRODUCT_BITS - 1);
```

This is likely a typing error, as there should have been parenthesis in the power argument like so:

```
int256 private constant DELTA_MAX = int256(2**(DELTA_BITS - 1))-1;
int256 private constant DELTA_MIN = -int256(2**(DELTA_BITS - 1));
int256 private constant PRODUCT_MAX = int256(2**(PRODUCT_BITS - 1))-1;
int256 private constant PRODUCT_MIN = -int256(2**(PRODUCT_BITS - 1));
```

## Impact Details

Library function accepts values that would likely cause an unexpected revert, and perhaps also data corruption in the theoretical worse case (sign bits in the internal data structure could potentially flip, causing the library to return incorrect results). There's likely no real impact on the Alchemist protocol itself, since the values required to trigger this incorrect behavior are unrealistically large.

## Proof of Concept

PoC demonstrating that values that are supposed to be accepted according to the constants defined in the library actually revert:

/// Externalized helper so Foundry can catch the internal revert on update(...) function \_addStakeExternal(int256 amount, uint256 start, uint256 duration) external { graph.addStake(amount, start, duration); }

function test\_BoundsBug\_ParamAccepted\_ButRevertsInsideUpdate() public { // Pick a value that passes the library’s CURRENT (wrong) DELTA bounds (sign bit set), // but triggers a revert during Fenwick accumulation. int256 amount = (int256(1) << 111); // Reverts for values >= 2^111 uint256 start = 1234; uint256 duration = 1;

```
console.log("start");    console.log(start);
console.log("duration"); console.log(duration);
console.log("amount");   console.logInt(amount);

vm.expectRevert();
this._addStakeExternal(amount, start, duration);
```

}
