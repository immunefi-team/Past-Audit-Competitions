# 88785 sc low firelightvault redeem burns the caller s shares and queues nothing when the redeemed asset value floors to zero the assets 0 guard present on deposit and withdraw is missing

**Submitted on Aug 15th 2026 at 02:58:44 UTC by @Darkseid for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88785
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

`redeem()` is missing a zero-check that both of its sibling functions have, and the result is that a user can destroy a chunk of their own position and get nothing back.

`deposit()` rejects a deposit that would floor to zero shares, and the developers left a comment explaining exactly why:

```solidity
// contracts/core/FirelightVault.sol:667-669
// A dust deposit can floor to zero shares once the share price exceeds 1;
// reject it instead of pulling assets in exchange for nothing.
if (shares == 0) revert InvalidAmount();
```

`withdraw()` is covered too, since it rejects a zero `assets` argument at L769. `redeem()` only checks the argument it was handed:

```solidity
// contracts/core/FirelightVault.sol:730-743
if (shares == 0) revert InvalidAmount();               // the only guard

(uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
    shares, false, Math.Rounding.Floor                  // assets can floor to 0 here
);

uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
```

Nothing downstream catches it. `_requestWithdraw` runs the full bookkeeping with `assets == 0`, adding zero to every accumulator, and then burns the shares anyway:

```solidity
// contracts/core/FirelightVault.sol:1008-1025
uint256 sharesWithdraw = _convertToSharesTotals(
    assets, withdrawShares[period], withdrawAssets[period], Math.Rounding.Floor
);                                                      // 0
withdrawAssets[period]             += assets;           // += 0
withdrawShares[period]             += sharesWithdraw;   // += 0
withdrawSharesOf[period][receiver] += sharesWithdraw;   // += 0
pendingWithdrawAssets              += assets;           // += 0
...
_update(owner, address(0), shares);                     // shares burned
```

The conversion at L1067-1074 is `floor(shares * (totalAssets + 1) / (totalSupply + 1))`, so it returns zero for any `shares < (S+1)/(A+1)`.

That threshold is the part worth pausing on. When `A >= S` it's effectively 1 wei and nobody cares. But `A < S` — share price under 1 — is not an exotic state here. It's what `payout()` produces every time it runs, because it reduces `totalAssets()` and deliberately leaves `totalSupply()` alone. That's the whole design of loss socialisation. After a payout of fraction `f`, everything under `1/(1-f)` shares gets destroyed: 2 shares after a 50% payout, 20 after 95%, a thousand after 99.9%.

So the preconditions are just "an incident has been paid out at some point." No attacker, no privileged role, no ordering trick. The victim is whoever calls `redeem()` with a small enough number.

On magnitude, I'd rather state the limit myself than have you find it. `assets == 0` requires `shares * (A+1)/(S+1) < 1`, so the immediate loss per call is under one unit of the underlying. That's why I've filed this as Low and not Medium — I'm not going to argue it's worth more than it is. What isn't bounded is the share side: my PoC destroys 10 of a user's 20 shares in one call, half their position, and those shares stop participating in any future recovery. Rewards forwarded through `VaultRewardDistributor` go to everyone else instead.

The part I find most awkward is that the call succeeds. No revert, and `WithdrawRequest` still fires with `assets = 0`, so on-chain it looks like a normal withdrawal request. `withdrawalsOf(period, user)` then returns 0. A contract redeeming programmatically — a router cleaning up dust, an automated exit — gets a successful transaction back and no indication that the position was destroyed rather than queued.

A note on the known issues, since one of them is broad enough to catch anything involving payouts. This isn't the aggregate-claim-cap issue: the `payout()` in my PoC is a single legitimate within-cap call, used only to get the share price under 1, and any other route to that state reproduces the bug identically. It has nothing to do with the first-loss buffer or the external wallet either. And it doesn't overlap the Phase-1 findings — those covered `periodAtTimestamp()` (fixed on this branch at L316-320), missing `_disableInitializers()` (present at L197-199), and a `withdrawalsOf()` view discrepancy. None of them reach `redeem()` or the burn in `_requestWithdraw`.

Fix is one line, copying what `deposit()` already does:

```diff
         (uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
             shares,
             false,
             Math.Rounding.Floor
         );
+        // A dust redeem can floor to zero assets once the share price falls below 1;
+        // reject it instead of burning shares in exchange for nothing.
+        if (assets == 0) revert InvalidAmount();
 
         uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
```

Putting it in `_requestWithdraw` instead would cover both entry points.

## References

* `redeem()` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L717-L744>
* `_requestWithdraw()` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1028>
* the guard on the sibling path — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L667-L669>

## Link to Proof of Concept

<https://gist.github.com/morpheus-csmith/624dbd2c9fa126ea92d170b395f48287>

## Proof of Concept

{% stepper %}
{% step %}

## A user deposits 20 assets and receives 20 shares

`totalAssets() == 20`, `totalSupply() == 20`, so the share price is exactly 1.
{% endstep %}

