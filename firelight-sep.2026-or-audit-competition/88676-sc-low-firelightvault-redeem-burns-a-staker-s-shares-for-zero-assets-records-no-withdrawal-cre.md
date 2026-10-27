# 88676 sc low firelightvault redeem burns a staker s shares for zero assets records no withdrawal credit and returns success

**Submitted on Aug 14th 2026 at 13:35:30 UTC by @whitehorse for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88676
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Severity

Low. In scope impact: "Contract fails to deliver promised returns, but doesn't lose value."

## Brief/Intro

`redeem` validates the caller supplied `shares`, converts it to `assets` with floor rounding, and never checks what the conversion returned. Once the vault has paid a claim and the share price falls below 1, redeeming a small share count burns the shares, adds zero to every withdrawal ledger entry, and returns success. The holder's balance is destroyed and nothing is recorded in exchange for it. The project already ruled this a bug on the entry side: 0xMacro's Firelight A-1 audit reported L-1, that `deposit` could mint zero shares while still transferring assets, because the deposit amount was guarded and the resulting share count was not. It was fixed in commit `6806f35`, "fix: \[L-1] prevent zero-share deposits", whose diff to `FirelightVault.sol` is the three lines now at `:667` to `:669` and touches no other function. The exit path kept the same shape and is still unguarded at the in-scope commit `42f9ea5e`.

## Vulnerability Details

```solidity
// FirelightVault.sol, in redeem() at :717
@>      if (shares == 0) revert InvalidAmount();                                 // :731 only the input is checked

        (uint256 assets, uint256 _totalSupply, uint256 _totalAssets) = _previewTotals(
            shares,
            false,
            Math.Rounding.Floor                                                  // :736 the derived assets, floored
        );

@>      uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner); // :739 reached with assets == 0
```

The conversion floors, at `_convertToAssetsTotals`:

```solidity
return shares.mulDiv(totAssets + 1, totSupply + 10 ** _decimalsOffset(), rounding);   // :1073
```

`_decimalsOffset()` is not overridden anywhere in the tree and has no setter, so the cushion is fixed at `10 ** 0 = 1` and the zero output condition is exact, with `A` the total assets and `S` the total supply:

```
assets == 0   <=>   shares * (A + 1) < S + 1   <=>   shares < (S + 1) / (A + 1)
```

While `A >= S` the right hand side is at most 1, so only `shares == 0` qualifies and `:731` already covers it. Once `A < S` the band widens to every share count strictly below `(S + 1) / (A + 1)`. Nothing rechecks the value:

```solidity
        // :1009 sharesWithdraw = _convertToSharesTotals(assets, ...), zero whenever assets is zero
        withdrawAssets[period] += assets;                     // :1015 += 0
        withdrawShares[period] += sharesWithdraw;             // :1016 += 0
        withdrawSharesOf[period][receiver] += sharesWithdraw; // :1017 += 0

        pendingWithdrawAssets += assets;                      // :1019 += 0

        // :1021 to :1023, the allowance spend when sender is not owner, omitted here

@>      _update(owner, address(0), shares);                   // :1025 burns the full shares unconditionally
```

In a later period `claimWithdraw` reads `withdrawSharesOf[period][account]`, finds zero, and reverts `NoWithdrawalAmount(period)` at `:803`, with the shares already gone.

`A < S` is the state the vault is built to enter. `payout()` at `:892` transfers the approved claim out at `:962` and pushes the reduced total at `:952` without touching `totalSupply`, and contains no `_mint`, `_burn` or `_update`. Absorbing claim losses into the share price is the product's core function, so a vault that has paid a claim sits at `A < S` until fresh yield restores the gap. `payout` is new in this phase. The legacy predeposit vault has no such function, and in `FirelightVaultPredeposits.sol` `totalAssets()` at `:382` nets out `pendingWithdrawAssets`, so the only site that lowers it is the withdrawal request at `:811`, which burns the matching shares at `:817`; the `claimWithdraw` transfer at `:708` is neutral, since `:705` decrements `pendingWithdrawAssets` by the same amount. Assets never fell without shares, so the price could not drop below 1 and the band this defect needs did not exist.

