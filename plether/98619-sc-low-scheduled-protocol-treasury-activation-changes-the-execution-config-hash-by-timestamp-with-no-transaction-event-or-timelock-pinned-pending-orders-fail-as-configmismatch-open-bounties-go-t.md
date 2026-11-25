# \#98619 [SC-Low] Scheduled protocol-treasury activation changes the execution-config hash by timestamp with no transaction, event or timelock: pinned pending orders fail as ConfigMismatch (open bounties go to the keeper) and commits built from the old hash are rejected

**Submitted on Nov 9th 2026 at 03:34:26 UTC by @enowk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98619
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Brief/Intro

Every fresh order pins the current execution-config digest, `OrderLifecycleBook.currentExecutionConfigHash()`, and is terminally failed as `ConfigMismatch` if the digest differs at execution. The digest includes `CfdEngine.protocolTreasury()`. That address changes in two ways that bypass the 72-hour configuration timelock every other digest input follows: the owner's immediate `setProtocolTreasury`, and a scheduled successor that becomes active purely by timestamp when the current treasury lock expires, with no transaction and no event. At that instant the digest rotates silently. All config-pinned pending orders then fail terminally; each failed open pays the trader's execution bounty to whichever keeper prunes it; pending closes (including protective closes a trader relies on) fail and must be re-committed; and any client that built a request from the digest read moments earlier is rejected with `OrderRouter__ExecutionConfigMismatch`. The documentation describes the treasury schedule and the digest separately and never states that a treasury activation invalidates in-flight orders.

## Vulnerability Details

**1. The digest pins the treasury address.** `currentExecutionConfigHash()` hashes, among identities and finalized config versions, `engine.protocolTreasury()`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L172-L207
```solidity
        bytes memory engineConfig = abi.encode(
            engine.planner(),
            engine.settlementSidecar(),
            engine.terminalNavBook(),
            engineAdmin,
            IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
            engine.protocolTreasury(),
            IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
        );
```

**2. The treasury flips by timestamp, without a transaction.** `protocolTreasury()` resolves a pending successor the moment `block.timestamp >= lockedUntil`:

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L101-L106
```solidity
    function _activeTreasury() internal view returns (TreasuryConfig storage active) {
        active = _currentTreasury;
        if (_pendingTreasury.account != address(0) && block.timestamp >= _currentTreasury.lockedUntil) {
            active = _pendingTreasury;
        }
    }
```
[`CfdTreasury.sol` L23](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L23): "Scheduled activation is resolved by timestamp and emits no automatic event." [`SECURITY.md` L151](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L151): "There is no autonomous activation event at expiry; monitors must resolve the effective getters."

**3. A digest change is terminal for pinned orders and pays the keeper.** The Router checks the digest before any oracle work and settles the order as `ConfigMismatch` ([`OrderRouterExecutionSidecar.sol` L911-L917](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L911-L917), [L405-L413](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L405-L413)). For every terminal reason other than risk-off and liquidation, the lifecycle Book requires the bounty to be paid to the executor ([`OrderLifecycleBook.sol` L716-L720](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L716-L720)), so a trader whose open was invalidated by the treasury schedule loses the open bounty (up to 0.20 USDG at release settings) to the pruning keeper.

**4. Fresh commits built from the pre-activation digest are rejected.** Commit admission requires the exact current digest ([`router/OrderLiquidationBatchLogic.sol` L294-L297](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L294-L297)). Integrations are told to read `currentExecutionConfigHash()` before constructing a request (`CANONICAL_ENTRYPOINTS.md`, "Funding and orders"). A request built one second before activation and included one second after reverts with `OrderRouter__ExecutionConfigMismatch`.

**5. This is the only digest input that changes outside the timelocked finalization model.** `SECURITY.md` L754-L792 ("Pinned financial policy and configuration") describes digest drift as the result of "finalized" Engine/Router configuration: every other input changes only through `finalizeRiskConfig`, `finalizeCalendarConfig`, `finalizeFreshnessConfig`, `finalizeRouterConfig` or `finalizeOracleConfig`, each behind a 72-hour proposal and emitting an event, or through one-time wiring. The treasury is listed as an input ([L769](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L769)) but changes through an immediate owner setter ([L239](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L239), "Instant owner controls") or by the eventless schedule above. Neither the treasury section (L114-L151) nor the pinned-policy section mentions that a treasury change invalidates pending orders.

**Why the treasury does not need to be pinned.** Pinning protects the trader against execution under a different financial policy. The treasury address only decides which clearinghouse account receives protocol fees and forfeited bounties; the trader's economics (fee rate, bounty, notional, bounds) are unchanged by it, and the fee is routed by the live getter at settlement anyway. Pinning it therefore adds a failure mode without protecting anyone.

## Impact Details

Submitted as an **Insight (Architectural / Documentation)**. Triggering requires the owner's documented, legitimate treasury governance; no untrusted actor can cause it. When it happens:

