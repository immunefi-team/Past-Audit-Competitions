# 58185 sc medium incorrect survivalaccumulator accounting logic after earmarkweight reaches 128 breaks core system invariants and can lead to protocol insolvency

**Submitted on Oct 31st 2025 at 08:44:55 UTC by @luc1jan for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58185
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

After state variable `_earmarkWeight` that represents index of amounts earmarked globally reaches `128` in decimal or `LOG2NEGFRAC_1`, `_survivalAccumulator` starts to accrue inflated amounts that leads to accounts debt being inflated and severely undercollateralized.

## Vulnerability Details

As users create and claim redemptions in `Transmuter`, `_survivalAccumulator` changes on each earmark and redemption. At some earmark event, delta `_survivalAccumulator` is product of `previousSurvival` and `earmarkedFraction`:

```js
_survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
```

Where: `previousSurvival` is product of all "survival" fractions of past earmark events. `_earmarkWeight` is just logarithmically transformed version of this value (e.g. 50% gets earmarked + another 50%; `previousSurvival` is 1/2 \* 1/2 = 1/4) and `earmarkFraction` is portion of globally unearmarked debt that was earmarked in current earmark event. (e.g. 20 out of 100 gets earmarked; `earmarkFraction` is 0.2)

With this in mind general `_survivalAccumulator` equation is: `ps1 * ef1 + ps2 * ef2 + ...`

Note that `ps_n` is product of all previous survival fractions and first survival is 1 so we can write above as: `1 * ef1 + s1 * ef2 + s1*s2 * ef3 + s1*s2*s3 * ef4 + ...`

Users on each sync update their earmarked debt based on change between their last stored `lastSurvivalAccumulator` and the global one. This is very convenient mathematically because that delta `_survivalAccumulator` on user sync will look something like this (e.g. 2 earmark events happened since last user sync, and there were 4 in total): `s1*s2 * ef3 + s1*s2*s3 * ef4`

Users have their `previousSurvival` stored as well in form of `lastAccruedEarmarkWeight` which is exactly the `previousSurvival` that can be factored out in this delta: `s1*s2(ef3 + s3*ef4)`

The sum inside brackets is exactly the fraction of users unearmarked debt that should be earmarked. The point here to understand is that `previousSurvival` is very important because it scales down the next fraction because new portion is portion of previously survived debt, not all time total.

However, when `_earmarkWeight` reaches `128` ("all of unearmarked debt has been earmarked"), `previousSurvival` becomes 0 ("nothing survived"). To safeguard against division by 0, it's set to 1:

```js
uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
if (previousSurvival == 0) previousSurvival = ONE_Q128;
```

This means that `_mulQ128(previousSurvival, earmarkedFraction)` product will always be just an `earmarkedFraction`, NOT scaled down by `previousSurvival`.

This will make the difference between users `lastAccruedEarmarkWeight` and `previousSurvival` look like this: `1*ef3 + 1*ef4`

The problem is that 1\*ef4 is not scaled down by previous survival and this will make users earmarked debt inflated which will lead to overall debt inflated. Note that the vulnerability does not require any malicious actors as system will break by itself once `_earmarkWeight` reaches `128` in decimal.

## Impact Details

After `_earmarkWeight`reaches `128`, new inflows of debt will be more and more inflated on each earmark event. This will cause core system invariant to break:

`Sum of all users debt > Global debt`

Additionally, users will become immediately undercollateralized and liquidatable because of `account.debt` inflation. This can cause several issues down the line as accounts debt becomes bigger than collateral which makes it look like bad debt accrued so every liquidation will take outsourced fees or none which will remove incentive for liquidators.

On each liquidation that goes through `globalDebt` will decrease more than it should because of inflated `account.debt` subtraction, which means global state will also be incorrect which breaks the whole protocol.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1117-L1129>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1042-L1095>

## Proof of Concept

## Proof of Concept

