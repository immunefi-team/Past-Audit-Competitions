# \#97602 [SC-Low] A scheduled treasury rotation changes the execution config hash with no transaction, so every pinned pending order becomes clearable by anyone and the clearer is paid each victim's open bounty

**Submitted on Nov 2nd 2026 at 14:11:55 UTC by @Griffinf for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97602
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Severity claimed: Low (Insight).

`OrderLifecycleBook.currentExecutionConfigHash()` hashes `engine.protocolTreasury()` together with the contract identities and admin config versions (`OrderLifecycleBook.sol:190-203`). Every other input to that hash changes only through a finalize transaction that bumps a version. The treasury does not: `CfdTreasury` activates a proposed successor purely by timestamp when the current lock expires (`CfdTreasury.sol:45, :101`), and the immediate `setProtocolTreasury` has no timelock. At the expiry second the hash changes with no transaction and no event. From that moment every pending order whose `expectedConfigHash` is nonzero is "config-invalid", and anyone may call `clearInvalidOrders` on it (`OrderRecoverySidecar.sol:88-99`). The terminal receipt for a `ConfigMismatch` open pays the order's reserved execution bounty to the caller as if they had executed it (`OrderLifecycleBook.sol:712-722`, `receipt.executor`), so the clearer collects the bounty of every victim whose order was cancelled by a governance event the victim could not have reacted to.

Precondition, stated plainly: the owner must have scheduled a treasury rotation, which the program lists as a trusted, immediate owner action ("set the treasury"). The program's Trusted Roles section states: "In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol." This report is exactly that shape; no trusted role acts maliciously or outside its documented procedure. That is an ordinary governance action and the feature was added in the un-audited PR #130 (`1209ca8a`), after the Cecuro review. Nothing an unprivileged party does triggers the rotation. What an unprivileged party does is harvest it: watch for the lock expiry, then clear every pinned order in the same block and take the bounties. I claim Low because the per-order amount is the bounty cap (0.20 USDG at the deployed `minBountyAssets`), not the order margin, and because governance-triggered invalidation of pinned orders is a documented mechanism (`SECURITY.md:765-778`). I report it because (a) it is the only hash input that changes without a transaction, so users and keepers get no event to react to, (b) it hits every pending order at once, and (c) the bounty goes to the clearer rather than back to the account, which turns a config change into a small transfer from users to an opportunist.

The expected pushback, answered: "config drift producing `ConfigMismatch` is intended." It is, for changes that go through a finalize transaction with a 72-hour timelock and emit an event at the moment they take effect. The scheduled rotation takes effect silently at a timestamp. A trader who commits five seconds before the expiry has a live 60-second execution window, but their order is invalid before any keeper can fill it, and whoever clears it is paid for doing so.

Asset: `packages/perps/src/OrderLifecycleBook.sol`, `packages/perps/src/CfdTreasury.sol`, `packages/perps/src/OrderRecoverySidecar.sol` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Relationship to published known issues

- Program known-issues entry "Cecuro MED-13: a governance oracle rotation retargets armed stop-losses" concerns protections under an oracle change, not order config-hash invalidation. Entries "SECURITY.md: Known Limitations" and "Router and keeper limitations" describe terminal `ConfigMismatch` only for finalized config changes; none describes a hash input that changes at a timestamp with no transaction, nor the bounty on a `ConfigMismatch` clear going to the clearer.
- `SECURITY.md:765-778` documents that the treasury is part of the config hash and that "finalized policy drift produces terminal `ConfigMismatch` for config-pinned external orders". It says nothing about a drift that is not finalized by a transaction.
- Cecuro MED-3 (treasury rotation) was fixed before #130 replaced the mechanism with `CfdTreasury`'s lock and auto-rotation; the audit did not see the timestamp-driven switch.
- Not in the 2026-10-15 scan or the pre-audit report.

## Root cause

The hash binds a value that can change without a transaction (`OrderLifecycleBook.sol:193-202`):

