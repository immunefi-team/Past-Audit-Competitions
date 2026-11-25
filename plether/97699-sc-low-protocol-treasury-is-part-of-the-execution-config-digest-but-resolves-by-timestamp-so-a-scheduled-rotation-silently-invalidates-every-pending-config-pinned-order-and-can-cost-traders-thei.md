# \#97699 [SC-Low] Protocol treasury is part of the execution config digest but resolves by timestamp, so a scheduled rotation silently invalidates every pending config-pinned order and can cost traders their position margin

**Submitted on Nov 2nd 2026 at 18:30:42 UTC by @crypt0n1337 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97699
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

`currentExecutionConfigHash()` pins every external order to the current execution configuration. The protocol treasury address is one of its 19 components (`OrderLifecycleBook.sol:172-203`, treasury at line 199).

Since commit `1209ca8a` (#130, 2026-10-27) the treasury is no longer stored state: `CfdTreasury.protocolTreasury()` resolves it from `block.timestamp` (`CfdTreasury.sol:45-47` to `:100-107`). When a successor has been scheduled and `block.timestamp >= _currentTreasury.lockedUntil`, the getter returns the pending successor with no storage write, no event and no `activeConfigVersion` increment.

Consequences:

1. The digest changes at a publicly predictable instant, because `pendingProtocolTreasury()` exposes `activationTime`.
2. Every order committed with the previous digest becomes terminally invalid: the execution sidecar classifies `expectedConfigHash != currentExecutionConfigHash()` as terminal `ConfigMismatch` (`OrderRouterExecutionSidecar.sol:913-917` then `OrderLifecycleBook.sol:635-645`) before any oracle read.
3. For open orders the reserved execution bounty is forfeited to whoever finalizes them (`OrderLifecycleBook.sol:717-721`), and cleanup is permissionless and batched (`OrderRouter.sol:198-204` then `OrderRecoverySidecar.sol:67-115`, up to 64 orders per call).
4. For close orders the order reserved no bounty, so the loss is not the bounty: the protective exit is destroyed while the position stays open. If the market gaps before the trader can build and commit a new intent, the position is liquidated and the trader loses the pledged margin.

The second consequence is measured. On the release risk parameters (maintenance 0.1 percent, initial margin 0.2 percent, execution fee 0.04 percent, vpiFactor 0.01e18, maxSkew 0.4) a maximum-leverage position becomes liquidatable after a gap of 0.0999 percent, and the measured loss against the counterfactual where the close executes is 209.797698 USDG on a 100 000 USDG notional, rising to 309.897542 USDG at a 0.2 percent gap, which is 104.9 to 154.9 percent of the pledged margin. At 10x leverage the same defect costs 9 060.000000 USDG when a 9 percent gap arrives in the reaction window.

An armed stop-loss is only partial mitigation, not a bypass: it is unpinned, so it survives the rotation (measured: the protected path ends at the same value with and without the rotation), but it triggers on the gapped price, so it still realises 95.35 percent of the pledge as a loss. If a liquidator wins the race, protection ends as Liquidated with a larger loss. Also, while a protection is armed an ordinary pinned close cannot be committed at all (`OrderRouter__ProtectionActive`), so the two exits are mutually exclusive.

This contradicts the documented change model. `SECURITY.md:105-107` states that each successful Engine/Router finalization changes the digest and lists config finalization as the mutation path. A pure timestamp transition is a fifth, unobservable path. Operators and indexers cannot distinguish "orders died because the configuration changed" from any other terminal cause, because nothing was emitted and no version moved.

Why this is not a listed known issue: `CfdTreasury.sol` is absent from the Cecuro audit commit (`6d6622f2`, 2026-10-16) and from the AI scan; the feature arrived in commit `1209ca8a` on 2026-10-27, two days before the freeze. MED-3 has a different root (dust held by the old treasury blocked rotation, and the fix removed a balance check). MED-13 covers the opposite direction (a protection is not bound to the hash) while confirming that config-pinned orders invalidate on a config change by design; it does not describe a digest that changes by timestamp without an event or version, nor the resulting cascade onto every pending order.

## Proof of Concept

Environment: forge 1.8.1, solc 0.8.35+commit.47b9dedd, optimizer 200, via_ir enabled, HEAD a82d4fef. Local deployment only, no network access and no transactions to any public chain. No in-scope contract is modified; only Pyth updates and the settlement token are mocked, which the program allows.

Test files:
packages/perps/test/poc/F1_TreasuryHashCascade.t.sol (4 tests)
packages/perps/test/poc/F1_RotationLiquidation.t.sol (4 tests)
packages/perps/test/poc/F1_RotationLiquidationReleaseConfig.t.sol (4 tests)
Run: forge test --root packages/perps --match-path "test/poc/F1_*.t.sol" -vv
Result at the in-scope commit: 12 passed, 0 failed.

Part 1, the digest flips on a timestamp alone.
1. digest before 0x0aef4624...3a503a. The owner calls setProtocolTreasury(A, lock) and proposeProtocolTreasury(B, 0), then vm.warp(lockedUntil). digest after 0x7d2edf69...020593.
2. vm.record() and vm.accesses(engine) report zero storage writes, and slot values read before and after are byte identical. protocolTreasury() flips 0xFEE50001 to 0xFEE50002.
3. An order committed with the old digest terminates with ConfigMismatch (reason 4) before any oracle call (updatePriceFeedsCallCount == 0), and its reserved bounty 200000 goes to the caller.

Part 2, a killed close leaves the position exposed (harness risk parameters, maintenance 100 bps, initial 150 bps).
1. 10x case, 100 000 notional with 10 000 margin. The close order is pinned to the pre-rotation digest with a 60 s window, the rotation activates inside that window, execution returns ConfigMismatch, the position stays open (size 100 000e18, keeper paid 0), then a 9 percent gap arrives and the engine liquidates: PnL -9 000, equity 859.999859 below maintenance 1 090, charge 100.
2. Counterfactual at the same timestamp: no successor queued, the close executes at 1.00, size becomes 0, the gap finds no position (CfdEngine__NoPositionToLiquidate). The control account keeps 49 919.999859 against 40 859.999859 in the attack world.
3. Impact = difference of the two measured worlds = 9 060.000000 USDG. The test asserts the impact is greater than half the margin and never exceeds the posted margin, so the loss is the trapped margin, not a bounty.
4. 50x case, 300 000 notional with 6 000 margin and a 1 percent gap: PnL -3 000, equity 2 579.998492 below maintenance 3 030, impact 3 180.000000 USDG.

Part 3, release risk parameters (maintenance 10 bps, initial 20 bps, execution fee 4 bps, vpiFactor 0.01e18, maxSkew 0.4, bountyBps 10, minBounty 1 USDG, keeperShare 5000, protocolShare 0). The test applies them through _setRiskParams and reads them back from engine.riskParams().
1. A position with marginDelta equal to the initial margin does not open: the fee (40 USDG), the VPI charge (49.900199 USDG) and the liquidation reserve (100 USDG) are carved out of the deposit, and initial margin is checked against the post-open pledge. Depositing 389.900200 USDG yields a pledge of 200.000001 USDG, which is 4 999 999 bps, about 500x.
2. Minimal gap that makes the position liquidatable is 0.0999 percent (price 100 099 900 with 8 decimals). One atom lower is still solvent, at that price the account is liquidatable. The boundary is found by binary search and asserted from both sides.
3. Loss at the minimal gap: 199.900157 USDG on 100 000 notional, which is 99.95 percent of the pledged margin. Impact against the counterfactual close at 1.00 is 209.797698 USDG, because the counterfactual also earns a VPI rebate and pays only the close fee.
4. Grid: 0.05 percent does not liquidate (mark impact 59.897541), 0.10 percent gives 209.897698, 0.15 percent gives 259.897698, and 0.20 percent and above saturate at 309.897542 USDG, which is 154.9 percent of the pledge. The liquidation charge is always 100 USDG because it is capped by the liquidation reserve.

Part 4, protection is partial mitigation only.
1. An armed stop-loss at 1.0005 is unpinned, so the rotation does not touch it: after the warp the digest changed and the protection is still Armed. It then triggers (Triggered, generated close with expectedConfigHash = 0, Executed) and closes the position.
2. Value with and without the rotation is identical (49 719.399999), so on the protected path the rotation costs nothing.
3. It is not a bypass: the protection triggers on the gapped price of 1.01, so it realises 190.699802 USDG of loss (95.35 percent of the pledge) and does not restore the 1.00 exit. If a liquidator wins the race, production executeLiquidation takes the position, the protection ends as Liquidated, and the loss is 300.200001 USDG.
4. While a protection is armed, an ordinary pinned close cannot be committed at all (OrderRouter__ProtectionActive), so the two exits are mutually exclusive.

Negative controls: without the queued successor the digest does not change, the close executes, and the gap cannot liquidate an account that has no position. The impact assertions compare two worlds at the same timestamp and are bounded by the posted margin.
