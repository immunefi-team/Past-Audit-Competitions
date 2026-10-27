# #46924 \[SC-Low] Last user may exit with almost all of his values, but he'll purposefully leave a small 1e18 or a little more to grief \`destroy()\`

**Submitted on Jun 6th 2025 at 08:56:36 UTC by @onthehunt for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46924
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`CollateralPool::destroy()` is used to remove destroy an agent and collateral pool, but he may be griefed from that for a very small amount.

## Vulnerability Details

Let's say that agent wants to remove himself from the protocol for some reason and announce destroy via `announceDestroyAgent()`, which will make destroying the agent possible after the specified `settings.withdrawalWaitMinSeconds`

Nothing is left inside the pool and he successfully calls the aforementioned function.

But a malicious user spots that and decides to call `CollateralPool.enter()`, with the smaller amount available, which will be no more than a couple of bucks, if not less.

`MIN_NAT = 1e18` which is 1e18 of flare native token, costing literal cents.

Via that, a user can block the destruction of an agent and its collateral pool for as I said, a very small amount and accessible amount of money, thus preventing the whole action.

## Impact Details

Agent and its collateral pool will never be destroyed, leaving broken functionality the reason of the bug, as we should be able to do so after announcing it, leaving the users responsible for their action (we should just destroy the pool even if there is something left, as the announcement starts only if there's nothing in the pool).

## References

<https://github.com/flare-labs-ltd/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol#L833>

<https://github.com/flare-labs-ltd/fassets/blob/main/contracts/assetManager/library/AgentsCreateDestroy.sol#L145-L176>

<https://github.com/flare-labs-ltd/fassets/blob/main/contracts/assetManager/library/AgentsCreateDestroy.sol#L134>

## Proof of Concept

1. Agent wants to destroy himself and the collateral pool as they are empty and calls `announceDestroyAgent()`
2. Alice sees that and calls `CollateralPool::enter()` with the smallest value she can enter the pool
3. The time passes that makes destroying the pool possible, but the agent has been griefed by Alice and can't do that
