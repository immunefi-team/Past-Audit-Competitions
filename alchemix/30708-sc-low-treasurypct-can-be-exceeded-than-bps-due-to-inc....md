# 30708 - \[SC - Low] treasuryPct can be exceeded than BPS due to inc...

Submitted on May 5th 2024 at 11:43:39 UTC by @OxRizwan for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30708

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Logic errors
* Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`treasuryPct` can be exceeded than `BPS` due to incorrect validation in `RevenueHandler.sol` constructor

## Vulnerability Details

`RevenueHandler.sol` has constructor which initialize the state variables like `veALCX`, `treasury` and `treasuryPct` with their values.

```solidity
    constructor(address _veALCX, address _treasury, uint256 _treasuryPct) Ownable() {
        veALCX = _veALCX;
        require(_treasury != address(0), "treasury cannot be 0x0");
        treasury = _treasury;
@>      require(treasuryPct <= BPS, "treasury pct too large");
        treasuryPct = _treasuryPct;
    }
```

The issue here is `treasuryPct` value can be exceed than `BPS` due to incorrect input validation and logic error in constructor implementation. The constructor checks the BPS is less than equal with `treasuryPct` which is a state varibale. This is not correct as the input argument i.e `_treasuryPct` should be validated with BPS max value.

This would allow the value of `treasuryPct` to be able to set to maximum value of uint256 and can be bypassed the BPS check limit. It means that `treasuryPct` can be set to 150% or 200% or any number.

It should be noted, even this issue happend in real world, this can be corrected by calling `setTreasuryPct()` which is correctly implemented.

```solidity
    function setTreasuryPct(uint256 _treasuryPct) external override onlyOwner {
        require(_treasuryPct <= BPS, "treasury pct too large");
        require(_treasuryPct != treasuryPct, "treasury pct unchanged");
        treasuryPct = _treasuryPct;
        emit TreasuryPctUpdated(_treasuryPct);
    }
```

Therefore, this issue is identified as low severity due to logic error in handling input validation in current implementatation.

## Impact Details

`treasuryPct` value can be exceeded than `BPS`. It means the `treasuryPct` can be set upto type(uint256).max value at contract construction level. This would lead to incorrect calculations in contract.

## References

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/RevenueHandler.sol#L77>

## Recommendation to fix

Consider below changes:

```diff
    constructor(address _veALCX, address _treasury, uint256 _treasuryPct) Ownable() {
        veALCX = _veALCX;
        require(_treasury != address(0), "treasury cannot be 0x0");
        treasury = _treasury;
-        require(treasuryPct <= BPS, "treasury pct too large");
+        require(_treasuryPct <= BPS, "treasury pct too large");
        treasuryPct = _treasuryPct;
    }
```

## Proof of Concept

The issue is about incorrect logic error with respect to input error handling in current implementation of constructor. This would allow to by pass the treasury percent value greater than BPS.

Please check the `Recommendation to fix` above and further description to understand the issue. This can be easily understood as its not complex issue so there is no need for coded POC.

Thanks for your understanding.
