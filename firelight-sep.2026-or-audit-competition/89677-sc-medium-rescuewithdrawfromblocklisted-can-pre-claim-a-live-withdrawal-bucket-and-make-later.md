# 89677 sc medium rescuewithdrawfromblocklisted can pre claim a live withdrawal bucket and make later withdrawals unclaimable

**Submitted on Aug 20th 2026 at 14:10:28 UTC by @Nescient for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89677
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`claimWithdraw` can set `isWithdrawClaimed[period][account]` only after proving that `period` is closed, but `rescueWithdrawFromBlocklisted` can set the same settlement marker for any caller-supplied period without that bound. A rescue can therefore mark a live `currentPeriod() + 1` bucket as claimed; after the account is removed from the blocklist, an ordinary withdrawal can add a new entitlement to that bucket and burn the user's shares, but every later claim and rescue path rejects the entitlement because the marker was set before it existed (`contracts/core/FirelightVault.sol:788-806`, `:845-870`, `:994-1027`).

## Vulnerability Details

`isWithdrawClaimed` is defined as the record of whether an account has claimed its withdrawal for a period (`contracts/core/FirelightVaultStorage.sol:67-71`). The normal claim path preserves that meaning: `claimWithdraw` first requires `period < currentPeriod()` at `contracts/core/FirelightVault.sol:791`, rejects an existing marker at `:794`, computes and debits the entitlement at `:796-805`, and only then sets the marker at `:806`.

```solidity
// contracts/core/FirelightVault.sol:791-806
if (period >= currentPeriod()) revert InvalidPeriod();

address sender = _msgSender();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);

assets = _convertToAssetsTotals(
    withdrawSharesOf[period][sender],
    withdrawShares[period],
    withdrawAssets[period],
    Math.Rounding.Floor
);

if (assets == 0) revert NoWithdrawalAmount(period);

pendingWithdrawAssets -= assets;
isWithdrawClaimed[period][sender] = true;
```

`rescueWithdrawFromBlocklisted` accepts a caller-supplied `periods[]` array and checks the rescuer role, source blocklist status, destination status, existing entitlement, and both claim markers, but it never checks that a supplied period is closed (`contracts/core/FirelightVault.sol:845-870`). It moves the current entitlement to the recovery address and sets the source marker to `true` at `:868-870`, including when the supplied period is the live `currentPeriod() + 1` bucket.

```solidity
// contracts/core/FirelightVault.sol:861-870
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
    if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

    withdrawSharesOf[periods[i]][to] += _withdrawOf;
    withdrawSharesOf[periods[i]][from] = 0;
    isWithdrawClaimed[periods[i]][from] = true;
}
```

The ordinary withdrawal path always targets `currentPeriod() + 1` at `contracts/core/FirelightVault.sol:1008`. It adds assets, total withdrawal shares, and the receiver's withdrawal shares at `:1015-1017`, increases `pendingWithdrawAssets` at `:1019`, and burns the owner's active shares at `:1025`. It never reads `isWithdrawClaimed` before performing those mutations (`contracts/core/FirelightVault.sol:994-1027`).

```solidity
// contracts/core/FirelightVault.sol:1008-1025
uint256 period = currentPeriod() + 1;
uint256 sharesWithdraw = _convertToSharesTotals(
    assets,
    withdrawShares[period],
    withdrawAssets[period],
    Math.Rounding.Floor
);
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;

pendingWithdrawAssets += assets;
// ...
_update(owner, address(0), shares);
```

The resulting state has a positive `withdrawSharesOf[period][account]` and `isWithdrawClaimed[period][account] == true`. `claimWithdraw` always reverts on that marker at `contracts/core/FirelightVault.sol:794`; re-rescue always reverts on the same marker at `:864`; and `rescueSharesFromBlocklisted` reads only the active `balanceOf(from)` at `:827-830`, after `_requestWithdraw` has burned those shares at `:1025`. Both production writes to `isWithdrawClaimed` set it to `true` at `:806` and `:870`; no production path writes it back to `false`, and `FirelightVault` has no sweep or recovery function for this state (`contracts/core/FirelightVault.sol:788-876`).

