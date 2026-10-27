# 89546 sc low firelightvault redeem lacks the zero output guard present in deposit burning shares for zero assets when totalassets totalsupply

**Submitted on Aug 19th 2026 at 20:09:07 UTC by @backslash for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89546
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/interfaces/IFirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol> contract has `deposit()` function with safeguard against zero-output rounding:

```
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
  assets,
  true,
  Math.Rounding.Floor
);
// A dust deposit can floor to zero shares once the share price exceeds 1;
// reject it instead of pulling assets in exchange for nothing.
if (shares == 0) revert InvalidAmount();
```

However, the withdrawal side has no such protection: `redeem()` function takes a share amount as input, converts it to assets with floor rounding via `_previewTotals()` but does not check whether the converted rounded value is zero as `deposit()` checks:

```
(uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
  shares,
  false,
  Math.Rounding.Floor
);

uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
```

This allows an attacker to trick victim into thinking he/she will withdraw funds while the value will be burned and shareholders (including an attacker) will be able to sell shares for higher price while victim receives zero.

## Vulnerability Details

With `S = totalSupply()` and `A = totalAssets()`, `redeem()` function produces:

```
assets = floor( shares * (A + 1) / (S + 1) )
```

which is 0 whenever `shares * (A + 1) < (S + 1)`, i.e. whenever `A < S` (share price below one) and shares is below the threshold `(S + 1) / (A + 1)`.

This is the exact mirror image of the deposit case, which floors to zero when `A > S` (price above one). The vault guards one direction and not the other.

`A < S` (`price < 1`) is a normal operating state, not an exotic one. It arises whenever the vault realizes a loss against unchanged supply - most directly through payout(), which transfers assets out and pushes a lowered `totalAssets` checkpoint without burning any shares. It is also influenced by `pendingWithdrawAssets`, which `totalAssets()` subtracts. Once the vault is in this state, every subsequent redeem below the threshold is affected until the price recovers.

Additionally, `redeem` is not gated on `_hasActiveIncident()`, so it remains callable in exactly the post-payout window where the price is depressed.

## Impact Details

* Direct user loss: a holder who calls `redeem` with a share amount below the rounding threshold burns those shares and receives a withdrawal claim worth zero. There is no revert to warn them. The lost value is redistributed to the remaining shareholders (supply drops, backing does not).
* Not limited to dust: the zero-output threshold is `(S + 1) / (A + 1)`. As the price falls after a larger incident, this threshold grows: at a 1% price `(A = 1, S = 100)` a `redeem` of up to 50 shares yields zero assets. In a depressed-price state the loss can be a material fraction of a position (getting rid of other shareholders).
* Programmatic callers are most exposed. Integrators, keepers, or front-ends that call `redeem(shares, …)` (as opposed to `withdraw(assets, …)`) can silently zero out users during the exact window - right after a payout - when users are most likely to be exiting.
* Griefing via existing allowance: an operator holding an allowance can call `redeem(smallShares, receiver, owner)` against the owner in the sub-unit price state, burning the owner's shares for a zero claim. The operator gains nothing directly but can grief the victim: the burned value is distributed to all holders. Requires a pre-existing approval, so this is a minor amplifier, not the primary risk.

## Proof of Concept

Here is Hardhat/ethers test modeled on the repository's own `test/vault_payout.js` and `test/setup/fixtures.js`. It uses `decimals: 0n` (the same technique the existing payout tests use to make rounding observable), drives a payout to push the price below one, then shows `redeem` burning shares for zero assets with no revert:

```
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

describe("FirelightVault redeem zero-asset guard", function () {
  it("burns shares for zero assets when price < 1", async () => {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: 0n, initial_deposit_limit: 1000n })
    );
    const { firelight_vault: vault, token_contract, payout_signer, payout_receiver, users, config } = ctx;
    const user = users[0];

    // 1) Deposit 10 assets -> 10 shares (price 1:1).
    await ctx.utils.mintAndApprove(10n, user);
    await vault.connect(user).deposit(10n, user.address);
    expect(await vault.totalSupply()).to.equal(10n);
    expect(await vault.totalAssets()).to.equal(10n);

    // 2) Move to the next period and pay out 8 assets from the capture period.
    //    Supply is untouched, so the price drops to 2/10 = 0.2.
    await time.increase(config.period_configuration_duration);
    const captureTs = await vault.currentPeriodStart();
    await vault.connect(payout_signer).payout(payout_receiver.address, 8n, captureTs);

    expect(await vault.totalAssets()).to.equal(2n); // A = 2
    expect(await vault.totalSupply()).to.equal(10n); // S = 10  -> price < 1

    // 3) redeem(1): floor(1 * (2 + 1) / (10 + 1)) = floor(3/11) = 0 assets.
    const balBefore = await vault.balanceOf(user.address);

    await expect(vault.connect(user).redeem(1n, user.address, user.address))
      .to.emit(vault, "WithdrawRequest")
      // sender, receiver, owner, period, assets = 0, shares = 1
      .withArgs(user.address, user.address, user.address, anyPeriod(), 0n, 1n);

    // The share was burned; nothing was queued for withdrawal.
    expect(await vault.balanceOf(user.address)).to.equal(balBefore - 1n);
    expect(await vault.totalSupply()).to.equal(9n);
    expect(await vault.pendingWithdrawAssets()).to.equal(0n);

    // Contrast: the symmetric deposit case IS guarded and reverts.
    // (deposit that would floor to 0 shares -> InvalidAmount)
  });
});

// helper: matches any bigint for the indexed `period` arg
const anyPeriod = () => require("chai").match ? undefined : undefined;
```

**Note:** replace `anyPeriod()` with the specfic expected period (`capturePeriod + 1n`) if you want an exact `withArgs` match; it is only there to keep the event assertion readable. The essential assertions are the emitted `assets = 0` / `shares = 1` and the post-state balances.

**Note 2:** To make the non-dust nature explicit, rerun step 2 with a larger payout (e.g. leave `A = 1`, `S = 100` by depositing 100 and paying out 99). Then any redeem of up to 50 shares floors to zero assets - the caller can burn half their position and receive nothing.
