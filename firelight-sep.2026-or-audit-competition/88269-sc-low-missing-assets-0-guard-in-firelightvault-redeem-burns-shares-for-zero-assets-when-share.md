# 88269 sc low missing assets 0 guard in firelightvault redeem burns shares for zero assets when share price 1 unfixed variant of macro l 1&#x20;

**Submitted on Aug 12th 2026 at 14:30:49 UTC by @promael for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88269
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Firelight fixed Macro finding L-1 ("deposit can mint zero shares") by rejecting the zero case on the value the caller receives: `deposit` and `mint` both revert `InvalidAmount()` when the computed `shares == 0`. The symmetric guard on the redemption path was not added. `FirelightVault.redeem()` checks only its input (`shares == 0`) and never checks that the computed `assets != 0`. When the vault share price is below 1 — the expected state after a `payout()` loss, where `totalAssets < totalSupply` — a `redeem()` for a small share amount computes `assets = 0` (floor rounding) but still burns the caller's shares and queues nothing for withdrawal. The caller (or a third party spending an allowance) permanently loses those shares for zero assets; the value is redistributed to the remaining holders.

## Vulnerability Details

In `contracts/core/FirelightVault.sol`, the deposit path carries the L-1 fix:

deposit(...) (\~660-673): (uint256 shares, , ) = \_previewTotals(assets, true, Math.Rounding.Floor); if (shares == 0) revert InvalidAmount(); // rejects the zero case

mint(...) (\~697): if (shares == 0) revert InvalidAmount();

The redeem path does not:

redeem(uint256 shares, address receiver, address owner) (717-744): if (shares == 0) revert InvalidAmount(); // guards INPUT only (uint256 assets, uint256 \_totalSupply, uint256 \_totalAssets) = \_previewTotals(shares, false, Math.Rounding.Floor); // assets can be 0 // <-- no `if (assets == 0) revert InvalidAmount();` uint256 ownerBalance = \_requestWithdraw(assets, shares, receiver, owner); \_logTrace(owner, ownerBalance, \_totalSupply - shares, \_totalAssets - assets, true); return assets; // returns 0

`_requestWithdraw(assets = 0, shares, ...)` (1000-1028) still burns the shares while queuing nothing: withdrawAssets\[period] += assets; // += 0 withdrawShares\[period] += sharesWithdraw; // convertToShares(0) = 0 withdrawSharesOf\[period]\[receiver] += sharesWithdraw; // += 0 pendingWithdrawAssets += assets; // += 0 \_update(owner, address(0), shares); // <-- SHARES BURNED

Conversion (1067-1074): \_convertToAssetsTotals(shares) = shares.mulDiv(totalAssets + 1, totalSupply + 10\*\*offset, Floor)

With `totalAssets < totalSupply`, any `shares` small enough that `shares*(totalAssets+1) < (totalSupply + 10**offset)` yields `assets = 0`. This state is reached whenever the vault has realized a loss via `payout()` (an expected event for a cover vault). `redeem()` has no `whenNotPaused`-independent guard against it, so the share is destroyed with no asset delivered and nothing recorded in the withdrawal buckets, i.e. the loss is permanent (the user cannot later claim anything for those shares).

## Impact Details

When `totalAssets < totalSupply` (share price < 1, the normal post-payout state), a `redeem()` of a share amount that floors to zero assets burns the caller's shares and delivers/queues nothing. The affected party permanently loses those shares for zero value; the value accrues to the remaining share holders. It is deterministic (no revert) and is also reachable by a third party via `redeem(shares, receiver, owner)` spending an allowance over a victim's shares. No net protocol value is lost (it redistributes), and the per-call magnitude is bounded by the share-price shortfall (dust-scale), so the impact is Low — but it is a genuine, unfixed variant of the already-reported-and-fixed L-1 and is contrary to the vault's own zero-guard convention on the deposit/mint side. Selected impact: "Contract fails to deliver promised returns, but doesn't lose value."

## References