```js
    function test_earmarkOverweightPoc() external {
        // 1. Create 2 user 1 and user 2 and mint both 1M yield tokens
        uint256 _depositAmount = 112e18; // such that at 90% LTV user can borrow 100e18
        uint256 _mintAmount = 100e18;
        address user1 = address(0x123);
        address user2 = address(0x456);
        deal(address(vault), user1, _depositAmount);
        deal(address(vault), user2, _depositAmount);

        // 2. Mint 100 synth with user1 and create redemption to bring weight over LOG2NEGFRAC_1 (128)
        vm.startPrank(user1);
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(_depositAmount, user1, 0);
        uint256 user1TokenId = AlchemistNFTHelper.getFirstTokenId(address(user1), address(alchemistNFT));

        alchemist.mint(user1TokenId, _mintAmount, user1); // +100 unearmarked
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), type(uint256).max);
        transmuterLogic.createRedemption(_mintAmount); // +100 to earmark

        vm.roll(vm.getBlockNumber() + 5_256_000);
        alchemist.poke(user1TokenId);
        vm.stopPrank();
        // _earmarkWeight: 128, _survivalAccumulator: 1

        // 3. Mint 100 synth with user2
        vm.startPrank(user2);
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(_depositAmount, user2, 0);
        uint256 user2TokenId = AlchemistNFTHelper.getFirstTokenId(address(user2), address(alchemistNFT));
        alchemist.mint(user2TokenId, _mintAmount, user2); // +100 unearmarked
        vm.stopPrank();

        // 4. Create several redemptions with user1 to inflate _survivalAccumulator
        deal(address(alToken), user1, 100e18);
        vm.startPrank(user1);
        transmuterLogic.createRedemption(60e18); // +60 to earmark
        vm.roll(vm.getBlockNumber() + 5_256_000);
        alchemist.poke(user1TokenId);
        // _earmarkWeight: 129.321, _survivalAccumulator: 1.6 delta => +0.6

        transmuterLogic.createRedemption(20e18); // +20 to earmark
        vm.roll(vm.getBlockNumber() + 5_256_000);
        alchemist.poke(user1TokenId);
        // _earmarkWeight: 130.321, _survivalAccumulator: 2.1 delta => +0.5, should be 0.4*0.5= +0.2

        transmuterLogic.createRedemption(10e18); // +10 to earmark
        vm.roll(vm.getBlockNumber() + 5_256_000);
        alchemist.poke(user1TokenId); 
        vm.stopPrank();
        // _earmarkWeight: 131.321, _survivalAccumulator: 2.6 delta => +0.5, should be 0.2*0.5= +0.1

        // 5. Claim half redemptions to bring _survivalAccumulator down so it's not capped with earmarkRaw check
        vm.startPrank(user1);
        transmuterLogic.claimRedemption(1); // 100e18 claimed
        alchemist.poke(user1TokenId); 
        alchemist.poke(user2TokenId);
        vm.stopPrank();
        // _earmarkWeight: 131.321, _survivalAccumulator: 1.231

        // 6. Show users and global earmarked rounded to 2 decimals
        // Global
        emit log_named_uint("totalDebt", alchemist.totalDebt() / 1e16); // 100e18
        emit log_named_uint("cumulativeEarmarked", alchemist.cumulativeEarmarked() / 1e16); // 90e18
        // User 1 CDP
        (uint256 collateral1, uint256 debt1, uint256 earmarked1) = alchemist.getCDP(user1TokenId);
        console.logString("User1:");
        emit log_named_uint("collateral", collateral1 / 1e16); // 47.36 e18
        emit log_named_uint("debt", debt1 / 1e16); // 47.36 e18
        emit log_named_uint("earmarked", earmarked1 / 1e16); // 47.36 e18
        // User 2 CDP
        (uint256 collateral2, uint256 debt2, uint256 earmarked2) = alchemist.getCDP(user2TokenId);
        console.logString("User2:");
        emit log_named_uint("collateral", collateral2 / 1e16); // 85.78 e18 (should be 52.64)       INFLATED 163%
        emit log_named_uint("debt", debt2 / 1e16); // 85.78 e18 (should be 52.64)                   INFLATED 163%
        emit log_named_uint("earmarked", earmarked2 / 1e16); // 75.78 e18 (should be 42.64)         INFLATED 177%

        // Logs:
        //   cumulativeEarmarked: 90.00
        //   totalDebt: 100.00

        //   User1:
        //   collateral: 62.00
        //   debt: 47.36
        //   earmarked: 47.36

        //   User2:                         UNDERCOLLATERALIZED
        //   collateral: 62.00
        //   debt: 85.78
        //   earmarked: 75.78

        // Users combined debt: 13.31       INFLATED
        // Users combined earmark: 12.31    INFLATED
    }
```
