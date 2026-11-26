# \#99080 [SC-High] Permissionless protection retries keep a Latched protection's close pending: the owner's amend and its other single-call exits revert, and an under-margined position whose owner tries only those is held until liquidation

**Submitted on Nov 11th 2026 at 19:10:38 UTC by @mhkarazeybek for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99080
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro
When a take-profit or stop-loss close fails on the owner's signed close limits, the protection becomes Latched. While it is Latched the owner cannot close, reduce, cancel or replace; the remedy is to amend the limits and retry, and the amend is allowed only while no order of the account is pending. `retryPositionProtectionClose` has no caller check and costs only gas. Any caller that clears the expired attempt and retries in one transaction, once per expiry window, keeps an attempt pending with the owner's unchanged limits. The owner's amend, sent as a single call, then reverts every time, so an owner that neither batches calls nor executes the pending attempt itself and wins the ordering that follows cannot leave the position or recover its margin while the caller keeps paying gas. If the position's margin is below the Friday frozen-market requirement, it is carried to the step-up and liquidated, and the same caller can execute the liquidation and receive the keeper share.

The limits need not be a mistake: in the PoC the owner's only finite limit is a gross-account-debit ceiling 20% above the close preview at the stop price, and the stop executes 5 bps past its price. The result has clear limits, stated below: the web app signs unlimited ceilings, so its users are not exposed; an account that can batch calls escapes in one transaction; a plain EOA can get out with two transactions unless a retry from the caller lands between them; and for an owner who does nothing the outcome is the same with or without the caller.

## Vulnerability Details
Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

