# 56702 sc critical claimredemption would not return all alasset that is not get converted to myt in some case

**Submitted on Oct 19th 2025 at 16:22:21 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56702
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/Transmuter.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`claimRedemption` function in the `Transmuter` is designed to handle cases where the system can't obtain enough yield tokens (MYT) to fulfill a user's entire claim. It correctly caps the amount of MYT sent to the user. but the function fails to refund the corresponding portion of the user's synthetic asset (`alAsset`) that would be burned but not converted. resulting in a permanent loss of funds for the user, as their `alAsset` is destroyed without them receiving the equivalent value in yield tokens.

## Vulnerability Details

the issue is because how the cap mechanism does not completely handle if the MYT in transmuter after redeem does not sufficient to cover user transmuted amount:

```solidity
        uint256 totalYield = alchemist.convertDebtTokensToYield(scaledTransmuted);

        // Cap to what we actually hold now (handles redeem() rounding shortfalls).
        uint256 balAfterRedeem = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
@>      uint256 distributable = totalYield <= balAfterRedeem ? totalYield : balAfterRedeem;

        uint256 syntheticFee = amountNottransmuted * exitFee / BPS;
@>      uint256 syntheticReturned = amountNottransmuted - syntheticFee;
```

in `distributable` , if the `totalYield` which user should get is greater than `balAfterRedeem` then it would cap the `distributable` into `balAfterRedeem` . but if we check the `syntheticReturned` there are no adjustment if the `distributable` is capped, making it always return the `syntheticReturned` amount which if converted to yield token it would be greater than `distributable` amount.

## Impact Details

user would burn more alAsset compared to what MYT amount they received, making it user loss.

if transmuter would sent lower MYT amount than what user would burn, we should only burn the alAsset that equal to actual MYT that get sent. the cap should not only applied on the MYT side, but the user amount side.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L237-L245>

## Proof of Concept

## Proof of Concept

to understand this better, lets list the preconditions:

1. Transmuter have some MYT on the contract (via user repay). this also reduce the `cumulativeEarmark` which is used to cap redeem amount on step 4.
2. after some time price of MYT dropped
3. user1 `claimRedemption`
4. now the cap happen because the `Alchemsit::redeem` would return fewer MYT (because of step 1 and 2 combined). this combined with what Transmuter held before redeem would not sufficient for what user1 supposed to get. for example user1 would get 100 MYT but contract only held 80 MYT, the 80 MYT would get sent to user1
5. user1 still burn alAsset equivalent of 100 MYT, where he get only 80 MYT. user1 loss 20 MYT worth of alAsset for nothing because of it

now add the test into `src/test/AlchemistV3.t.sol`:

```solidity
    function test_excessAlAssetFromCappedRedeemDoesNotReturnedToUser() public {
        // 1. create position
        uint256 amount = 100e18;

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 borrowedAmount = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, borrowedAmount, address(0xbeef));
        vm.stopPrank();

        // 2. redemption using the same amount
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), borrowedAmount);
        transmuterLogic.createRedemption(borrowedAmount);
        vm.stopPrank();

        // 3. maturing 2/3 transmute amount
        vm.roll(block.number + 5_256_000 * 2 / 3);

        // 4. 0xbeef repay debt the position
        vm.prank(address(0xbeef));
        alchemist.repay(borrowedAmount, tokenId);

        // 5. simulate price drop
        console.log("price", IMockYieldToken(mockStrategyYieldToken).price());
        deal(address(IMockYieldToken(mockStrategyYieldToken).underlyingToken()), address(mockStrategyYieldToken),  amount / 2);
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(amount);
        console.log("price", IMockYieldToken(mockStrategyYieldToken).price());

        // 6. Redeem.
        uint256 dadAlAssetBalBefore = alToken.balanceOf(address(0xdad));
        uint256 dadMYTBalBefore = vault.balanceOf(address(0xdad));
        uint256 feeReceiverAlAssetBalBefore = alToken.balanceOf(protocolFeeReceiver);
        uint256 feeReceiverMYTBalBefore = vault.balanceOf(protocolFeeReceiver);
        vm.prank(address(0xdad));
        transmuterLogic.claimRedemption(1);

        // 7. get how many alAsset and MYT 0xdad receive back, and the one get sent to protocolFeeReceiver
        uint256 dadAlAssetBalAfter = alToken.balanceOf(address(0xdad));
        uint256 dadMYTBalAfter = vault.balanceOf(address(0xdad));
        uint256 feeReceiverAlAssetBalAfter = alToken.balanceOf(protocolFeeReceiver);
        uint256 feeReceiverMYTBalAfter = vault.balanceOf(protocolFeeReceiver);
        uint256 alAssetReturned = (dadAlAssetBalAfter - dadAlAssetBalBefore) + (feeReceiverAlAssetBalAfter - feeReceiverAlAssetBalBefore);
        uint256 mytOut = (dadMYTBalAfter - dadMYTBalBefore) + (feeReceiverMYTBalAfter - feeReceiverMYTBalBefore);

        // 8. compare mytOut with actual alAsset that get burned, by converting it to current conversion to yield
        // we can get this by removing alAssetReturned from total amount that is used when creating position, which is borrowedAmount
        uint256 alAssetBurnedInYield = alchemist.convertDebtTokensToYield(borrowedAmount - alAssetReturned);
        assertApproxEqAbs(mytOut, alAssetBurnedInYield, 1e18);
    }

```

run with `forge test --mt test_excessAlAssetFromCappedRedeemDoesNotReturnedToUser`

```bash
Failing tests:
Encountered 1 failing test in src/test/AlchemistV3.t.sol:AlchemistV3Test
[FAIL: assertion failed: 89910000000000000009 !~= 120120000000000000011 (max delta: 1000000000000000000, real delta: 30210000000000000002)] test_excessAlAssetFromCappedRedeemDoesNotReturnedToUser() (gas: 3959541)

Encountered a total of 1 failing tests, 0 tests succeeded
```

this prove that the mytOut is lower than what user alAsset burned converted to yield