{% step %}

## Advance to the next period

Advance to the next period, so the deposit's `_traceTotalAssets` checkpoint sits at or before the capture period start. Take `captureTimestamp = currentPeriodStart()`.
{% endstep %}

{% step %}

## Call `payout()`

`PAYOUT_ROLE` calls `payout(receiver, 19, captureTimestamp)`. This is an ordinary within-cap payout — the exposure cap is `totalAssetsAt(capturePeriodStart)` which is 20, and 19 is under it.
{% endstep %}

{% step %}

## Observe the reduced share price

The vault now holds `totalAssets() == 1` against `totalSupply() == 20`. Share price is 0.05, which is the intended result of loss socialisation: `payout()` reduces assets and leaves supply alone.
{% endstep %}

{% step %}

## Redeem 10 shares

The user calls `redeem(10, user, user)`. Internally:

```solidity
assets = floor(10 * (1 + 1) / (20 + 1)) = floor(20 / 21) = 0
```

{% endstep %}

{% step %}

## Shares are burned without queuing assets

`redeem()` does not check `assets`. It passes 0 into `_requestWithdraw`, which adds 0 to `withdrawAssets[period]`, 0 to `withdrawShares[period]`, 0 to `withdrawSharesOf[period][receiver]`, 0 to `pendingWithdrawAssets`, then calls `_update(owner, address(0), shares)` — burning all 10 shares.
{% endstep %}

{% step %}

## The transaction succeeds with no recovery path

The transaction succeeds and returns 0. The user's balance drops from 20 to 10. Every withdrawal accumulator is still zero and `withdrawalsOf(period, user)` returns 0. There is no queued position, now or in any future period, and no path in the contract to recover it.

Half the user's position is gone and nothing was received in exchange.
{% endstep %}
{% endstepper %}

## Reproduction

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install

# Save the file below as test/audit_pocs.js
# Also available as a secret Gist:
#   https://gist.github.com/morpheus-csmith/624dbd2c9fa126ea92d170b395f48287
#   sha256: 64529474656a335ff4d4cbb07d3a85300f0a41ffad252cfed656094e24952716
# It contains all four PoCs from this submission set; this report's is PoC1.

npx hardhat test test/audit_pocs.js --grep "PoC1"
```

## Test

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

const tinyVault = () =>
  loadFixture(deployVault.bind(null, { decimals: 0n, initial_deposit_limit: 20n }));

describe("AUDIT PoCs", function () {
  it("PoC1: redeem() burns shares and queues nothing when the asset value floors to zero", async () => {
    const { firelight_vault, payout_signer, payout_receiver, users, utils, config } =
      await tinyVault();
    const user = users[0];

    await utils.mintAndApprove(20n, user);
    await firelight_vault.connect(user).deposit(20n, user.address);
    expect(await firelight_vault.totalAssets()).to.equal(20n);
    expect(await firelight_vault.totalSupply()).to.equal(20n);

    await time.increase(config.period_configuration_duration);
    const captureTimestamp = await firelight_vault.currentPeriodStart();
    await firelight_vault
      .connect(payout_signer)
      .payout(payout_receiver.address, 19n, captureTimestamp);

    expect(await firelight_vault.totalAssets()).to.equal(1n);
    expect(await firelight_vault.totalSupply()).to.equal(20n);

    const period = (await firelight_vault.currentPeriod()) + 1n;
    const sharesBefore = await firelight_vault.balanceOf(user.address);

    // assets = floor(10 * (1+1) / (20+1)) = 0
    await firelight_vault.connect(user).redeem(10n, user.address, user.address);

    // 10 shares destroyed
    expect(await firelight_vault.balanceOf(user.address)).to.equal(sharesBefore - 10n);

    // nothing queued in exchange, now or ever
    expect(await firelight_vault.withdrawSharesOf(period, user.address)).to.equal(0n);
    expect(await firelight_vault.withdrawAssets(period)).to.equal(0n);
    expect(await firelight_vault.pendingWithdrawAssets()).to.equal(0n);
    expect(await firelight_vault.withdrawalsOf(period, user.address)).to.equal(0n);
  });
});
```

## Result

```
  AUDIT PoCs
    ✔ PoC1: redeem() burns shares and queues nothing when the asset value floors to zero (1001ms)

  1 passing (1s)
```

Environment: Ubuntu on WSL2, Node v22.20.0, npm 10.9.3, Hardhat 2.22.19, solc 0.8.28 with the optimizer on at `runs: 800` and viaIR — the repository's own production setting for this contract.

The repository's existing suite passes on the same tree, so this runs against a known-good baseline:

```
$ npx hardhat test test/vault_payout.js

  FirelightVault payout accounting
    ✔ does not assign rounding residual to an empty next withdrawal bucket (825ms)
    ✔ caps payout to available assets while reducing both withdrawal buckets in the following period (130ms)
    ✔ caps payout to assets at the capture period start (106ms)
    ✔ reverts when payout is executed after the payout window expires (88ms)

  4 passing (1s)
```
