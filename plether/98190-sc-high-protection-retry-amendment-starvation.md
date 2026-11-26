# \#98190 [SC-High] Protection Retry Amendment Starvation

**Submitted on Nov 5th 2026 at 20:12:20 UTC by @hackio9238 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98190
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief / Intro

`PositionProtectionBook` lets any caller retry a failed protection immediately, but only the protected account may amend close limits and only while the protection has no live child. A keeper can atomically expire and retry each child before an EOA owner reaches that amendment state. Repeating the call keeps the funded position locked for more than 24 hours.

## Vulnerability Details

A protected close can become invalid after an ordinary price movement. The owner may set a finite `maxExecutionNotionalAssets` while the protection is armed. The stored ceiling applies to every triggered or retried close.

`PositionProtectionBook.handleFailedProtectionAttempt` changes a matching failed child from `Triggered` to `Latched`. The `Latched` state has no live child and is the only state accepted by `amendLatchedPositionProtectionCloseLimits`.

Amendment authenticates `msg.sender` as the protected account. An external helper cannot amend for an EOA. `retryPositionProtectionClose` has no ownership check. It accepts any caller and creates a new child. `_recordCloseAttempt` changes the protection from `Latched` back to `Triggered` in that caller's transaction.

The transaction below is therefore available to an arbitrary keeper after each 60-second child deadline:

```solidity
router.expireOrder(attemptId);
protectionBook.retryPositionProtectionClose(protectionId);
```

The first call exposes the owner-only recovery state. The second call consumes it before another transaction can amend the limits. A new child again blocks the account.

The owner cannot use another ordinary exit. `OrderRouter` rejects discretionary orders while an active protection exists. Protection cancellation and `cancelPositionProtectionAndCommitClose` accept `Armed` protections, not `Triggered` or `Latched` protections.

The PoC opens an EOA-owned 10,000-token LONG with 2,000 USDG of margin. Its protection has a 10,000 USDG execution-notional ceiling. This ceiling is valid at the armed price. A genuine stop-loss move increases the close notional to 11,000 USDG, and a correct post-commit oracle update terminalizes the child with `ConstraintViolation`.

The attacker performs one failure-and-retry transaction, then 1,417 bounded expiry-and-retry transactions at successive deadlines. The EOA submits an amendment immediately before and immediately after every recurring transaction. All of those active-owner attempts revert because the protection is `Triggered` at both external transaction boundaries. The attack finishes after more than 24 hours with 1,995.500999 USDG of position collateral unchanged, one live child, and every owner exit rejected. The recurring transactions use no oracle update and receive no bounty.

The matched control starts from the same position, protection, limit failure, and price. Without the atomic retry, the same EOA amends the limit in `Latched`, retries the protection, closes the position, and releases both collateral buckets.

### Technical Fix

- Allow the protected account to version the limits for the next retry while a child remains `Triggered`.
- Apply those versioned limits only to the next child, so the current child keeps its immutable policy.
- Alternatively, reserve a recovery interval after relatching in which only the protected account may amend or retry.

## Impact Details

A targeted denial-of-service attack could temporarily freeze an EOA trader's funded position collateral for more than 24 hours. The attacker may keep the owner from amending an obsolete protection limit even though that amendment is the documented recovery path. The victim would have to win the recurring transaction-ordering race or wait for the attacker to stop.

## References

- Frozen target commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- `packages/perps/src/PositionProtectionBook.sol:315-328,382-406,564-577,637-668`
- `packages/perps/src/OrderRouter.sol:191-196,287-329`
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:302-315`
- `packages/perps/SECURITY.md:530-562,1345-1360`
- `packages/perps/ACCOUNTING_SPEC.md:954-986`

## Proof of Concept

# Proof of Concept

## Archive

Archive PoC link: https://drive.google.com/file/d/1gNrNK2sUrdfZhyL9pZShKjBw8iK2XWP1/view?usp=sharing

## Safety

This PoC runs only on a local Foundry EVM. It does not contact a chain, RPC service, application, or Pyth endpoint.

## Actors and Preconditions

- The victim is an EOA with an active funded position and a finite protection limit that was valid when armed.
- A normal trigger-price movement makes the stored close notional ceiling too low.
- The attacker is an arbitrary permissionless keeper. It supplies no token principal and receives no retry bounty.
- The price remains beyond the victim's old ceiling while the lock is maintained.

### PoC Files

- `test/ProtectionRetryAmendmentStarvation.t.sol`

## Environment

- Repository: `https://github.com/immunefi-team/audit-comp-plether.git`
- Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- Foundry: `v1.5.1`
- Solidity: `0.8.35`

The test deploys the target's `BasePerpTest` stack. MockUSDG and MockPyth are the only substitutes. The program permits both mocks. The test overrides the fixture's risk parameters with the disclosed v2.0.0 release values and asserts the release-default 60-second execution lifetime through the number of attack transactions.

## Reproduction

Clone the exact target and initialize its submodules:

```bash
git clone --recursive https://github.com/immunefi-team/audit-comp-plether.git plether-poc
cd plether-poc
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

Extract `plether-protection-retry-amendment-starvation-poc.zip` at `packages/perps`, then run:

```bash
forge test --root packages/perps \
  --match-path test/ProtectionRetryAmendmentStarvation.t.sol \
  --match-test 'test_(AttackerAtomicallyExtendsFundedEoaLockPast24Hours|ControlWithoutAtomicRetryLetsSameEoaAmendAndClose)' \
  --gas-limit 10000000000000 -vv --color never
```

The high harness gas limit is needed because one test function simulates 1,418 separate attack transactions. It is not the cost of one production transaction.

## Expected Result

Both tests pass. The attack test logs:

```text
attacker transactions over 24 hours: 1418
victim position collateral locked: 1995500999
```

The attack test finishes after more than 24 hours with the protection `Triggered`, one current child, the full original position open, and its victim-funded collateral unchanged. The victim tries to amend immediately before and immediately after every recurring atomic attack cycle. Every amendment reverts. Cancellation, an atomic manual close, and an ordinary close also revert.

The control exposes `Latched` without an atomic attacker retry. The same EOA amends the close ceiling, and the next permissionless retry executes and releases the position's PnL pledge and liquidation reserve.

## What The PoC Establishes

- A finite close ceiling can be valid when protection is armed and fail after a genuine trigger-price movement.
- Failure or expiry exposes `Latched`, which is the only owner-amendable state.
- One arbitrary caller can consume `Latched` inside the same terminalization transaction.
- Every recurring attack call has bounded work, uses no oracle data, and pays no attacker bounty.
- Sustaining the call for 24 hours keeps actual victim-funded position collateral inaccessible.
- The nearest valid control reaches the documented owner amendment and closes normally.
