# \#99388 [SC-Insight] CEI deviation in `CfdEngine.settleTraderClaim`: claim ledger is decremented after the external payout

**Submitted on Nov 13th 2026 at 13:19:45 UTC by @Bozwkd for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99388
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

## Summary

`CfdEngine.settleTraderClaim` moves value out of the HousePool and into the clearinghouse **before** it decrements the trader-claim ledger (`traderClaimBalanceAssets[account]` and `totalTraderClaimBalanceAssets`). This is a Checks-Effects-Interactions (CEI) ordering deviation: the state that authorizes the payout is updated *after* the external calls that execute it.

The practical consequence is a design smell worth fixing on its own merits: the invariant **"a trader claim is settled exactly once"** is not enforced by the function's own state ordering. It currently rests on two external facts — the `nonReentrant` modifier and the documented standard-ERC-20 settlement token with no transfer callbacks — instead of on local checks-effects-interactions discipline. The accompanying PoC makes this concrete: during the payout, the claim ledger still shows the full claim and the solvency gate still passes, and a reentrant settlement attempt is stopped specifically by the reentrancy guard.

---

## Root Cause

In `CfdEngine.settleTraderClaim` the payout happens, then the claim ledger is reduced (`CfdEngine.sol:572-586`):

```solidity
uint256 amount = traderClaimBalanceAssets[account];              // CHECK
if (amount == 0) revert CfdEngine__NoTraderClaim();
...
uint256 claimAmountAssets =
    _settleTraderClaimBalance(amount, account, pos.lots > 0);     // INTERACTION: pool.payOut + clearinghouse credit
...
traderClaimBalanceAssets[account] -= claimAmountAssets;          // EFFECT (after interaction)
totalTraderClaimBalanceAssets     -= claimAmountAssets;          // EFFECT (after interaction)
```

`_settleTraderClaimBalance` → `_payTraderClaim` executes the external value movement first (`CfdEngine.sol:219-244`):

```solidity
// CfdEngine.sol (_settleTraderClaimBalance)
if (amount == 0 || pool.totalAssets() < totalTraderClaimBalanceAssets) {
    revert CfdEngine__InsufficientPoolLiquidity();
}
claimAmountAssets = amount;
_payTraderClaim(account, claimAmountAssets, positionRemainsOpen);

// CfdEngine.sol (_payTraderClaim)
pool.payOut(address(clearinghouse), amountAssets);               // external
if (positionRemainsOpen) clearinghouse.creditPnlPledge(account, amountAssets);
else                     clearinghouse.settleAssets(account, int256(amountAssets));
```

The read of `traderClaimBalanceAssets[account]` (the amount authorized) and its decrement straddle the external calls. Correct CEI would decrement the per-account and aggregate claim ledgers **before** calling `_payTraderClaim`, so that any code executing during the payout observes the claim as already consumed.

---

## Proof of Concept

**File:** `test/perps/accounting/claims/SettleTraderClaimCeiInsight.t.sol`

The PoC drives the **unmodified production `CfdEngine`** through `settleTraderClaim` with a `100e6` claim. Only the three dependencies are lightweight stand-ins:

- `RecordingPool` implements the pool surface used by the flow and, inside `payOut` (i.e. *during* the interaction), records the engine's live claim ledger and attempts a reentrant `settleTraderClaim`.
- `MockClearinghouse` / `MockNavBook` satisfy the engine's constructor and terminal-curve hooks.

### What it demonstrates

**Test 1 — `test_InteractionPrecedesEffect`.** Observations recorded *inside* `pool.payOut`, before the engine's decrement:

| Observation during the payout | Value | Meaning |
|---|---|---|
| `engine.traderClaimBalanceAssets(TRADER)` | `100e6` | Per-account claim **not yet decremented** while value moves |
| `engine.totalTraderClaimBalanceAssets()` | `100e6` | Aggregate claims **not yet decremented** |
| `pool.totalAssets() >= totalTraderClaimBalanceAssets` | `true` | The solvency gate **still passes** mid-interaction |

After the call returns, the effects have landed exactly once (`traderClaimBalanceAssets == 0`, aggregate `== 0`, clearinghouse holds `100e6`). The bad practice is therefore not theoretical: there is a live window inside the settlement where the ledger says "claim still payable" while the tokens have already left the pool.

**Test 2 — `test_OnlyReentrancyGuardBlocksSecondPayout`.** The reentrant `settleTraderClaim` attempted during the payout fails, and the revert reason is exactly `ReentrancyGuardReentrantCall.selector`. The second payout is stopped by the **guard**, not by the claim ledger or the solvency gate — the two in-repo checks that *should* own the "settled exactly once" invariant play no role in stopping it.