The public `isWithdrawClaimed` getter exposes the raw marker, but no preview or maximum-withdrawal view reports that the next ordinary withdrawal will target a pre-marked bucket. `withdrawalsOf` returns the numeric entitlement "whether claimed or not" and does not consult the marker (`contracts/core/FirelightVault.sol:460-473`), while `_requestWithdraw` derives the target internally and does not consult the marker (`contracts/core/FirelightVault.sol:1008-1017`). The contract therefore provides no preflight that identifies an ordinary withdrawal as destined for an unusable bucket.

The state transition is:

1. In period `P`, the account requests withdrawal `A` into bucket `P + 1` through `_requestWithdraw` (`contracts/core/FirelightVault.sol:994-1027`).
2. While `P + 1` is still live, the account is blocklisted and an authorized rescuer moves both `A` and all remaining active shares to a recovery address; the pending-withdrawal rescue sets the source marker for `P + 1` to `true` (`contracts/core/FirelightVault.sol:818-875`).
3. The authorized blocklister removes the zero-balance account from the blocklist during period `P`, after every old position has been rescued (`contracts/core/FirelightVault.sol:556-563`).
4. The cleared account makes a fresh deposit `B` and then requests withdrawal of `B`; `_requestWithdraw` adds this new principal to the same `P + 1` bucket and burns the newly minted shares without reading the old marker (`contracts/core/FirelightVault.sol:994-1025`).
5. After `P + 1` closes, `claimWithdraw(P + 1)` and re-rescue both revert on the old marker, while active-share rescue has no shares to move (`contracts/core/FirelightVault.sol:794`, `:827-830`, `:864`).

### Duplicate check

I checked the Macro A-1 audit report (0xMacro, 29 July 2026), the published competition report index at <https://reports.immunefi.com/firelight>, the program's disclosed known issues, the program's scope and asset page, and the project's test suite. The Macro report describes the compliance rescue model but does not address the period scope of the `isWithdrawClaimed` marker, and the scope page documents the vault's delayed-withdrawal behavior without covering it either.

Two published competition reports touch adjacent surface without covering this state transition: #59027 (`withdrawalsOf` does not account for already-withdrawn funds) is a view-accuracy issue, and #59288 (repeated array access in the `rescueWithdrawFromBlocklisted` loop) is a gas issue. Neither concerns a marker written to a live bucket or a later request burning shares into it. The existing rescue tests exercise active-share rescue separately at `test/rescue.js:23-79` and pending-withdrawal rescue at `test/rescue.js:81-184`; they do not add a new request to a rescued live bucket.

## Why this is not operator error

Every privileged action in the sequence is within its intended role and purpose. `BLOCKLIST_ROLE` adds the account to the blocklist in response to a compliance flag (`contracts/core/FirelightVault.sol:546-554`), `RESCUER_ROLE` moves both the queued withdrawal and all active shares to a non-blocklisted recovery address (`:813-875`), and `BLOCKLIST_ROLE` later removes the zero-balance account after clearance (`:556-563`). No role misbehaves, exceeds its authority, or receives a gain; the loss is triggered only after the cleared user makes a fresh deposit and submits an ordinary withdrawal through `_requestWithdraw` (`contracts/core/FirelightVault.sol:994-1027`).

**Counter: "The rescuer must also rescue the account's active shares."** The PoC does rescue all active shares as well as the pending withdrawal before the account is unblocked, leaving the account with no old position. The counter therefore does not explain the loss of the entirely fresh principal deposited after clearance; independently, the contract treats `rescueSharesFromBlocklisted` and `rescueWithdrawFromBlocklisted` as separate external functions and the project's suite tests them independently (`contracts/core/FirelightVault.sol:813-876`, `test/rescue.js:23-79`, `:81-184`).

**Counter: "The account must remain blocklisted until the current period advances."** No contract check, NatSpec statement, or supplied documentation states that unblocklisting within a period is unsafe. The shipped `removeFromBlocklist` function has only role and blocklist-state guards and no period condition (`contracts/core/FirelightVault.sol:556-563`); if safe operation depends on an undocumented timing rule that the code does not enforce, that gap is the finding.

