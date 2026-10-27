# 89825 sc low missing assets 0 guard in firelightvault redeem burns a user s shares for zero withdrawal credit when the share price is below 1

**Submitted on Aug 21st 2026 at 11:16:54 UTC by @aksme for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89825
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

`FirelightVault.redeem()` validates only that `shares != 0`, but never checks that the resulting `assets` is non-zero after the floor-rounded preview. When the vault's share price is below 1 (assets per share < 1), a `redeem()` for a small number of shares floors to `assets == 0`. The call still proceeds: it burns the caller's shares and creates a withdrawal request for zero assets. The user permanently loses the burned shares and receives nothing in return.

This is confirmed by an asymmetry in the contract's own code: `deposit()` and `withdraw()` both guard against the symmetric zero case, and `deposit()` even carries an explicit comment describing this exact floor-to-zero hazard. `redeem()` is the one withdrawal entry point that omits the guard.

## Root cause

In `contracts/core/FirelightVault.sol`, `redeem()`:

```
function redeem(uint256 shares, address receiver, address owner) public override ... {
    if (shares == 0) revert InvalidAmount();

    (uint256 assets, uint256 _totalSupply, uint256 _totalAssets) =
        _previewTotals(shares, false, Math.Rounding.Floor);   // <-- floors to 0 when price < 1

    uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);  // <-- burns shares even when assets == 0

    _logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
    return assets;
}
```

`_previewTotals(..., Math.Rounding.Floor)` returns `assets == 0` for any `shares` small enough that `shares * totalAssets / totalSupply` rounds down to zero — which happens whenever the share price is below 1. The function only reverts on `shares == 0`; it never reverts on `assets == 0`. `_requestWithdraw` is then called with `assets == 0`: it burns the caller's `shares` via `_update(owner, 0, shares)` and credits `withdrawSharesOf[period] += 0`. Net effect: shares destroyed, zero withdrawal credit created.

## The asymmetry that shows this is an oversight

The protocol enforces the symmetric guard everywhere else in the same file:

* `deposit()`: `if (assets == 0) revert InvalidAmount();` — plus the explicit comment: "A dust deposit can floor to zero shares once the share price exceeds 1; reject it instead of pulling assets in exchange for nothing."
* `withdraw()`: `if (assets == 0) revert InvalidAmount();`
* `redeem()`: **no `assets == 0` check.**

The developers demonstrably understood this bug class (they wrote a comment about it and guarded two of the three functions), but the guard was not applied to `redeem()`.

## Reachability

A share price below 1 (totalAssets < totalSupply) is the normal, expected state of this vault after any payout. `payout()` reduces the vault's assets while total supply is unchanged, so every incident payout drives the price below 1 — this is core protocol behaviour, not an exotic edge case.

The loss is reachable two ways:

1. A user directly calling `redeem()` with a small share amount in the post-payout state.
2. An ERC4626 integrator following the standard: `maxRedeem()` returns `balanceOf(owner)` without accounting for the sub-1 price, so an integrator that reads `maxRedeem` and redeems in small increments walks users into zero-credit burns.

## Impact

A user permanently loses the shares burned by such a `redeem()` call and receives zero assets in return — an uncompensated loss of value. This is a failure of the contract to deliver the promised withdrawal for a permissionless, user-facing entry point. It contrasts with the protocol's own stated intent, since `deposit()`/`withdraw()` explicitly reject the equivalent zero case.

## Recommended fix

Add the same guard `redeem()` is missing, mirroring `deposit()` and `withdraw()`:

```
(uint256 assets, ...) = _previewTotals(shares, false, Math.Rounding.Floor);
if (assets == 0) revert InvalidAmount();
```

## Affected code

contracts/core/FirelightVault.sol — redeem() (the `shares == 0`-only guard, and the subsequent `_requestWithdraw(assets, shares, ...)` call reached with assets == 0).

## Link to Proof of Concept

<https://gist.github.com/AkshataM17/388dd4e357f38f9ced57eabf29a50210>

## Proof of Concept

The PoC runs against the in-scope repo at the frozen commit: repo: github.com/immunefi-team/audit-comp-firelight branch: v1\_audit\_ready commit: 42f9ea5e43d88b35197b901acc2a8c24b314fa62

It uses the project's own Hardhat setup with a MockERC20 as the vault asset, drives the share price below 1 through the protocol's real payout mechanism (no injected values), then performs the zero-credit redeem.

Step-by-step:

1. Deploy FirelightVault behind the standard upgradeable proxy (via upgrades.deployProxy) over a 6-decimal MockERC20, with a large deposit limit and a 1-week period duration.
2. Grant PAYOUT\_ROLE and PAYOUT\_ALLOWLIST\_ROLE, and add a sink address to the payout allowlist (standard payout wiring, identical to the project's own fixtures).
3. A user deposits 1000 tokens -> receives 1000e6 shares at price 1.0.
4. Advance one period (so the deposit is baked into the period-start checkpoint that payout() caps against).
5. A trusted PAYOUT\_ROLE payout of 999 tokens is executed against the captured period. This is used only to establish the below-1 share price that is the finding's precondition — it is normal core protocol behaviour, not the exploit step. After it, totalAssets = 1,000,000 and totalSupply = 1,000,000,000, i.e. share price = 0.001.
6. Read previewRedeem(1): returns 0. A 1-share redeem floors to zero assets.
7. The user calls redeem(1, user, user). Because redeem() only checks `shares == 0`, the call SUCCEEDS: the share is burned (balanceOf drops by 1) and zero withdrawal credit is created.

Observed result (console output from the run):

after haircut: totalSupply=1000000000 totalAssets=1000000 price=0.001 previewRedeem(1 share) = 0 assets (floors to zero) shares burned=1, withdrawal credit=0 -> uncompensated loss

The share was destroyed and the user received nothing. Repeating this (or an integrator following maxRedeem) compounds the loss across many shares — any redeem of up to \~(totalSupply+1)/(totalAssets+1) shares floors to zero.

By contrast, deposit() and withdraw() both revert with InvalidAmount() in the symmetric zero case; only redeem() proceeds.

The full runnable test file is provided in the secret Gist below. Run with: npx hardhat test test/zzz\_reverify.js --grep "#1"