`redeem` is the only entry point that rounds a derived quantity down and does not check it. `deposit` also floors, at `:665`, and rejects the zero at `:669`. `withdraw` and `mint` round up, at `:771` and `:701`, so their derived quantity cannot reach zero. The two sides are the same trade reversed: in `deposit` assets enter and no shares come back, in `redeem` shares are destroyed and no assets come back, and either way the value accrues to the other holders. `redeem` additionally splits the failure across two transactions, since the burn settles immediately while the empty claim reverts only a period later. `redeem` at `:717` carries `whenNotPaused`, three `notBlocklisted` modifiers and `nonReentrant`, and no role, allowlist, price test or timing gate sits between the external call and the burn at `:1025`.

## Impact Details

After a claim payout, any staker who calls `redeem` with a share count inside the band loses those shares outright and receives nothing. The call does not revert. It emits `WithdrawRequest` with `assets` zero and leaves a withdrawal request carrying no claim.

On the live vault, `totalAssets()` 57,360,574.888550 FXRP against `totalSupply()` 57,356,448.484049 stXRP, the band opens once a claim exceeds the current surplus of 4,126.404501 FXRP and widens with the claim: 1 share unit at a claim of 10 percent of the vault, 99 share units at 99 percent. The assets the burned shares were entitled to stay in the vault and accrue pro rata to the remaining holders, and that entitlement is strictly below one raw asset unit per call, below 0.000001 FXRP, because the floor identity that zeroes the output also bounds it. The vault's asset balance is unchanged by the burn.

## Recommended fix

Give the exit the guard the entry already has, immediately after the conversion at `:737`:

```solidity
if (assets == 0) revert InvalidAmount();
```

## Proof of Concept

`poc_zero_asset_redeem.js` runs on the repository's own Hardhat suite at commit `42f9ea5e` with no contract modified. Three tests pass at live mainnet parameters: test 1 is the negative control, showing the defect does not fire at the live price, test 2 is the defect after a claim of 10 percent of the vault, test 3 pins the band boundary and the value ceiling.

```
price above 1: deposit(1) reverts, redeem(1) credits 1 asset unit
price below 1 (A=51624517399695 S=57356448484049): redeem(1) returned 0,
  supply fell by 1, ledger credit 0, claimWithdraw reverted
burnable band = 99 share units, redeem(99) returned 0, redeem(100) returned 1
entitlement destroyed = 0.990071 raw asset units, strictly below 1
3 passing
```

Adding the guard above after `:737` turns tests 2 and 3 into `InvalidAmount` reverts and both zero credit assertions fail.

## References

* `contracts/core/FirelightVault.sol:717` to `:739`, `redeem`, and `:731`, the input only check
* `contracts/core/FirelightVault.sol:1064`, `:1067`, `:1073`, the floor conversion and the never overridden `_decimalsOffset`
* `contracts/core/FirelightVault.sol:1015` to `:1019`, `:1025`, the zero ledger writes and the unconditional burn
* `contracts/core/FirelightVault.sol:803`, the later `NoWithdrawalAmount` revert
* `contracts/core/FirelightVault.sol:665`, `:667` to `:669`, the guarded floor conversion in `deposit`
* `contracts/core/FirelightVault.sol:892`, `:952`, `:962`, `payout` lowering assets without touching supply
* 0xMacro Firelight A-1, finding L-1, "prevent zero-share deposits", commit `6806f35`

## Proof of Concept