No withdrawal preview identifies the poisoned target bucket before the ordinary user burns shares. `withdrawalsOf` reports the entitlement even after the marker makes it unclaimable (`contracts/core/FirelightVault.sol:460-473`), and `_requestWithdraw` does not read the marker (`:994-1027`).

## Reachability

The sequence requires an account with a live `currentPeriod() + 1` entitlement to be blocklisted, have both its pending entitlement and active shares rescued, be removed from the blocklist before the period changes, and then deposit and request withdrawal of fresh principal in the same period. No attacker can force the blocklisting, rescue, or clearance decisions.

`removeFromBlocklist` is a shipped role-gated function that immediately clears `isBlocklisted[account]` without a period advance (`contracts/core/FirelightVault.sol:556-563`). The project's own suite calls `removeFromBlocklist` at `test/blocklist.js:143` and, in the next test beginning at `:149`, asserts that the account transacts immediately at `:152`; the first later `time.increase` is only at `:158`. The sequence is reachable through documented, tested functionality, at low frequency.

## Impact Details

**Submitted severity: Medium.**\
**Impact: High. Likelihood: Low.**\
**Selected in-scope impact: Griefing (no profit motive, but damage to users/protocol).**

In the PoC, the user initially deposits 6,000 units and requests 2,000 units. The authorized rescue moves that 2,000-unit pending entitlement and all remaining 4,000 active shares to the recovery address, leaving the user with no old vault position before clearance (`contracts/core/FirelightVault.sol:818-875`). After clearance, the user deposits a fresh 4,000 units and requests withdrawal of the full fresh deposit into the marked live bucket; the new shares are burned at `contracts/core/FirelightVault.sol:1025`, the claim is rejected at `:794`, re-rescue is rejected at `:864`, and active-share rescue finds no shares at `:827-830`. The recovery account claims the correctly rescued 2,000 units, while the fresh 4,000 units remain recorded in `pendingWithdrawAssets` and held by the vault with no claim or rescue path.

The impact is high because the affected user's later withdrawal principal becomes permanently unclaimable. The likelihood is low because no attacker can force the required blocklisting, rescue, clearance, and same-period withdrawal sequence. Combining the high impact with the low likelihood, the submitted severity is Medium.

## Recommendation

Apply the same closed-period invariant to `rescueWithdrawFromBlocklisted` that `claimWithdraw` already enforces. Check every supplied period before moving its entitlement or setting its marker:

```solidity
uint256 period = periods[i];
if (period >= currentPeriod()) revert InvalidPeriod();
```

Add a defense before `_requestWithdraw` changes accounting or burns shares. After computing `period = currentPeriod() + 1`, revert if `isWithdrawClaimed[period][receiver]` is already true:

```solidity
uint256 period = currentPeriod() + 1;
if (isWithdrawClaimed[period][receiver]) revert AlreadyClaimedPeriod(period);
```

The first check prevents rescue from assigning a settlement marker to a live or future bucket. The second check ensures that an inconsistent marker cannot cause an ordinary withdrawal to burn shares before failing.

## Proof of Concept

Save the following file as `test/FirelightVault/rescueMarkerLiveBucketPoisoning.js` and run:

```bash
npx hardhat test test/FirelightVault/rescueMarkerLiveBucketPoisoning.js
npx hardhat test
```

The PoC uses these harness facilities:

1. `loadFixture` creates a deterministic fresh deployment and does not alter transaction ordering.
2. The audited upgradeable `FirelightVault` is deployed through `upgrades.deployProxy`; only the external ERC20 asset is mocked.
3. `BLOCKLIST_ROLE` and `RESCUER_ROLE` are granted by the real `DEFAULT_ADMIN_ROLE` holder through `grantRole`.
4. Normal automining remains enabled; the test uses no `evm_setNextBlockTimestamp`, timestamp-setting RPC, `impersonateAccount`, `setStorageAt`, or fabricated role caller.
5. Each `time.increase` is commented and is used only to cross a withdrawal period boundary after the vulnerable state already exists.

