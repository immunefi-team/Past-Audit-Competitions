# \#98253 [SC-Insight] Flawed validation on order admissions for opposing sides when failure is predictable

**Submitted on Nov 6th 2026 at 10:51:38 UTC by @blackgrease for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98253
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

`OrderLiquidationBatchLogic._queuedPosition()` incorrectly accounts for expired full CLOSE orders when validating opposing-side OPEN orders. This allows an OPEN order to be accepted even though its execution failure is already predictable - unnecessarily exposing the trader to a loss of their execution bounty.

### Root Cause

The Router calculates an account's projected position by processing its pending orders. However, `OrderLiquidationBatchLogic._queuedPosition()` subtracts the size of a pending CLOSE without checking whether its execution deadline has passed.

```solidity
    function _queuedPosition(
        IOrderLiquidationBatchHost host,
        ICfdEngineCore engine,
        address account
    ) private view returns (QueuedPositionView memory queuedPosition) {
            //--SNIP---
                if (pending.isClose) {
                    if (queuedPosition.exists && pending.side == queuedPosition.side) {
                        queuedPosition.size =
                            queuedPosition.size > pending.sizeDelta ? queuedPosition.size - pending.sizeDelta : 0;
                        if (queuedPosition.size == 0) {
                            queuedPosition.exists = false;
                        }
            //--SNIP---
    }
```
_Code location:_ https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L384-L391

When a full close order has expired, it can no longer execute successfully, meaning the projection is provably inaccurate. However, the Router still assumes the expired order will close the existing position. `_validatedOpenBounty()` accepts the opposing-side OPEN based on the incorrect projection. 

When the expired CLOSE is cleared from the queue, the existing position remains unchanged and the OPEN predictably fails with `MUST_CLOSE_OPPOSING` - as a trader cannot hold two opposing sides at the same time. 


### Example Scenario

1. Alice holds a 10,000-unit SHORT.
2. Alice queues a full SHORT close with a 20 second execution window.
3. The deadline passes. The close is still Pending in the queue because expiry requires an on-chain cleanup transaction.
4. Alice submits a LONG open. The Router projects the expired close as successful, so it accepts the LONG and reserves an execution bounty.
5. A keeper expires the SHORT close (zero bounty; the SHORT remains open), then attempts the LONG.
6. The LONG fails with `MUST_CLOSE_OPPOSING`, causing Alice to unfavourably lose the LONG's execution bounty to the keeper - despite the order being unexecutable at submission.

Although the protocol acknowledges that an opposing-side OPEN may fail if a preceding CLOSE fails, this case differs because the CLOSE has already expired before the OPEN is submitted. The resulting OPEN failure is therefore predictable at admission and could have been prevented through proper validation, avoiding an unnecessary bounty loss.

## Impact

**Logic flaw resulting in loss of execution bounty:** Insufficient validation allows an OPEN order to be accepted _even though the protocol already knows its execution will fail_. Consequently, the trader unnecessarily loses their execution bounty, as the keeper still performs its assigned task.

An order whose execution failure can be predicted prior should not be allowed to enter into the orderbook. The protocol already has rigourous input validation to prevent orders that will ultimately fail from being accepted.


## Recommendations

Update `_queuedPosition()` to check the execution deadlines of pending orders when calculating an account's projected position. Orders that have already expired should be excluded from the calculation.

This prevents the Router from admitting opposing-side OPEN orders based on CLOSE orders that can no longer execute successfully.


## Link to Proof of Concept

https://gist.github.com/blackgrease/40e526835399d8cd8a5bd6299dd4009c

## Proof of Concept

The PoC in the private gist link - https://gist.github.com/blackgrease/40e526835399d8cd8a5bd6299dd4009c - reproduces the issue.

The PoC:
1. Confirms that an opposing-side OPEN is rejected when an existing position has no pending CLOSE.
2. Queues a CLOSE, lets it expire, and verifies that the expired order remains pending.
3. Demonstrates that the Router incorrectly admits an opposing-side OPEN based on the expired CLOSE.
4. Expires the CLOSE and verifies that the original position remains unchanged.
5. Attempts to execute the OPEN and confirms it fails with `MUST_CLOSE_OPPOSING`, causing the trader to lose the reserved execution bounty to the keeper.
6. Confirms that opposing-side OPEN admission is correctly rejected after cleanup.

**Instructions to run:**

- In the project root:
  - Run with: `forge test --mt testPoC_ExpiredCloseAdmitsOpposingOpenAndPaysKeeperOnFailure --root packages/perps --threads 1 -vvvv`


