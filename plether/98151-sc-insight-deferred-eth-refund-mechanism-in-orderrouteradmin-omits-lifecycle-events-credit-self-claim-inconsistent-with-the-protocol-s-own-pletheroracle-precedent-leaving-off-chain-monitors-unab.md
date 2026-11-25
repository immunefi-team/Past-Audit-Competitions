# \#98151 [SC-Insight] Deferred-ETH refund mechanism in OrderRouterAdmin omits lifecycle events (credit + self-claim), inconsistent with the protocol's own PletherOracle precedent, leaving off-chain monitors unable to detect failed/deferred user refunds

**Submitted on Nov 5th 2026 at 14:24:51 UTC by @edjose128 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98151
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

### Brief/Intro

After an order is executed or a position is liquidated, the router sends the leftover ETH (whatever the caller paid above the Pyth fee) back to the user. If that send fails, either because the recipient reverts or because it runs out of the 30,000 gas the router forwards, the money isn't lost. The router records it in a `claimableEth` ledger inside `OrderRouterAdmin` so the user can withdraw it later.

The problem is that this ledger doesn't emit events on two of its three paths. When a refund gets deferred, nothing is logged. When the user later claims it with `claimBalance()`, nothing is logged either. Only `claimBalanceTo()` emits anything. This matters because the exact same mechanism exists in `PletherOracle`, and there it emits an event on all three paths. So the protocol already decided these events are worth having; the router copy just never got them. No funds are at risk, the balances are all readable from the public mapping, but anything watching the chain through logs (a monitoring bot, a user dashboard, the team's own alerting) is blind to refunds that fail on the router and to users recovering them afterwards.

### Vulnerability Details

The ledger has three operations: a credit (a refund failed, so the amount is parked), a self-claim (the owner pulls their own balance), and a claim-to (the owner pulls it to some other address). `OrderRouterAdmin` only declares one event for all of this, at line 110:

```solidity
event EthRefundClaimedTo(address indexed beneficiary, address indexed recipient, uint256 amount);
```

The credit path is `creditClaimableEth`, which the router calls when a direct send fails. It updates the mapping and returns without logging anything (`OrderRouterAdmin.sol:264-275`):

```solidity
function creditClaimableEth(address beneficiary, uint256 amount) external payable onlyRouterHost {
    if (amount == 0) {
        return;
    }
    if (msg.value != amount) {
        revert OrderRouterAdmin__EthAmountMismatch();
    }
    claimableEth[beneficiary] += amount;   // no event here
}
```

The self-claim path is `claimBalance`, which clears the balance and transfers it out, again with no event (`OrderRouterAdmin.sol:279-281`, forwarding to `_claimBalance` at 293-308):

```solidity
function claimBalance() external {
    _claimBalance(payable(msg.sender));   // nothing logged on this path
}
```

The only path that logs is `claimBalanceTo` (`OrderRouterAdmin.sol:286-291`):

```solidity
function claimBalanceTo(address payable recipient) external {
    uint256 amount = _claimBalance(recipient);
    emit EthRefundClaimedTo(msg.sender, recipient, amount);
}
```

Now compare `PletherOracle`. It has the same `claimableEth` mapping (`PletherOracle.sol:98`) and the same three operations, but every one of them logs. The credit path emits `EthRefundDeferred` inside `_refundValue` (`PletherOracle.sol:1051-1052` and again at `1066-1067`):

```solidity
claimableEth[refundRecipient] += amount;
emit EthRefundDeferred(refundRecipient, amount);
```

The self-claim emits `EthRefundClaimed` (`PletherOracle.sol:384-386`):

```solidity
function claimEthRefund() external override nonReentrant {
    uint256 amount = _claimEthRefund(payable(msg.sender));
    emit EthRefundClaimed(msg.sender, amount);
}
```

And the claim-to emits `EthRefundClaimedTo` (`PletherOracle.sol:390-394`). All three signatures are declared together in the shared interface `IPletherOracle.sol:231,235,240`:

```solidity
event EthRefundDeferred(address indexed recipient, uint256 amount);     // 231
event EthRefundClaimed(address indexed recipient, uint256 amount);      // 235
event EthRefundClaimedTo(address indexed beneficiary, address indexed recipient, uint256 amount); // 240
```

So the two events the router is missing (`EthRefundDeferred`, `EthRefundClaimed`) already exist and are already used a few files over, for the identical bookkeeping.