Run with:

```bash
forge test --match-path test/perps/accounting/claims/SettleTraderClaimCeiInsight.t.sol -vvv
```

Both tests pass, which is the point: the PoC documents the ordering itself rather than relying on a hypothetical trace.

---

## Security Considerations

This is an **Insight**, not an exploit report. The window the PoC exposes is contained today by the `nonReentrant` modifier and by the documented settlement-token assumption (standard ERC-20, no transfer callbacks; `HousePool.payOut` performs only `accountedAssets -= amount` + `safeTransfer`, and the clearinghouse credit is internal bookkeeping).

The reason to fix it is robustness of the core invariant:

- The "settled exactly once" property and the `totalTraderClaimBalanceAssets == Σ traderClaimBalanceAssets` conservation identity currently depend on the reentrancy guard and on settlement-token behavior, not on local state ordering.
- The code already treats claims as senior liabilities with an aggregate solvency gate; letting that gate's own ledger lag the payout is exactly the kind of latent dependency that turns into a double-claim if the surrounding assumptions ever change (a future settlement asset with transfer hooks, or a refactor of the guard coverage).
- CEI is the project's own idiom elsewhere (e.g. `_consumeTraderClaim` debits the ledger during planning, before cash moves). `settleTraderClaim` is the outlier.

---

## Recommendation

Reorder `settleTraderClaim` to follow checks-effects-interactions: decrement the claim ledgers **before** the external payout.

```diff
-    uint256 claimAmountAssets =
-        _settleTraderClaimBalance(amount, account, pos.lots > 0);
     ...
-    traderClaimBalanceAssets[account] -= claimAmountAssets;
-    totalTraderClaimBalanceAssets     -= claimAmountAssets;
+    uint256 claimAmountAssets = amount;
+    traderClaimBalanceAssets[account] -= claimAmountAssets;   // EFFECT before INTERACTION
+    totalTraderClaimBalanceAssets     -= claimAmountAssets;
+    _settleTraderClaimBalance(claimAmountAssets, account, pos.lots > 0);  // INTERACTION last
```

Two details keep the change semantics-preserving:

1. **Solvency gate input.** `_settleTraderClaimBalance` currently compares `pool.totalAssets()` against the *pre-decrement* `totalTraderClaimBalanceAssets`. If the decrement moves first, the gate must be evaluated against the original aggregate (pass it in, or hoist the check above the decrement) so its meaning is unchanged: pool cash covers all outstanding claims **including the one being settled**. Since payout and decrement reduce assets and claims by the same amount, evaluating against the pre-decrement aggregate keeps every currently-passing settlement passing.
2. **Revert atomicity.** Any failure in `pool.payOut` or the clearinghouse credit reverts the whole call, so the earlier decrement rolls back with it; the PoC's `test_TraderClaim_DownstreamTransferFailureRollsBackAccounting` pattern in `TraderClaimsMatrix.t.sol` covers this behavior and should be re-run after the change.

## Proof of Concept

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {StdStorage, Test, stdStorage} from "forge-std/Test.sol";
import {ReentrancyGuardTransient} from "@openzeppelin/contracts/utils/ReentrancyGuardTransient.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {MockToken} from "../../../mocks/MockToken.sol";

/// @notice Pool stand-in that records the engine's claim ledger at the exact moment value moves,
///         and attempts a reentrant `settleTraderClaim` during the payout.
contract RecordingPool {

    MockToken public immutable settlementToken;
    CfdEngine public immutable ENGINE;

    uint256 public accountedAssets;
    address public watchedAccount;

    uint256 public midAccountClaim;
    uint256 public midTotalClaims;
    uint256 public midTotalAssets;
    bool public midGateOpen;
    bool public reentrantSucceeded;
    bytes public reentrantReason;

    constructor(
        MockToken token,
        CfdEngine engine
    ) {
        settlementToken = token;
        ENGINE = engine;
    }

    function setAccountedAssets(
        uint256 amount
    ) external {
        accountedAssets = amount;
    }

    function setWatchedAccount(
        address account
    ) external {
        watchedAccount = account;
    }

    function totalAssets() external view returns (uint256) {
        uint256 raw = settlementToken.balanceOf(address(this));
        return raw < accountedAssets ? raw : accountedAssets;
    }

    function payOut(
        address recipient,
        uint256 amount
    ) external {
        require(msg.sender == address(ENGINE), "only engine");

        // State observed DURING the interaction, before the engine's ledger decrement.
        midAccountClaim = ENGINE.traderClaimBalanceAssets(watchedAccount);
        midTotalClaims = ENGINE.totalTraderClaimBalanceAssets();
        midTotalAssets = this.totalAssets();
        midGateOpen = midTotalAssets >= midTotalClaims;

        try ENGINE.settleTraderClaim(watchedAccount) {
            reentrantSucceeded = true;
        } catch (bytes memory reason) {
            reentrantReason = reason;
        }

        accountedAssets -= amount;
        settlementToken.transfer(recipient, amount);
    }

}

