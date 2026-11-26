# \#100065 [SC-Low] CfdEngineAdmin and MarginClearinghouse omit the renounceOwnership guard already applied to three sibling contracts, permanently disabling timelocked admin configuration

**Submitted on Nov 16th 2026 at 07:12:39 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100065
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

# Description

**Category: Insight - Security Best Practices.** Three owner-controlled contracts in the `packages/perps/src` tree —
`EmergencyPauseCoordinator`, `HousePool`, and `OrderRouterAdmin` — override the inherited
`Ownable.renounceOwnership()` with an identical guard-and-revert pattern, disabling self-renunciation. At the pinned
commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, two other owner-controlled contracts in the same tree,
`CfdEngineAdmin` and `MarginClearinghouse`, inherit the same `Ownable2Step` base but do not carry this guard.

`CfdEngineAdmin` is the primary material target because renunciation permanently disables all nine timelocked risk,
calendar, and freshness administration paths, and the already-bound engine has no in-protocol replacement-admin
path. `MarginClearinghouse` is secondary corroborating scope: after engine binding, its incremental
post-renunciation loss is principally `rescueToken`.

## Residual targets

The primary target, `CfdEngineAdmin` (`packages/perps/src/CfdEngineAdmin.sol`), is, in its own NatSpec (lines 10-15),
a "timelocked, two-step owner-controlled administrator for risk, FAD calendar, and mark freshness," with a 72-hour
`TIMELOCK_DELAY` (line 20) between proposing and finalizing a configuration change. It inherits `Ownable2Step`
(line 17: `contract CfdEngineAdmin is Ownable2Step`) but **does not override `renounceOwnership()`**.
The secondary target, `MarginClearinghouse` (`packages/perps/src/MarginClearinghouse.sol`, line 30:
`contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTransient`) also inherits
`Ownable2Step` and also **does not override
`renounceOwnership()`**. Grepping both files directly for `function renounceOwnership` returns no match in either.

Because neither contract overrides it, OpenZeppelin's base `Ownable.renounceOwnership()` is live on both: the
current owner can call it at any time, unconditionally, transferring ownership to `address(0)` irreversibly. (Note:
`Ownable2Step` changes how a *transfer* to a new owner works — a two-step propose/accept — but it does not itself
override `renounceOwnership()`, so renunciation on both contracts bypasses the two-step protection entirely and
goes straight through the base `Ownable` path.)

## Three sibling contracts already implement the same guard

Three other owner-controlled contracts in the same `packages/perps/src` tree override `renounceOwnership()` with an
identical function body and revert. `HousePool` and `OrderRouterAdmin` use the NatSpec sentence shown below;
`EmergencyPauseCoordinator` uses a guardian-specific rationale.