It's worth showing that the silent credit path is actually reached in normal operation, not just in some contrived case. The refund goes out through a gas-capped call, and the fallback to the ledger happens on any failure. From `OrderExecutionHandler._sendEth` (`router/OrderExecutionHandler.sol:30-45`):

```solidity
(bool ok,) = payable(to).call{value: amount, gas: 30_000}("");
if (!ok) {
    OrderRouterAdmin(admin).creditClaimableEth{value: amount}(to, amount);
}
```

The liquidation side has the same pattern in `OrderLiquidationBatchLogic._sendEth` (`router/OrderLiquidationBatchLogic.sol:846-858`), reached from four refund sites (lines 580, 621, 724, 758). A plain EOA will accept the transfer fine, but a contract account whose `receive`/`fallback` costs more than 30,000 gas will fail it and land in the credit path. Smart-contract wallets and multisigs routinely cost more than that, so this isn't an edge case you have to go looking for. When it happens the user's ETH is safe, but it has quietly turned into a pending claim that produces no log.

The two paths that stay silent are the two an operator would most want to see: a refund failing (the credit) and the user getting their money back (the self-claim). The one event that does fire, `EthRefundClaimedTo`, covers the least-used path, withdrawing to a third address. Anything reconstructing refund state from logs on the router ends up with an incomplete picture, while the same tooling pointed at the oracle gets the full story. This fits Immunefi's Security Best Practices insight category, and reads as a consistency gap between two implementations of one mechanism rather than a matter of taste, precisely because the protocol already made the call in `PletherOracle`.

### Recommendation

Give `OrderRouterAdmin` the same two events the oracle already uses:

```solidity
event EthRefundDeferred(address indexed recipient, uint256 amount);
event EthRefundClaimed(address indexed recipient, uint256 amount);
```

Emit `EthRefundDeferred(beneficiary, amount)` at the end of `creditClaimableEth`, right after the `claimableEth` increment, and `EthRefundClaimed(msg.sender, amount)` in `claimBalance` once `_claimBalance` returns. That puts the router's refund logging on the same footing as the oracle's and lets off-chain systems track deferred ETH the same way across both.

## Proof of Concept

Immunefi's Insight guidelines ask the PoC section to describe the conditions under which the insight is worth acting on, looking at the system as a whole, rather than to supply a running exploit. There is no exploit here. No funds are at risk and every balance in the ledger is readable from the public `claimableEth` mapping. What proves the point is the code itself, and the fact that the same protocol already logs these events in `PletherOracle`. Here is the sequence that makes the gap concrete.

Start with a case that happens in ordinary use. A user, or a contract acting for one, sends an order or gets liquidated through the router and pays more than the Pyth fee, so a refund of `msg.value - pythFee` is owed back to them. The recipient is a contract account whose `receive`/`fallback` costs more than the 30,000 gas the router forwards in `_sendEth` (`OrderExecutionHandler.sol:38`). This is normal for smart-contract wallets and multisigs.

The bounded call fails, so the router falls back to the ledger and calls `OrderRouterAdmin.creditClaimableEth{value: amount}(to, amount)` (`OrderExecutionHandler.sol:41`). The user's `claimableEth[to]` goes up by the refund amount.

That credit produces no log (`OrderRouterAdmin.sol:264-275`). Anything watching the router through events sees nothing happen. It can't warn the user that their refund failed and is now sitting as a pending claim, it can't reconcile the ETH the router is holding against what it owes from log history alone (it would have to scan the mapping's storage across every address, which no event-driven alerting pipeline does), and it can't tie the deferral back to the order or liquidation that caused it.

The same blind spot repeats when the money leaves. The user calls `claimBalance()` to pull the refund back out (`OrderRouterAdmin.sol:279-281`). The transfer goes through, the balance is cleared, and still nothing is logged (`_claimBalance`, `OrderRouterAdmin.sol:293-308`). A dashboard that tracks unclaimed refunds from logs never sees the claim, so it keeps showing a liability that was already settled.

Set that against `PletherOracle`, which runs the identical ledger and logs both of these. It emits `EthRefundDeferred` on the credit (`PletherOracle.sol:1052,1067`) and `EthRefundClaimed` on the self-claim (`PletherOracle.sol:386`), both declared in the shared interface at `IPletherOracle.sol:231,235`. Point the same monitoring at the oracle and you get the whole refund history; point it at the router and you get one event out of three. The fix is one emit on each of the two silent paths, it touches no funds, and it costs only the two log writes. What you get back is a refund trail that reads the same whether you're watching the oracle or the router, which is what lets the team and any integrator build accounting, user notifications, and incident alerting on top of it without a hole on the router side.