contract MockClearinghouse {

    MockToken public immutable settlementAsset;

    constructor(
        MockToken token
    ) {
        settlementAsset = token;
    }

    function settleAssets(
        address,
        int256
    ) external {}

    function pnlPledgeAssets(
        address
    ) external pure returns (uint256) {
        return 0;
    }

}

contract MockNavBook {

    function validateEngineBinding() external pure returns (bool) {
        return true;
    }

    function authenticateEngineState(
        address
    ) external pure returns (bytes32) {
        return bytes32(0);
    }

    function syncFromEngine(
        address,
        bytes32
    ) external pure returns (bytes32, uint64) {
        return (bytes32(0), 0);
    }

}

/// @notice PoC for the Insight: `CfdEngine.settleTraderClaim` performs the external payout
///         BEFORE decrementing the claim ledger (Checks-Effects-Interactions deviation).
///         The production engine is used unmodified; only its three dependencies are lightweight stand-ins.
contract SettleTraderClaimCeiInsightTest is Test {

    using stdStorage for StdStorage;

    uint256 internal constant CLAIM = 100e6;
    address internal constant TRADER = address(0xCA11);

    MockToken internal token;
    CfdEngine internal engine;
    RecordingPool internal pool;
    MockClearinghouse internal clearinghouse;

    function setUp() public {
        token = new MockToken("Mock USDG", "USDG", 6);
        clearinghouse = new MockClearinghouse(token);

        CfdTypes.RiskParams memory params = CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
        engine = new CfdEngine(address(token), address(clearinghouse), 2e8, params, 50);
        pool = new RecordingPool(token, engine);
        engine.setPool(address(pool));
        engine.setTerminalNavBook(address(new MockNavBook()));

        pool.setAccountedAssets(CLAIM);
        token.mint(address(pool), CLAIM);
        pool.setWatchedAccount(TRADER);

        stdstore.target(address(engine)).sig("traderClaimBalanceAssets(address)").with_key(TRADER).checked_write(CLAIM);
        stdstore.target(address(engine)).sig("totalTraderClaimBalanceAssets()").checked_write(CLAIM);
    }

    function test_InteractionPrecedesEffect() public {
        vm.prank(TRADER);
        engine.settleTraderClaim(TRADER);

        // At the moment the pool paid out, the claim ledger still held the full amount:
        // the effect (decrement) had not run yet.
        assertEq(pool.midAccountClaim(), CLAIM, "per-account claim not yet decremented during payout");
        assertEq(pool.midTotalClaims(), CLAIM, "aggregate claims not yet decremented during payout");
        assertEq(pool.midTotalAssets(), CLAIM, "pool assets still pre-payout");

        // The solvency gate itself still passes mid-interaction: it cannot prevent a second
        // payout on its own while the ledger is in this state.
        assertTrue(pool.midGateOpen(), "solvency gate still satisfied mid-interaction");

        // Effects land only after the interaction, exactly once.
        assertEq(engine.traderClaimBalanceAssets(TRADER), 0, "claim cleared after the call");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "aggregate cleared after the call");
        assertEq(token.balanceOf(address(clearinghouse)), CLAIM, "payout reached the clearinghouse");
        assertEq(token.balanceOf(address(pool)), 0, "pool paid out its assets");
    }

    function test_OnlyReentrancyGuardBlocksSecondPayout() public {
        vm.prank(TRADER);
        engine.settleTraderClaim(TRADER);

        // The reentrant settlement attempted during the payout failed, and it failed specifically
        // on the reentrancy guard — not on any claim-ledger or solvency check.
        assertFalse(pool.reentrantSucceeded(), "reentrant settle must not succeed");
        assertEq(
            bytes4(pool.reentrantReason()),
            ReentrancyGuardTransient.ReentrancyGuardReentrantCall.selector,
            "revert came from the reentrancy guard"
        );

        // Exactly one payout happened: the clearinghouse holds one claim amount, not two.
        assertEq(token.balanceOf(address(clearinghouse)), CLAIM, "single payout");
        assertEq(engine.traderClaimBalanceAssets(TRADER), 0, "claim consumed once");
    }

}
```
