# 29225 - \[SC - Insight] EarlyZEROVesting is having a rounding issue and...

Submitted on Mar 11th 2024 at 06:28:52 UTC by @dontonka for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #29225

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/zerolend/governance>

Impacts:

* EarlyZEROVesting contract improvement: rounding issue and missing approval

## Description

## Brief/Intro

`EarlyZEROVesting` is having at least two minor issues which would need to be addressed. There is one rounding issue and one missing approval in order to transfer the vested NFT to the staking bonus contract.

## Vulnerability Details

**Rounding issue**

If we take the following example calling `startVesting` with an amount of `99`.

pending = `74.25` = (amount \* 75) / 100 --> `74`

upfront = `24.75` = (amount \* 25) / 100 --> `24`

spent = 99

As you can see, the NFT will represent a value of `98` (74 + 24) instead of 99, which is a loss of 1 wei.

**Missing approval**

When startVesting with staking operation, the current implementation will not work as it is `missing an approval` operation, so currently the `vesting.safeTransferFrom` will always revert.

## Impact Details

* 1 Wei precision lost per startVesting call.
* Missing approval when staking, which will make this always reverts.

## Recommendation

Apply the following changes in order to fix the two issues reported in this report.

```diff
    function startVesting(uint256 amount, bool stake) external {
        require(enableVesting || stake, "vesting not enabled; staking only");
        earlyZERO.burnFrom(msg.sender, amount);

        // vesting for earlyZERO is 25% upfront, 1 month cliff and
        // the rest (75%) across 3 months

+        //@audit: this will fix the 1 wei loss in precicion rounding issue.
+        uint256 upfront = (amount * 25) / 100;
+        uint256 pending = amount - upfront; 

        uint256 id = vesting.mint(
            stake ? address(this) : msg.sender, // address _who,
-            (amount * 75) / 100, // uint256 _pending,
-            (amount * 25) / 100, // uint256 _upfront,
+            pending, // uint256 _pending,
+            upfront, // uint256 _upfront,
            86400 * 30 * 3, // uint256 _linearDuration,
            86400 * 30, // uint256 _cliffDuration,
            block.timestamp, // uint256 _unlockDate,
            false, // bool _hasPenalty
            IVestedZeroNFT.VestCategory.EARLY_ZERO
        );

        // if the user chooses to stake then make sure to give the staking bonus
        if (stake) {
+           //@audit: this will fix the missing approval
+           vesting.approve(address(vesting), id);
            vesting.safeTransferFrom(
                address(this),
                stakingBonus,
                id,
                abi.encode(true, msg.sender)
            );
        }

        spent += amount;
    }
```

## Proof of Concept

Not applicable really, simple by inspecting the code.