The single test proves that all old active shares and pending withdrawals are rescued before unblocklisting, the user then deposits fresh principal, and an ordinary withdrawal burns the new shares into the pre-marked bucket. It also proves that the correct recovery address can claim the entitlement that was actually rescued, re-blocklisting and re-rescue revert on the source marker, `rescueSharesFromBlocklisted` cannot reach the newly burned shares, and the user's claim still reverts after additional period boundaries.

Targeted result:

```
Live withdrawal bucket rescue marker
  ✔ permanently locks a withdrawal requested after the live bucket was rescued

1 passing
```

Full-suite result on the stated commit:

```
458 passing (7s)
```

Full PoC file:

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { expect } = require("chai");
const { ethers, upgrades } = require("hardhat");

const ASSET_DECIMALS = 6;
const PERIOD_DURATION = 7 * 24 * 60 * 60;
const INITIAL_WITHDRAWAL = ethers.parseUnits("2000", ASSET_DECIMALS);
const RESCUED_ACTIVE_ASSETS = ethers.parseUnits("4000", ASSET_DECIMALS);
const INITIAL_DEPOSIT = INITIAL_WITHDRAWAL + RESCUED_ACTIVE_ASSETS;
const FRESH_PRINCIPAL = ethers.parseUnits("4000", ASSET_DECIMALS);
const DEPOSIT_LIMIT = ethers.parseUnits("1000000", ASSET_DECIMALS);

async function deployRealVaultFixture() {
  const [deployer, victim, blocklistOperator, rescueOperator, recovery] =
    await ethers.getSigners();

  // Only the external asset is mocked. The vault is the audited upgradeable implementation.
  const MockERC20 = await ethers.getContractFactory("MockERC20");
  const asset = await MockERC20.deploy("Mock FXRP", "mFXRP", ASSET_DECIMALS);
  await asset.waitForDeployment();

  const FirelightVault = await ethers.getContractFactory("FirelightVault");
  const initParams = ethers.AbiCoder.defaultAbiCoder().encode(
    ["address", "address", "address", "address", "address", "address", "uint256", "uint48"],
    [
      deployer.address,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      ethers.ZeroAddress,
      DEPOSIT_LIMIT,
      PERIOD_DURATION,
    ],
  );
  const vault = await upgrades.deployProxy(FirelightVault, [
    asset.target,
    "Staked FXRP",
    "stFXRP",
    initParams,
  ]);
  await vault.waitForDeployment();

  // Roles are granted by the real DEFAULT_ADMIN_ROLE holder through AccessControl.
  await vault
    .connect(deployer)
    .grantRole(await vault.BLOCKLIST_ROLE(), blocklistOperator.address);
  await vault
    .connect(deployer)
    .grantRole(await vault.RESCUER_ROLE(), rescueOperator.address);

  await asset.mint(victim.address, INITIAL_DEPOSIT);
  await asset.connect(victim).approve(vault.target, INITIAL_DEPOSIT);

  return { asset, vault, victim, blocklistOperator, rescueOperator, recovery };
}