* Affected code: contracts/core/FirelightVault.sol — `redeem` (L717-744), `_requestWithdraw` (L1000-1028), `_convertToAssetsTotals` (L1067-1074); guarded counterpart in `deposit`/`mint`.
* Macro audit A-1, finding L-1 ("deposit can mint zero shares", fixed for deposit/mint only): <https://library-pr-440.onrender.com/library/audits/firelight-1>

## Link to Proof of Concept

<https://gist.github.com/Kyenghwan-Hwang/de9da7a0bb1281265737079bedacb2e2>

## Proof of Concept

## Proof of Concept

Runnable Hardhat test against the in-scope source (uses the repo's own `deployVault` fixture). Place at `test/ZzRedeemZeroPoc.js` and run `npx hardhat test test/ZzRedeemZeroPoc.js` — it passes.

Step-by-step:

1. Deploy the vault (fixture, `decimals: 0`) and have a user `deposit(100)` → `totalSupply = 100`, `totalAssets = 100` (share price = 1).
2. Advance one period; a `payout()` of 99 (PAYOUT\_ROLE) realizes a loss → `totalAssets = 1`, `totalSupply = 100` (share price = 0.01). Confirm `convertToAssets(1) == 0`.
3. The user calls `redeem(1, user, user)`. It returns `0` assets (static call confirms) yet burns the share: `totalSupply 100 → 99`, user balance −1, and `withdrawSharesOf[period][user] == 0` / `withdrawAssets[period] == 0` (nothing queued → permanent loss).
4. Contrast: `deposit(0)` reverts `InvalidAmount` — the guarded side rejects the symmetric zero case, the redeem side does not.

Test:

```
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

describe("PoC: redeem() burns shares for zero assets (unfixed variant of Macro L-1)", function () {
  it("redeem(dust) when share price < 1 destroys shares and returns 0 assets", async () => {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: 0n, initial_deposit_limit: 1000n })
    );
    const { firelight_vault, payout_signer, payout_receiver, users } = ctx;
    const user = users[0];

    // 1) deposit 100 -> supply=100, assets=100 (price 1)
    await ctx.utils.mintAndApprove(100n, user);
    await firelight_vault.connect(user).deposit(100n, user.address);
    const sharesBefore = await firelight_vault.balanceOf(user.address); // 100

    // 2) payout takes a loss -> assets=1, supply=100 (price 0.01)
    await time.increase(ctx.config.period_configuration_duration);
    const captureTimestamp = await firelight_vault.currentPeriodStart();
    await firelight_vault.connect(payout_signer).payout(payout_receiver.address, 99n, captureTimestamp);
    expect(await firelight_vault.totalAssets()).to.equal(1n);
    expect(await firelight_vault.totalSupply()).to.equal(100n);
    expect(await firelight_vault.convertToAssets(1n)).to.equal(0n);

    // 3) redeem(1) returns 0 assets but burns the share
    const period = (await firelight_vault.currentPeriod()) + 1n;
    expect(
      await firelight_vault.connect(user).redeem.staticCall(1n, user.address, user.address)
    ).to.equal(0n);
    await firelight_vault.connect(user).redeem(1n, user.address, user.address);
    expect(await firelight_vault.balanceOf(user.address)).to.equal(sharesBefore - 1n); // burned
    expect(await firelight_vault.totalSupply()).to.equal(99n);
    expect(await firelight_vault.withdrawSharesOf(period, user.address)).to.equal(0n);  // nothing owed
    expect(await firelight_vault.withdrawAssets(period)).to.equal(0n);

    // 4) guarded side rejects the zero case
    await expect(firelight_vault.connect(user).deposit(0n, user.address))
      .to.be.revertedWithCustomError(firelight_vault, "InvalidAmount");
  });
});
```

Recommended fix: mirror the deposit/mint guard in `redeem` after computing `assets`: if (assets == 0) revert InvalidAmount(); (`withdraw` is unaffected: it rounds shares up and its `assets` input is user-supplied and already `!= 0`-checked.)