```solidity
/// @notice Ownership cannot be renounced because governance must retain emergency recovery authority.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

- `EmergencyPauseCoordinator.sol` line 102.
- `HousePool.sol` line 265.
- `OrderRouterAdmin.sol` line 125.

All three use the identical function body and revert (`public view override onlyOwner { revert
OwnableInvalidOwner(address(0)); }`). `HousePool.sol` and `OrderRouterAdmin.sol` use byte-for-byte the same NatSpec
sentence quoted above. `EmergencyPauseCoordinator.sol` instead states that governance must retain authority to rotate
or disable the guardian. The standardized element is the guard's function body and governance-recovery purpose, while
the NatSpec rationale is contract-specific.

## Concrete consequences and target priority

OpenZeppelin's unmodified base `renounceOwnership()` is
`function renounceOwnership() public virtual onlyOwner { _transferOwnership(address(0)); }`
(`lib/openzeppelin-contracts/contracts/access/Ownable.sol` line 76). The `onlyOwner` modifier checks the caller
against `owner()`; after ownership becomes `address(0)`, no calling account, including the former owner, can satisfy
that modifier.

- On the primary target, `CfdEngineAdmin`, `renounceOwnership()` succeeds because no override blocks it and sets
  `owner() == address(0)`. Every one of this contract's nine `onlyOwner` functions permanently
  reverts for everyone, including the former owner and an arbitrary third party: `proposeRiskConfig()` (line 109),
  `finalizeRiskConfig()` (line 131), `cancelRiskConfig()` (line 143), `proposeCalendarConfig()` (line 155),
  `finalizeCalendarConfig()` (line 173), `cancelCalendarConfig()` (line 187), `proposeFreshnessConfig()` (line 207),
  `finalizeFreshnessConfig()` (line 221), and `cancelFreshnessConfig()` (line 233). `acceptOwnership()` (inherited
  from `Ownable2Step`) reverts too, since there is no pending owner and no path to create one. A proposal made
  before renunciation stays recorded forever — there is no expiry and no way to clear it.
- The engine's binding to its admin contract cannot be redirected to a freshly deployed replacement admin either:
  `CfdEngine.setDependencies(...)` is a one-shot call that reverts `CfdEngine__DependenciesAlreadySet()` once
  already bound (`packages/perps/src/CfdEngine.sol` line 354). The already-bound engine's risk, calendar, and
  freshness administration path is therefore permanently locked after this single renunciation.
- On the secondary target, `MarginClearinghouse`, renunciation similarly succeeds and both `onlyOwner` functions
  become unavailable: `setEngine(...)` (line 306) and `rescueToken(...)` (line 324). `setEngine` is already one-shot
  after binding, so the incremental post-binding consequence is the permanent loss of `rescueToken`.
- `MarginClearinghouse.withdraw(...)` (line 403) and
  `withdrawMargin(...)` (line 414) carry no `onlyOwner` (or any other owner-dependent) gate anywhere in either
  function, so ordinary trader withdrawals remain callable regardless of the contract's ownership state.

## Category fit: Security Best Practices

The recommendation is to **disable one specific, irreversible, self-directed privileged action** while leaving every
risk, calendar, freshness, engine-binding, and rescue input unchanged. Category 3's Invalid Example concerns
"validation checks for change operations performed by privileged or access-controlled functions"; this report
proposes no input-validation change. The three existing sibling guards establish the recommendation as a security guard rather than an architecture,
modularity, decentralization, or composability change.

Category 1's own definition — "strengthens the security design or implementation of the code... addressing
potential threats, even in the absence of an immediate or known vulnerability" — and its own Valid Examples
(adopting Checks-Effects-Interactions, adopting SafeMath) describe exactly this class of ask: adopt an
already-proven-in-this-codebase defensive pattern on the two contracts that are missing it.

## Operational documentation does not identify the two residual targets

No project operational document identifies
`CfdEngineAdmin` or `MarginClearinghouse` as missing this guard or explains their
post-renunciation consequences. `CfdEngineAdmin`'s NatSpec (lines 10-15) describes it as "timelocked, two-step
owner-controlled" without addressing renunciation. `SECURITY.md`'s "Instant owner controls" enumeration and
`EMERGENCY_RESPONSE_GUIDE.md`'s "Canonical references" table discuss `CfdEngine`-lineage owner actions without
identifying either residual target. `DEPLOYMENT.md` states generally that ownership-bearing contracts use
`Ownable2Step` and that a proposed new owner must call `acceptOwnership()`; that describes the transfer path rather
than the inherited renunciation path or the consequences above.

Applying the existing sibling guard to primary `CfdEngineAdmin` and secondary `MarginClearinghouse` closes
the exact residual shown at the pinned commit.

## References

- `packages/perps/src/CfdEngineAdmin.sol` line 17 (contract declaration, `Ownable2Step`), line 20
  (`TIMELOCK_DELAY`), lines 109, 131, 143, 155, 173, 187, 207, 221, 233 (`proposeRiskConfig`, `finalizeRiskConfig`,
  `cancelRiskConfig`, `proposeCalendarConfig`, `finalizeCalendarConfig`, `cancelCalendarConfig`,
  `proposeFreshnessConfig`, `finalizeFreshnessConfig`, `cancelFreshnessConfig`, all `onlyOwner`) — no
  `renounceOwnership` override anywhere in this file.
- `packages/perps/src/MarginClearinghouse.sol` line 30 (contract declaration, `Ownable2Step`), line 306
  (`setEngine`), line 324 (`rescueToken`), lines 403 and 414 (`withdraw`, `withdrawMargin`, neither `onlyOwner`) —
  no `renounceOwnership` override anywhere in this file.
- `packages/perps/src/EmergencyPauseCoordinator.sol` line 102, `packages/perps/src/HousePool.sol` line 265,
  `packages/perps/src/OrderRouterAdmin.sol` line 125 — the existing override-and-revert pattern on the three
  sibling contracts.
- `packages/perps/src/CfdEngine.sol` line 354 (`CfdEngine__DependenciesAlreadySet`, inside `setDependencies`) — the
  one-shot admin-binding mechanism that forecloses a replacement-admin recovery path.
- frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.


## Proof of Concept

# Proof of Concept

**Conditions under which this Insight is valuable.** 

This is a structural property of the pinned source: the inherited `renounceOwnership()` guard is absent from primary `CfdEngineAdmin` and secondary `MarginClearinghouse` regardless of market state. Its value is highest before mainnet deployment: release v2.0.0 deploys the in-scope code to Arbitrum Sepolia, and the programme page states the perps contracts "are not upgradeable," so closing the residual after deployment requires a contract migration. Adding the existing guard before deployment avoids that migration.

**Read-only reproduction at the frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.** The steps below confirm
the inherited behavior and contract-specific consequences from source:

1. Clone and checkout:
   ```
   git clone https://github.com/immunefi-team/audit-comp-plether.git
   cd audit-comp-plether
   git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
   cd packages/perps
   ```
2. Read primary `src/CfdEngineAdmin.sol` line 17 (`contract CfdEngineAdmin is Ownable2Step`) and secondary
   `src/MarginClearinghouse.sol`
   line 30 (`contract MarginClearinghouse is IMarginAccount, Ownable2Step, ReentrancyGuardTransient`), and confirm by
   direct text search (`grep -n "function renounceOwnership"`) that neither file defines an override.
3. Read `src/EmergencyPauseCoordinator.sol` line 102, `src/HousePool.sol` line 265, and `src/OrderRouterAdmin.sol`
   line 125: confirm all three already carry the identical override-and-revert pattern quoted in `03_DESCRIPTION.md`
   (the function body and revert are identical on all three; `EmergencyPauseCoordinator.sol` alone uses a different
   NatSpec sentence, giving a guardian-specific rationale rather than the other two's emergency-recovery wording).
4. Confirm the OpenZeppelin base behavior directly: `grep -n -A2 "function renounceOwnership" lib/openzeppelin-contracts/contracts/access/Ownable.sol`
   shows `function renounceOwnership() public virtual onlyOwner { _transferOwnership(address(0)); }` — the
   unmodified base path both target contracts fall through to, since neither overrides it.
5. Read every `onlyOwner` function on both target contracts directly (`grep -n onlyOwner src/CfdEngineAdmin.sol
   src/MarginClearinghouse.sol`) and confirm the function lists and line numbers quoted in `03_DESCRIPTION.md`'s
   "Concrete consequences and target priority" section and `06_REFERENCES.md`. Read
   `src/CfdEngine.sol` line 354 and confirm the one-shot dependency binding prevents redirecting the engine to a new
   admin after it is bound.
6. Read `src/MarginClearinghouse.sol` lines 403 and 414 (`withdraw`, `withdrawMargin`) and confirm neither carries
   an `onlyOwner` or other owner-dependent gate; ownership state therefore does not gate those ordinary trader
   withdrawal functions.

**Fix/control check:** the remediation in `05_REMEDIATION.md` adds the sibling override-and-revert pattern first to
primary `CfdEngineAdmin` and likewise to secondary `MarginClearinghouse`. That change makes
`renounceOwnership()` fail exactly as it already does on `EmergencyPauseCoordinator`, `HousePool`, and
`OrderRouterAdmin`, confirmed by direct comparison against those contracts (step 3). The override is absent from
both residual targets at the pinned commit (step 2).



