# \#98940 [SC-Low] Incomplete LOW-59 fix: `renounceOwnership` is still callable on the engine, the engine admin and the clearinghouse

**Submitted on Nov 11th 2026 at 02:36:30 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98940
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

**The audit item.** Cecuro LOW-59 ("Owner can leave emergency recovery ownerless") found that governance could permanently remove its own recovery authority by calling the inherited `Ownable.renounceOwnership()` on the perps `Ownable2Step` contracts. Its remediation reads: override `renounceOwnership()` in every governance-bearing perps contract and make it revert.

**The fix at `a82d4fe`.** The override exists on `HousePool` (`HousePool.sol:265`), `OrderRouterAdmin` (`OrderRouterAdmin.sol:125`) and `EmergencyPauseCoordinator` (`EmergencyPauseCoordinator.sol:102`), the three contracts listed in the item's Locations.

**What was left out.** Three more `Ownable2Step` contracts of the same system keep the inherited function unchanged:
- `CfdTreasury` (`CfdTreasury.sol:10`), inherited by `CfdEngine`;
- `CfdEngineAdmin` (`CfdEngineAdmin.sol:17`);
- `MarginClearinghouse` (`MarginClearinghouse.sol:30`).

The engine owner is the documented recovery authority: `EMERGENCY_RESPONSE_GUIDE.md:47` ("The Engine owner can call `clearDegradedMode()` once raw `E >= L`") and `SECURITY.md:157` (`recapitalizePool` is "an immediate, owner-only, non-reentrant recovery action"). The engine admin owner holds the timelocked risk, calendar and freshness configuration (`CfdEngineAdmin.sol:143-233`); the clearinghouse owner holds the engine/router binding and token rescue. One `renounceOwnership()` call on any of them makes those functions permanently uncallable, which is the consequence LOW-59 was raised and fixed for on the other three contracts. A renounced engine makes a degraded-mode latch irreversible on-chain, a stronger consequence than the unpause paths the fix protects.

**Measured in the PoC** (repo fixture, production `CfdEngine` / `CfdEngineAdmin` / `MarginClearinghouse` / `HousePool` / `OrderRouterAdmin` / `EmergencyPauseCoordinator`): as the owner, `renounceOwnership()` reverts `OwnableInvalidOwner(0x0)` on the three fixed contracts and succeeds on the three others (`owner() == address(0)`); afterwards `clearDegradedMode()`, `cancelRiskConfig()` and `rescueToken()` revert `OwnableUnauthorizedAccount`.

## Vulnerability Details

Code at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

### 1. The override exists on three contracts

`HousePool.sol:265`, `OrderRouterAdmin.sol:125` and `EmergencyPauseCoordinator.sol:102` define `function renounceOwnership() public view override onlyOwner { revert OwnableInvalidOwner(address(0)); }`. Two-step transfers stay available.

### 2. Three contracts inherit the OpenZeppelin default

`CfdTreasury` (`CfdTreasury.sol:10`, the base of `CfdEngine`), `CfdEngineAdmin` (`CfdEngineAdmin.sol:17`) and `MarginClearinghouse` (`MarginClearinghouse.sol:30`) declare `Ownable2Step` without the override, so `Ownable.renounceOwnership()` (`lib/openzeppelin-contracts/contracts/access/Ownable.sol:76-78`) sets the owner to `address(0)`.

### 3. The authority that is lost

- `CfdEngine`: `clearDegradedMode()` (`CfdEngine.sol:658`), `recapitalizePool()` (`:634`), the dependency setters (`:349-432`) and the treasury proposal flow of `CfdTreasury`.
- `CfdEngineAdmin`: proposing, finalizing and cancelling risk, calendar and freshness configuration (`CfdEngineAdmin.sol:143-233`).
- `MarginClearinghouse`: `setEngine` (`MarginClearinghouse.sol:306`) and `rescueToken` (`:319-336`).

### 4. Why the engine case matters more than the fixed ones

Degraded mode is cleared only by the engine owner once `E >= L` (`CfdEngine.sol:658-667`); the documented non-dilutive recapitalization is engine-owner-only (`:634`). After a renounce, a latched degraded mode has no on-chain exit, while the paths the fix protects (pool unpause, router config, guardian management) are recoverable through the remaining owners.

### Relation to prior audits and known issues

LOW-59 is marked Fixed; the program accepts a report on a Fixed root cause when the fix at `a82d4fe` is incomplete. The item's remediation covers every governance-bearing perps contract, and the three contracts above are governance-bearing in the same sense (owner-only recovery and configuration). `SECURITY.md:128` ("renouncing ownership does not prevent activation") concerns the treasury lock only. No other audit item or document covers renunciation on the engine, the engine admin or the clearinghouse.

## Impact Details

In-scope impact: **Low: "Contract fails to deliver promised returns, but doesn't lose value"**. The promised return is the LOW-59 remedy itself: ownership of a contract whose owner holds recovery authority cannot be renounced. Half of the contracts deliver it; the engine, engine admin and clearinghouse do not. No funds are lost by the omission; after a renounce the degraded-mode exit, the recapitalization, the configuration flows and token rescue become permanently uncallable.

The trigger is an owner transaction, as in the original LOW-59, which the audit rated Low and the project fixed. The severity is the audit's own.

### Preconditions

- Any time after deployment; one transaction by the owner of the affected contract.
- No external state, price or third party is involved.

## Recommendation

Add the same override to the three remaining contracts (see `fix.diff`):

```solidity
/// @notice Ownership cannot be renounced: the owner holds recovery and configuration authority.
function renounceOwnership() public view override onlyOwner {
    revert OwnableInvalidOwner(address(0));
}
```

in `CfdTreasury`, `CfdEngineAdmin` and `MarginClearinghouse`. Two-step transfers remain available. With the patch applied, the PoC's success-path assertions fail because all six contracts revert.


## Link to Proof of Concept

https://gist.github.com/the-red-ace/84353bbc5f25d8f8dfb678a9a142c807

## Proof of Concept

## Proof of Concept

The gist https://gist.github.com/the-red-ace/84353bbc5f25d8f8dfb678a9a142c807 holds `L3Renounce.t.sol`, the fixture files (`G1Base.sol`, `X2GridBase.sol`, `X2ErrNames.sol`), `fix.diff` and a README with the exact commands.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/l3renounce
cp X2GridBase.sol X2ErrNames.sol G1Base.sol L3Renounce.t.sol packages/perps/test/perps/l3renounce/
forge test --root packages/perps --match-path 'test/perps/l3renounce/L3Renounce.t.sol' -vv
```

Expected: 1 passed. The log shows the three fixed contracts reverting `OwnableInvalidOwner(0x0)` and keeping their owner, the three others renouncing to `owner() == address(0)`, and `clearDegradedMode()`, `cancelRiskConfig()` and `rescueToken()` then reverting `OwnableUnauthorizedAccount`. With `fix.diff` applied (`git apply fix.diff` from the repo root), the test fails at the first expected-success call.

The test contract is the deployer and owner of the fixture, so no role is faked; the `EmergencyPauseCoordinator` is deployed by the test because the fixture omits it. Cheats: `vm.prank` as the owner and `vm.expectRevert`. No in-scope contract is modified.