```solidity
bytes memory engineConfig = abi.encode(
    engine.planner(),
    engine.settlementSidecar(),
    engine.terminalNavBook(),
    engineAdmin,
    IOrderLifecycleAdminConfigView(engineAdmin).activeConfigVersion(),
    engine.protocolTreasury(),            // resolves by timestamp through CfdTreasury._activeTreasury()
    IOrderLifecyclePoolConfigView(HOUSE_POOL).markStalenessLimit()
);
```

Cleanup is permissionless on mismatch (`OrderRecoverySidecar.sol:92-95`):

```solidity
bool configInvalid = pending.bounds.expectedConfigHash != bytes32(0)
    && pending.bounds.expectedConfigHash != observedConfigHash;
if (!expired && !configInvalid) continue;
```

The bounty disposition on that terminal path pays the executor (`OrderLifecycleBook.sol:717-721`):

```solidity
} else {
    paid = bounty;
    recipient = receipt.executor;
    disposition = OrderV3Types.BountyDisposition.Paid;
}
```

## Impact

- Every pending order with a pinned config hash is cancelled at the rotation timestamp. In the PoC, both Alice's and Bob's opens are cleared; Bob's was not at the FIFO head.
- Each victim loses their reserved open bounty (0.20 USDG at the deployed cap) to the clearer; the PoC shows a 0.40 USDG transfer to the clearer for two orders. Margin is refunded. The intent never executes, so each victim also loses whatever the fill was worth to them.
- No event marks the moment of invalidation, so wallets and keepers cannot warn users to wait or re-pin.

## Recommended fix

1. Remove the treasury from `currentExecutionConfigHash`. It decides where fees are credited, not how an order executes, and it is the only hashed input that can change without a transaction.
2. If it must remain, refund the bounty to the account on `ConfigMismatch` instead of paying the clearer, and route the immediate `setProtocolTreasury` through the same 72-hour timelock as other config.
3. Emit an event when a scheduled rotation activates (a cheap `_syncTreasury` call in any owner path does not help; the hash changes at the timestamp regardless).

## Link to Proof of Concept

https://gist.github.com/aussinfosec/e323f4eeaa35586152186e2c7fad1c49

## Proof of Concept

Gist (secret): https://gist.github.com/aussinfosec/e323f4eeaa35586152186e2c7fad1c49

One Foundry test on a clean checkout of `a82d4fef`, on the repository's `BasePerpTest`.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc && cp /path/to/gist/PoC_R2_TreasuryRotationCancelsOrders.t.sol packages/perps/test/perps/poc/
forge test --root packages/perps --match-path test/perps/poc/PoC_R2_TreasuryRotationCancelsOrders.t.sol -vv
```

Stages, all passing (`test_PoC_TreasuryAutoRotationMassCancelsPinnedOrders`):

1. The owner calls `setProtocolTreasury(A, 1 days)` then `proposeProtocolTreasury(B, 0)`, scheduling B to activate when the one-day lock expires.
2. Alice and Bob each commit a pinned V3 open five seconds before expiry. Assert each order's execution deadline is after the activation timestamp.
3. **Counterfactual** (snapshot): with no rotation, Alice's order executes normally (`LifecycleStatus.Executed`).
4. **Rotation** (revert snapshot, warp to expiry): assert `engine.protocolTreasury() == B` and `currentExecutionConfigHash() != pinned` with no transaction sent.
5. Mallory calls `clearInvalidOrders([alice, bob])`: assert `cleared == 2`, both accounts have zero pending orders, Alice's position size is 0, and Mallory's settlement gain equals the sum of both victims' bounties.

Observed output:

```text
[PASS] test_PoC_TreasuryAutoRotationMassCancelsPinnedOrders()
  execution window seconds 60
  alice bounty 200000
  bob bounty 200000
  alice settlement loss 200000
  bob settlement loss 200000
  mallory (clearer) gain 400000
```