describe("Live withdrawal bucket rescue marker", function () {
  it("permanently locks a withdrawal requested after the live bucket was rescued", async function () {
    const { asset, vault, victim, blocklistOperator, rescueOperator, recovery } =
      await loadFixture(deployRealVaultFixture);

    await vault.connect(victim).deposit(INITIAL_DEPOSIT, victim.address);

    const startPeriod = await vault.currentPeriod();
    const livePeriod = startPeriod + 1n;

    // Create a legitimate pending withdrawal in the still-open P+1 bucket.
    await vault
      .connect(victim)
      .withdraw(INITIAL_WITHDRAWAL, victim.address, victim.address);
    expect(await vault.withdrawalsOf(livePeriod, victim.address)).to.equal(
      INITIAL_WITHDRAWAL,
    );

    // Correct compliance flow: blocklist and rescue every old position.
    await vault.connect(blocklistOperator).addToBlocklist(victim.address);
    await vault
      .connect(rescueOperator)
      .rescueWithdrawFromBlocklisted(victim.address, recovery.address, [livePeriod]);
    expect(await vault.isWithdrawClaimed(livePeriod, victim.address)).to.equal(true);
    expect(await vault.withdrawalsOf(livePeriod, recovery.address)).to.equal(
      INITIAL_WITHDRAWAL,
    );
    await vault
      .connect(rescueOperator)
      .rescueSharesFromBlocklisted(victim.address, recovery.address);
    expect(await vault.balanceOf(victim.address)).to.equal(0n);
    expect(await vault.balanceOf(recovery.address)).to.equal(RESCUED_ACTIVE_ASSETS);

    // The compliance flag is cleared only after all old positions have been rescued.
    await vault.connect(blocklistOperator).removeFromBlocklist(victim.address);

    // The cleared account deposits entirely fresh principal.
    await asset.mint(victim.address, FRESH_PRINCIPAL);
    await asset.connect(victim).approve(vault.target, FRESH_PRINCIPAL);
    await vault.connect(victim).deposit(FRESH_PRINCIPAL, victim.address);
    expect(await vault.balanceOf(victim.address)).to.equal(FRESH_PRINCIPAL);

    // With no legacy position left, an ordinary withdrawal reuses the marked live bucket.
    await vault
      .connect(victim)
      .withdraw(FRESH_PRINCIPAL, victim.address, victim.address);
    expect(await vault.balanceOf(victim.address)).to.equal(0n);
    expect(await vault.isWithdrawClaimed(livePeriod, victim.address)).to.equal(true);
    expect(await vault.withdrawalsOf(livePeriod, victim.address)).to.equal(
      FRESH_PRINCIPAL,
    );

    // Neither rescue path can recover the entitlement backed by the burned shares.
    await vault.connect(blocklistOperator).addToBlocklist(victim.address);
    await expect(
      vault
        .connect(rescueOperator)
        .rescueWithdrawFromBlocklisted(victim.address, recovery.address, [livePeriod]),
    )
      .to.be.revertedWithCustomError(vault, "AlreadyClaimedPeriod")
      .withArgs(livePeriod);
    await expect(
      vault
        .connect(rescueOperator)
        .rescueSharesFromBlocklisted(victim.address, recovery.address),
    ).to.be.revertedWithCustomError(vault, "InsufficientShares");
    await vault.connect(blocklistOperator).removeFromBlocklist(victim.address);

    // Advance only to cross two period boundaries and make the P+1 bucket claimable.
    await time.increase(PERIOD_DURATION * 2);

    // The correctly rescued entitlement remains claimable by the recovery account.
    await expect(vault.connect(recovery).claimWithdraw(livePeriod)).to.changeTokenBalances(
      asset,
      [vault, recovery],
      [-INITIAL_WITHDRAWAL, INITIAL_WITHDRAWAL],
    );

    // The later ordinary withdrawal is rejected by the marker written before it existed.
    await expect(vault.connect(victim).claimWithdraw(livePeriod))
      .to.be.revertedWithCustomError(vault, "AlreadyClaimedPeriod")
      .withArgs(livePeriod);

    // Advance only to cross one more period boundary and prove the lock persists.
    await time.increase(PERIOD_DURATION);
    await expect(vault.connect(victim).claimWithdraw(livePeriod))
      .to.be.revertedWithCustomError(vault, "AlreadyClaimedPeriod")
      .withArgs(livePeriod);

    expect(await vault.withdrawalsOf(livePeriod, victim.address)).to.equal(
      FRESH_PRINCIPAL,
    );
    expect(await vault.pendingWithdrawAssets()).to.equal(FRESH_PRINCIPAL);
    expect(await vault.totalAssets()).to.equal(RESCUED_ACTIVE_ASSETS);
    expect(await asset.balanceOf(vault.target)).to.equal(
      RESCUED_ACTIVE_ASSETS + FRESH_PRINCIPAL,
    );
  });
});
```