```solidity
// PositionProtectionBook.sol:315-328, amendLatchedPositionProtectionCloseLimits (owner only)
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

```solidity
// PositionProtectionBook.sol:382-389, retryPositionProtectionClose (any caller)
StoredProtection storage protection = _activeProtection(protectionId);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
```

A retry commits a new close attempt with the stored limits; while it is pending the protection is `Triggered`, not Latched, so the amend and the owner's own retry revert `OrderRouter__ProtectionNotLatched`. When the attempt is executed and fails, or is cleared after it expires (`clearInvalidOrders`, any caller), the protection is Latched again, and the transaction that executes or clears it can also retry. The interface describes the amend as available "after a failed attempt relatches, before a fresh retry. … A live child cannot be changed" (`interfaces/IPositionProtectionActions.sol:32-33`); because anyone may send the fresh retry, anyone decides whether that moment exists for the owner.

The limits and the amend were added by the fix for Cecuro MED-9 (PR #124); permissionless retry is older (PR #78) and is in the commit Cecuro audited. The team's comment on MED-9 names the remedy: "Tight limits may prevent execution until conditions improve or the owner amends them", and PR #124 says "the owner may amend limits after relatch". In the PoC the owner can amend after none of the 383 relatches that follow the caller's first retry. Its one window is the initial one, between the first failed attempt and that first retry (1 of 768 transaction boundaries); control A's amend lands there.

How the protection latches. The close is checked against four signed ceilings. A gross-account-debit ceiling includes the price loss, so a ceiling taken from the preview at the stop price with headroom `k` is exceeded when the close executes about `d × (k − 1)` bps past a stop `d` bps away: for `k` = 1.2 and `d` = 10 the rule gives 2 bps, and the measured threshold was 2 bps at 1M and 3 bps at 100k (both sides, both sizes, stops 10 and 100 bps away, four headrooms; separate run, not in the gist).

## Impact Details
From the PoC (USDG; a plain-EOA owner, LONG with 1% margin, stop 10 bps away; three branches from one snapshot taken at the latch, 6.5 hours before the Friday step-up):

| | Control A: owner amends and closes | Control B: nobody acts | Attack: retry loop to the step-up |
|---|---|---|---|
| Owner's amend | succeeds | not attempted | reverts in 383 of 383 windows |
| Owner's result, 1M position | −133.97 | −2,530.49 (liquidated) | −2,530.49 (liquidated) |
| Owner's result, 100k position | −173.85 | −251.76 | −251.76 |
| Liquidation keeper share | — | honest keeper +500.00 / +50.00 | the caller's contract +500.00 / +50.00 |

During the attack the owner tries eight single-call exits in every window and all revert (3,064 refusals). Six of them — full close, partial reduce, increase, cancel, cancel-and-close, replace — revert in any Latched state, with or without the caller; the caller removes the remaining two, the amend and the owner's own retry, which is the documented way out. The position's margin (10,000 at 1M) stays in the position, since none of these exits releases it; in the separate 10%-margin run a withdrawal of the whole deposit reverted `MarginClearinghouse__InsufficientBalance` at each checkpoint (not in the gist). The owner can still add margin (asserted in the PoC) and withdraw free balance (separate runs; not in the gist).

Against control A the owner loses 2,396.51 (1M) or 77.91 (100k) more. At 1M, 500.00 goes to the caller as keeper share and 1,896.52 to the pool; at 100k, 50.00 goes to the caller and 50.13 to the pool, and the protocol treasury receives 22.22 less than in control A. The caller's cost is gas: 384 loop transactions of about 0.94M gas and one liquidation, about 11 USD at 0.01 gwei and 109 USD at 0.1 gwei as measured inside the test (12 and 121 USD with each transaction measured in isolation; not in the gist), assuming ETH at 3,000 USD. At 0.1 gwei that is more than the 50.00 keeper share of the 100k position. The keeper share goes to whichever liquidation lands first at the step-up: the caller's, another keeper's, or the owner's own (the owner's result is then −2,030.49 at 1M; separate run, not in the gist).

Limits of the result, measured:

- Against control B there is no difference for the owner: an owner who would not have acted loses the same amount without the caller. The loss is attributable only to an owner who tries to amend.
- An account that can batch calls (a contract account or an EIP-7702 delegation) escapes in one transaction that executes the pending attempt, amends and retries (owner's result −146.57 at 1M; asserted). Arbitrum One has accepted EIP-7702 since the ArbOS 40 upgrade: Arbitrum's ArbOS 40 release notes list EIP-7702, and the governance proposal and the Arbitrum Foundation's announcement date the activation to June 2025 (public pages, read by us as search-result summaries; not checked on-chain). So an EOA can take this exit by delegating to a batching contract, which is how the owner in the PoC takes it.
- A plain EOA also has a two-transaction exit. It executes the pending attempt itself (`executeOrder` is open to any caller; the attempt fails on the ceiling and the protection is Latched with nothing pending) and amends in its next transaction. This closed the position mid-window and at expiry (owner's result −173.88 at 100k, against −173.85 in control A), and failed when a retry from the caller landed between the two transactions (separate runs; not in the gist). The caller in the PoC sends one transaction per 61 seconds and does not react to the owner; to hold an owner that does this it must win that ordering every time, which a local test cannot decide.
- Adding margin up to the step-up requirement (24,751.52 at 1M, 2,313.28 at 100k) plus the test's pad (10.00 and 1.00, about 0.1 bp of notional) prevents the liquidation, not the lock (asserted). This holds at the PoC's confidence. In separate runs on a copy with the release values (not in the gist), at each of the six basket confidences tested (0.40, 0.5, 1, 2, 5 and 9.9 bps) that amount no longer prevents the liquidation and the smallest top-up that does is larger: at 1M from 24,765.29 at 0.40 bps to 25,245.44 at 9.9 bps, at 100k from 2,314.52 to 2,362.61.
- Oracle confidence. In those runs, at basket confidences from 0.4 to 9.9 bps (the oracle refuses more than 10), the latch, the refusal of the owner's amend in every window and the equality of the attack with control B all hold, the stop executes 5.0 to 7.4 bps past its price, and the loss against control A stays about 2,397 at 1M (2,397.79 to 2,396.60) and 77.9 at 100k. The owner's own result worsens with confidence in every branch (at 1M, control A from −136.52 at 0.40 bps to −611.68 at 9.9 bps and the attack from −2,534.30 to −3,008.28).
- The PoC's loop is the case in which nobody else executes the pending attempts. If a keeper executes each one (it fails on the ceiling), the protection is Latched with nothing pending across a block in every cycle (600 of 600), and the owner's amend inside that block succeeds; the caller must then win that race each time, and its gas cost rises to about 87 USD per hour at 0.01 gwei (separate runs; not in the gist).
- If the price returns inside the ceilings, the pending attempt executes and the lock ends.
- Without a liquidation the same loop is a freeze only: with 10% margin and a Monday latch it ran 72 hours with all eight exits refused at each of 8 checkpoints between 1 and 72 hours, at about 1.85 USD per hour of gas at 0.01 gwei when nobody else executes the attempts (separate run; not in the gist).
- The project's web app signs unlimited ceilings, so its protections do not latch on ceilings (source of the app repository linked from the program page). The exposed owners are those who sign finite ceilings outside the app. The SDK's accounts are smart accounts that can batch, so they have the one-transaction exit, but not through the project's sponsored path, whose policy does not approve the amend selector (same source). The account in the PoC is a plain EOA that calls the contracts directly.

Closest exclusions.

- Known issues. Cecuro MED-9 (acknowledged; addressed by PR #124) is the related finding. The program keeps a report on a listed finding valid "only if it shows that the fix … is incomplete or introduces a new bug": we do not claim the fix is incomplete, since its ceilings are enforced as written. We claim it introduced this: the remedy it names for a Latched protection can be withheld by any caller, and that lock-out exists only since the fix. `SECURITY.md` accepts unpaid retries and asks operators to alert on a "prolonged `Latched` state without a live child"; under the loop an attempt is always pending after the caller's first retry.
- Trust. The program states that "Keepers are not trusted: anyone can execute orders, liquidations, price updates and protections, so harm caused by any caller acting as a keeper is in scope."
- Severity. The separate 72-hour run exceeds 24 hours; the PoC in the gist runs 6.5 hours. We file the 1-hour row for that reason, and because a batching account escapes, a plain EOA has the two-transaction exit, and with a keeper executing the attempts the lock is a race. We leave the row to the program.

## Recommendation
Let the owner amend the limits while a retry attempt is pending (cancelling that zero-bounty attempt), or give the owner a guaranteed window after each relatch (for example, a short period in which only the owner may retry). A cool-down on third-party retries of a Latched protection would also remove the loop. We have not built or tested a patch.

## References
`PositionProtectionBook.sol:315-328`, `:382-406`; `interfaces/IPositionProtectionActions.sol:32-33`; Cecuro MED-9 and its fix (PR #124); PR #78; `SECURITY.md` Known Limitations (failed protection-attempt cleanup).

## Link to Proof of Concept

https://gist.github.com/mhkarazeybek/fe0021cec78bfd0861955cb4dc64aa62

## Proof of Concept

## Proof of Concept
One Foundry test for the repository's suite at the in-scope commit (gist below): `H3Final.t.sol`, with its fixture files (`H3FinalBase.sol`, `S1Base.sol`, `R1H3DecideBase.sol`, `H3LatchedFadBase.sol`, `G3FreezeStraddleBase.sol`, `pythmt/MultiTimePyth.sol`, `pythmt/PythMultiTimeBase.sol`). It passes.

```
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc/pythmt
cp H3Final.t.sol H3FinalBase.sol S1Base.sol R1H3DecideBase.sol H3LatchedFadBase.sol G3FreezeStraddleBase.sol packages/perps/test/poc/
cp MultiTimePyth.sol PythMultiTimeBase.sol packages/perps/test/poc/pythmt/
forge test --root packages/perps --match-path 'test/poc/H3Final.t.sol' --match-test test_H3Final_PoC --gas-limit 1000000000000 -vv
```

What the test uses:

- No in-scope contract is modified. No storage writes and no mocked calls. One impersonation of a protocol contract, in setup only: the repository's `BasePerpTest` funds the two tranches with a helper that pranks the Router to install a mark price (twice); the fixture then refreshes the mark through the Router's public entrypoint before any position is opened.
- After setup, every state change is an external call to a public entrypoint from its caller's own address. The third party is a contract with no position and no role; in each round it calls `clearInvalidOrders` for the expired attempt and `retryPositionProtectionClose` in one transaction, and at the step-up `executeLiquidation`.
- The deployment is the repository's `BasePerpTest` with the release risk parameters and market calendar. These values are not the release values: the Router's minimum open notional is 100 USDG (release 1,000; positions here are 100,000 and 1,000,000), the Senior exposure cap and share limit are the test base's (release 40,000,000 USDG and 8,000 bps), the tranche seeds are 1,000 USDG (release 1). In a separate run (not in the gist) with the release seeds (1 USDG per tranche), Senior limits, Router minimum and multiplier installed and asserted, the test passes unchanged and only the seeds move the USDG figures, by at most 1.29: the loss against control A goes 2,396.51 → 2,397.79 at 1M and 77.91 → 77.92 at 100k, and the step-up requirement 24,751.52 → 24,752.80 at 1M. The pool sizes remain the fixture's. The Router is the repository's test harness `LegacyOrderRouterHarness`, a subclass of `OrderRouter`; the oracle is a test subclass of `PletherOracle` that only sets the adverse-confidence multiplier to the release value 2,500 in its constructor, as the release script's oracle does, and it prices a six-feed basket (four feeds inverted) whose basket confidence, from the per-feed confidences the Pyth double publishes, is about 0.33 bps, so closes and liquidations in the PoC already carry an adverse shift of about 0.08 bps; the pool holds 0.8M Junior and 2.0M Senior USDG (no release value); Pyth is a test double that serves per-feed publish times; USDG is the repository's mock.
- The index path is the fixture's: +15 bps at the latch, on the Friday 6.5 hours before the step-up, then flat.
- `--gas-limit` only raises the test call's own gas (384 rounds in one test).

Output, excerpt (other lines are omitted):

```
[PASS] test_H3Final_PoC() (gas: 842905030)
  [1M latch] first child failed ConstraintViolation(constraint=3 GrossAccountDebit): actual 1517.445130 > limit 1209.856156 (overrun 307.588974); exec price 5.0 bps past the stop; protection Latched, no child pending
  [1M]   amend limits         amendLatchedPositionProtectionCloseLimits | ok | OrderRouter__ProtectionNotLatched(0x5b54e624)
  [1M] [CONTROL-A] owner amends (unlimited) + retries + keeper fills 22799 s before F: victim -133.973295 (wallet vs funding -2318.812947)
  [1M] [CONTROL-B] nobody acts, honest keeper liquidates at F: victim -2530.487440 (wallet -4715.327092), keeper 499.995780
  [1M] [ATTACK] 384 rounds (1 tx / 61 s, 942384 gas/round warm); owner exits refused 3064 times in 383 windows; owner amend would succeed at 1 of 768 tx boundaries (the initial window before the 1st round)
  [1M] [DIFF] ATTACK - CONTROL-A = -2396.514145 to the victim = third party 499.995780 + pool 1896.518365 + treasury 0.000000; ATTACK vs CONTROL-B victim difference 0.000000
  [1M] [LIMIT 1: account type] same owner as a 7702 batch (execute child + amend + retry, 1 tx) 3 min before F: closed 217 s before F, victim -146.568365 (CONTROL-A -133.973295)
```

In the amend line the two results are control A (at the latch, nothing pending) and the attack (mid-window, an attempt pending). The 100k position prints the same lines with its own figures.