```javascript
// PoC for FirelightVault.redeem: shares are burned for zero assets and no withdrawal credit.
//
// HOW TO RUN
//   git clone https://github.com/immunefi-team/audit-comp-firelight.git
//   cd audit-comp-firelight && git checkout v1_audit_ready
//   npm install
//   cp <this file> test/poc_zero_asset_redeem.js
//   npx hardhat test test/poc_zero_asset_redeem.js
//
// Expected output: 3 passing.
//
// NO MODIFICATION of any contract in contracts/ is required or made. The test uses only the
// repository's own fixture (test/setup/fixtures.js) and the vault's public API.
//
// MAINNET PARAMETERS. Every number below is read from the live Firelight vault proxy
// 0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3 on Flare (chain id 14) at block 67,320,396,
// via https://flare-api.flare.network/ext/C/rpc:
//   totalAssets()  = 57,360,574.888550 FXRP   (eth_call 0x01e1d114)
//   totalSupply()  = 57,356,448.484049 stXRP  (eth_call 0x18160ddd)
//   depositLimit   = 60,168,000.000000 FXRP   (eth_call 0xecf70858)
//   FXRP decimals  = 6
//   period duration = 86,400 s
// The share price today is (A+1)/(S+1) = 1.000071943, i.e. above 1. Test 1 runs at exactly that
// state and shows the defect does NOT fire there. Tests 2 and 3 run after an incident payout,
// which is the vault's own loss mechanism and the only writer that moves the price below 1.
//
// ROLE USE. PAYOUT_ROLE is used only to execute a normal, correctly sized, allowlisted claim
// payment through the contract's own payout() function, exactly as the repository's own
// test/vault_payout.js does. No role misbehaves, no privileged call touches the redeem path,
// and the account that loses shares is an ordinary unprivileged staker.
//
// ISOLATION CHECK, performed in a separate copy and reverted there:
//   adding `if (assets == 0) revert InvalidAmount();` to FirelightVault.redeem immediately after
//   the _previewTotals call at line 733-737 (the guard deposit() already has at line 669) turns
//   tests 2 and 3 into reverts, and both zero-credit assertions fail. The defect is that single
//   missing check and nothing else.

const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

const DECIMALS = 6;
const LIVE_TOTAL_ASSETS = 57360574888550n; // 57,360,574.888550 FXRP
const LIVE_TOTAL_SUPPLY = 57356448484049n; // 57,356,448.484049 stXRP
const LIVE_DEPOSIT_LIMIT = 60168000000000n; // 60,168,000.000000 FXRP
const LIVE_PERIOD_DURATION = 86400; // seconds

// Rebuilds the live vault state: one staker holding the whole supply, plus the
// 4,126.404501 FXRP by which the live vault's assets already exceed its supply.
async function liveVault() {
  const ctx = await loadFixture(
    deployVault.bind(null, {
      decimals: DECIMALS,
      initial_deposit_limit: LIVE_DEPOSIT_LIMIT.toString(),
      period_configuration_duration: LIVE_PERIOD_DURATION,
    }),
  );

  const staker = ctx.users[0];
  const newcomer = ctx.users[1];
  const donor = ctx.users[2];

  await ctx.utils.mintAndApprove(LIVE_TOTAL_SUPPLY, staker);
  await ctx.firelight_vault.connect(staker).deposit(LIVE_TOTAL_SUPPLY, staker.address);

  const excess = LIVE_TOTAL_ASSETS - LIVE_TOTAL_SUPPLY;
  await ctx.utils.mintAndApprove(excess, donor);
  await ctx.token_contract.connect(donor).transfer(ctx.firelight_vault.target, excess);

  expect(await ctx.firelight_vault.totalAssets()).to.equal(LIVE_TOTAL_ASSETS);
  expect(await ctx.firelight_vault.totalSupply()).to.equal(LIVE_TOTAL_SUPPLY);

  // Move past the deposit period so the payout window has a capture-period checkpoint.
  await time.increase(LIVE_PERIOD_DURATION);

  return { ...ctx, staker, newcomer, donor };
}

// One approved incident claim, paid through the vault's own payout path.
async function payClaim(ctx, amount) {
  const captureTimestamp = await ctx.firelight_vault.currentPeriodStart();
  await ctx.firelight_vault
    .connect(ctx.payout_signer)
    .payout(ctx.payout_receiver.address, amount, captureTimestamp);
  return captureTimestamp;
}

describe("FirelightVault.redeem burns shares for zero assets", function () {
  it("live state, price above 1: the deposit side rejects dust, the redeem side credits it", async () => {
    const ctx = await liveVault();
    const vault = ctx.firelight_vault;

    // The entry point refuses to take assets for zero shares (FirelightVault.sol:669).
    await ctx.utils.mintAndApprove(1n, ctx.newcomer);
    await expect(
      vault.connect(ctx.newcomer).deposit(1n, ctx.newcomer.address),
    ).to.be.revertedWithCustomError(vault, "InvalidAmount");

    // At today's price the exit point is safe: 1 share unit converts to 1 asset unit.
    const assets = await vault
      .connect(ctx.staker)
      .redeem.staticCall(1n, ctx.staker.address, ctx.staker.address);
    expect(assets).to.equal(1n);

    const pendingBefore = await vault.pendingWithdrawAssets();
    await vault.connect(ctx.staker).redeem(1n, ctx.staker.address, ctx.staker.address);
    expect(await vault.pendingWithdrawAssets()).to.equal(pendingBefore + 1n);

    console.log("    price above 1: deposit(1) reverts, redeem(1) credits 1 asset unit");
  });

  it("after a 10 percent claim payout: redeem(1) burns the share, credits nothing, and the claim reverts", async () => {
    const ctx = await liveVault();
    const vault = ctx.firelight_vault;
    const staker = ctx.staker;

    const claim = LIVE_TOTAL_ASSETS / 10n; // 5,736,057.488855 FXRP
    await payClaim(ctx, claim);

    const A = await vault.totalAssets();
    const S = await vault.totalSupply();
    expect(A).to.equal(LIVE_TOTAL_ASSETS - claim);
    expect(S).to.equal(LIVE_TOTAL_SUPPLY);
    expect(A).to.be.lt(S); // share price is now below 1

    const period = await vault.currentPeriod();
    const requestPeriod = period + 1n;

    const balanceBefore = await vault.balanceOf(staker.address);
    const supplyBefore = await vault.totalSupply();
    const pendingBefore = await vault.pendingWithdrawAssets();
    const bucketBefore = await vault.withdrawAssets(requestPeriod);

    // redeem() reports the assets it will deliver. It reports zero and does not revert.
    const assets = await vault
      .connect(staker)
      .redeem.staticCall(1n, staker.address, staker.address);
    expect(assets).to.equal(0n);

    await expect(vault.connect(staker).redeem(1n, staker.address, staker.address))
      .to.emit(vault, "WithdrawRequest")
      .withArgs(staker.address, staker.address, staker.address, requestPeriod, 0n, 1n);

    // The share is gone from the holder and from the supply.
    expect(await vault.balanceOf(staker.address)).to.equal(balanceBefore - 1n);
    expect(await vault.totalSupply()).to.equal(supplyBefore - 1n);

    // Nothing was written to the withdrawal ledger in exchange.
    expect(await vault.withdrawAssets(requestPeriod)).to.equal(bucketBefore);
    expect(await vault.withdrawSharesOf(requestPeriod, staker.address)).to.equal(0n);
    expect(await vault.pendingWithdrawAssets()).to.equal(pendingBefore);

    // The holder cannot recover anything later either.
    await time.increase(2 * LIVE_PERIOD_DURATION);
    await expect(vault.connect(staker).claimWithdraw(requestPeriod))
      .to.be.revertedWithCustomError(vault, "NoWithdrawalAmount")
      .withArgs(requestPeriod);

    console.log(
      `    price below 1 (A=${A} S=${S}): redeem(1) returned 0, supply fell by 1, ledger credit 0, claimWithdraw reverted`,
    );
  });

  it("severe claim: the whole burnable band is destroyed in one call and the boundary is exact", async () => {
    const ctx = await liveVault();
    const vault = ctx.firelight_vault;
    const staker = ctx.staker;

    // A claim that consumes 99 percent of the vault. Both per-call caps in payout() are
    // respected: the capture-period exposure cap is the full deposit checkpoint.
    const claim = (LIVE_TOTAL_ASSETS * 99n) / 100n;
    await payClaim(ctx, claim);

    const A = await vault.totalAssets();
    const S = await vault.totalSupply();
    expect(A).to.be.lt(S);

    // Every share count strictly below (S+1)/(A+1) floors to zero assets.
    const band = (S + 1n) / (A + 1n); // 99 with these numbers
    expect(band).to.equal(99n);

    const zeroOut = await vault
      .connect(staker)
      .redeem.staticCall(band, staker.address, staker.address);
    expect(zeroOut).to.equal(0n);

    const firstPaying = await vault
      .connect(staker)
      .redeem.staticCall(band + 1n, staker.address, staker.address);
    expect(firstPaying).to.equal(1n);

    const supplyBefore = await vault.totalSupply();
    const pendingBefore = await vault.pendingWithdrawAssets();
    await vault.connect(staker).redeem(band, staker.address, staker.address);
    expect(await vault.totalSupply()).to.equal(supplyBefore - band);
    expect(await vault.pendingWithdrawAssets()).to.equal(pendingBefore);

    // The entitlement destroyed by that call, in raw asset units, stays below one unit.
    // This is the exact ceiling of the defect and it holds for every A, S and share count.
    const destroyed = (band * (A + 1n)) / (S + 1n);
    expect(destroyed).to.equal(0n);
    const destroyedTimes1e6 = (band * (A + 1n) * 1000000n) / (S + 1n);
    expect(destroyedTimes1e6).to.be.lt(1000000n);

    console.log(
      `    burnable band = ${band} share units, redeem(${band}) returned 0, redeem(${band + 1n}) returned 1`,
    );
    console.log(
      `    entitlement destroyed = 0.${destroyedTimes1e6.toString().padStart(6, "0")} raw asset units, strictly below 1`,
    );
  });
});
```