- Every config-pinned pending order at the activation instant fails terminally. Opens lose their bounty to the keeper that prunes them (release cap 0.20 USDG each; with up to 5 pending orders per account and many accounts, the aggregate is small but is a cost imposed on traders by a governance schedule they did not act on). Pending closes, including a trader's exit committed moments before, fail and must be re-committed, costing one execution window of exposure.
- Clients and keepers get no event to react to; the only signal is polling the digest. `SettlementMonitorLens.observableConfigDigest()` is advisory and separate from the Book digest.
- The immediate `setProtocolTreasury` path has the same effect on demand, so execution configuration effectively has an un-timelocked rotation lever, contrary to the model that all execution-affecting config drift is 72-hour finalized.

Possible hardening, in order of preference:
1. Remove `protocolTreasury()` from `currentExecutionConfigHash()`; fee routing uses the live getter and does not change trader economics.
2. Or materialize successor activation through an explicit, event-emitting call (and include the treasury in the Engine admin's `activeConfigVersion` so it follows the same finalization model), and document that treasury changes invalidate pending orders.
3. At minimum, document in `SECURITY.md` ("Protocol treasury commitment" and "Pinned financial policy") and `CANONICAL_ENTRYPOINTS.md` that a treasury change, scheduled or immediate, terminally fails all pinned pending orders, and advise operators to schedule activations at quiet times and keepers to watch `protocolTreasuryLockedUntil()`.

## References

- Digest inputs: [`OrderLifecycleBook.sol` L172-L207](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L172-L207)
- Timestamp activation, no event: [`CfdTreasury.sol` L23, L101-L113](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdTreasury.sol#L101-L113); [`SECURITY.md` L114-L151](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L114-L151)
- Terminal ConfigMismatch and bounty to executor: [`OrderRouterExecutionSidecar.sol` L911-L917](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L911-L917), [`OrderLifecycleBook.sol` L716-L720](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderLifecycleBook.sol#L716-L720)
- Commit-time digest requirement: [`router/OrderLiquidationBatchLogic.sol` L294-L297](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L294-L297)
- Pinned-policy model and instant owner controls: [`SECURITY.md` L754-L792](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L754-L792), [L230-L245](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L230-L245)
- Known-issue check: the Cecuro, SC Audit Studio and AI-scan reports discuss treasury rotation only in terms of balances and fee routing (Cecuro MED-3) and do not mention the execution-config digest. The program known-issues list and the SECURITY.md Known Limitations sections do not mention it either.


## Proof of Concept

## Proof of Concept

**When this insight matters:** whenever governance uses the documented treasury schedule (`setProtocolTreasury(address,uint256)` followed by `proposeProtocolTreasury`), or the immediate setter, while orders are in flight. At the activation timestamp, with no transaction and no event, the execution-config digest rotates: every pinned pending order fails terminally as `ConfigMismatch` (open bounties go to the pruning keeper, closes must be re-committed) and any request built from the previous digest is rejected at commit. It also matters for keepers and client SDKs, which have no event to react to and must poll the digest or `protocolTreasuryLockedUntil()`.

A runnable test is included. It uses the repo's own Foundry fixture (`BasePerpTest`) at the pinned commit, with no fork or RPC. Only Pyth (`MockPyth`) and USDG are mocked; nothing under `packages/perps/src` is modified. Release risk parameters from `DEPLOYMENT.md` are used. The fixture's router is the repo's `LegacyOrderRouterHarness` (`test/utils`): the production `OrderRouter` plus a test-only scalar `commitOrder` wrapper that forwards to the production V3 commit and pins the current digest exactly as a client would; the production V3 `commitOrder(OrderRequest)` is also called directly for the stale-hash rejection.

Setup and run:
```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the file below as packages/perps/test/poc/TreasuryActivationConfigFlipPoC.t.sol
FOUNDRY_TEST=test/poc forge test --root packages/perps --match-path test/poc/TreasuryActivationConfigFlipPoC.t.sol -vv
```

Expected output:
```
[PASS] test_scheduledTreasuryActivation_failsPinnedOrders_andRejectsPreparedCommits() (gas: 2659956)
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

What the assertions prove:
1. The owner locks treasury T1 for one hour and schedules successor T2 (documented flow). The digest is `hashBefore`.
2. Two seconds before activation, Alice commits an ordinary open; its pinned `expectedConfigHash` equals `hashBefore` and it carries a nonzero execution bounty.
3. At the activation timestamp, with no Engine transaction, `protocolTreasury()` returns T2 and `currentExecutionConfigHash()` differs from `hashBefore`.
4. A keeper executes the head: the result is `Failed` / `ConfigMismatch`, Alice has no position, and the keeper's clearinghouse balance increased by exactly Alice's bounty.
5. A production V3 request built with `hashBefore` reverts with `OrderRouter__ExecutionConfigMismatch(hashBefore, hashAfter)`; the same request with `hashAfter` is accepted (control).

`packages/perps/test/poc/TreasuryActivationConfigFlipPoC.t.sol`: see the attached file.

