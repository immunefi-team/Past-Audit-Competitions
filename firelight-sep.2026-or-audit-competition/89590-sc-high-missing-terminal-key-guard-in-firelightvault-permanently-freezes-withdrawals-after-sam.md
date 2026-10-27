# 89590 sc high missing terminal key guard in firelightvault permanently freezes withdrawals after same period rescue and restoration

**Submitted on Aug 20th 2026 at 01:47:08 UTC by @Ailenia for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89590
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`FirelightVault.rescueWithdrawFromBlocklisted()` can mark a source address as already claimed for a future period while that period's withdrawal bucket is still accepting requests. If the address is later removed from the blocklist before rollover, `withdraw()` and `redeem()` accept a new request into the same terminal `(period, receiver)` key, burn the owner's shares, and reserve the assets. Every normal claim and rescue path then rejects the positive entitlement with `AlreadyClaimedPeriod`.

The recovery requires an administrative implementation upgrade or storage migration.

## Vulnerability Details

The vulnerable asset is `contracts/core/FirelightVault.sol` at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`.

Withdrawal requests use `currentPeriod() + 1`. [`_requestWithdraw()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1027) appends the receiver's new withdrawal shares and assets to that bucket, increases `pendingWithdrawAssets`, and burns the owner's live vault shares. It does not check `isWithdrawClaimed[period][receiver]` before writing:

```solidity
uint256 period = currentPeriod() + 1;
withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
_update(owner, address(0), shares);
```

[`rescueWithdrawFromBlocklisted()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L845-L876) accepts any nonempty withdrawal period, including the still-open `currentPeriod() + 1` bucket. It moves the source's shares to a clean address, zeros the source balance, and marks the source key terminal:

```solidity
withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;
```

[`removeFromBlocklist()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L556-L563) restores ordinary account operations but clears only `isBlocklisted[account]`. The period-specific terminal flag persists.

If restoration occurs before rollover, a normal self-directed request selects the same numeric period and succeeds despite the stale terminal flag.

Once the period is claimable, [`claimWithdraw()`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L788-L811) checks the flag before reading the new positive entitlement and reverts `AlreadyClaimedPeriod`.

Re-blocklisting cannot recover the assets because `rescueWithdrawFromBlocklisted()` checks the same flag before moving shares. No normal scoped path clears the flag or moves the terminal source entitlement.

The sequence for the bug to happen is:

1. A user makes a partial withdrawal request into future period `p`, leaving part of the position as live vault shares.
2. Authorized compliance operations blocklist the user and completely preserve the position by rescuing both the pending withdrawal and the remaining live shares to a clean replacement address. Rescuing the pending withdrawal marks `(p,user)` claimed.
3. The authorized blocklist role restores the original address before rollover. Removal is an implemented lifecycle action and the repository tests verify that it restores ordinary operations, but the frequency of same-period restoration after rescue is unknown.
4. The user reconsolidates the rescued live shares into the restored address through an ordinary ERC20 transfer, then makes the usual self-directed `redeem()` or `withdraw()` call. This doesn't need new deposit or privileged call. The request succeeds, burns the shares, and records the new entitlement under the terminal key.
5. After rollover, both direct claim and authorized re-rescue reject that entitlement.

The privileged calls must happen before for this mechanism, but their timing matters. New requests are created under `currentPeriod() + 1`, while an unclaimed request remains stored after that period stops being future-facing.

So, the collision requires compliance to rescue the still-open future-period request and restore the source before rollover. Waiting until the period advances breaks the chain, which I show with the negative control.

Only `claimWithdraw()` and `rescueWithdrawFromBlocklisted()` ever write `isWithdrawClaimed`, both to `true` and never to `false`, and `claimWithdraw()` requires `period < currentPeriod()`. Because `_addPeriodConfiguration()` keeps `startingPeriod` continuous and strictly increasing across configuration changes, a claimed period can never become writable again, so no ordinary claim can substitute for the rescue.

This sequence does not require an attacker to access a privileged address. Rescue performs its documented preservation function for both forms of the user's position, removal from the blocklist is an implemented role action, and the principal loss is created afterward by ordinary transfer and vault entrypoints accepting an invalid terminal key. The affected staker triggers the terminal request while resuming normal use of the restored address.

### Separation from public prior fixes

Sentora M-02 reached a similar terminal symptom when changing a mutable period duration caused period numbers to move backward. The commit [`720d67d`](https://github.com/firelight-protocol/firelight-core/commit/720d67da9e83a18140bb96a7b95d5b3e250d1145) fixed that route with append-only period configurations and continuous `startingPeriod` values. My PoC uses one unchanged configuration and never rewinds or aliases a period, so that fix is active but does not prevent this rescue-created terminal key.

The commit [`a2def47`](https://github.com/firelight-protocol/firelight-core/commit/a2def47716389330c0ebc1f2006fca5c588c9469) added `isWithdrawClaimed[period][to]` validation so that a rescue cannot credit a destination whose key is already terminal. That check is active here and is strong evidence that crediting a terminal withdrawal key violates an intended invariant. It was applied to the role-gated writer and not to the public one.

`rescueWithdrawFromBlocklisted()` validates the flag at line 865 before crediting at line 868; `_requestWithdraw()` credits the identical `withdrawSharesOf[period][account]` entry at line 1017 with no check. My PoC uses both writers against the same `(period, address)` key: the rescue reverts `AlreadyClaimedPeriod` and `redeem()` succeeds. So, my report is the unguarded half of an incomplete fix, not a new design objection.

## Impact Details

To me, this causes the critical impact "Permanent freezing of funds". The affected amount is bounded by the restored user's available shares and the vault deposit limit.

My PoC starts from one 3,000-FXRP deposit, safely rescues a 1,000-FXRP pending withdrawal and the remaining 2,000 stXRP position, and then permanently freezes that remaining 2,000 FXRP without any redeposit. The vault physically retains 2,000 FXRP and continues to reserve exactly 2,000 FXRP in `pendingWithdrawAssets`, while `totalAssets()` reports zero active backing.

An incident payout drawing on that bucket while its window is open would reduce the reserved figure, but it does not restore any part of the entitlement: the owner's claim remains fully rejected by the terminal flag regardless of the bucket's residual. Other shareholders cannot absorb the reserved balance.

I tried, I think every scoped remedy and they are rejected by the same flag:

* the owner's `claimWithdraw()` reverts `AlreadyClaimedPeriod`.
* It ends up re-blocklisting and re-rescuing reverts `AlreadyClaimedPeriod` on the source side to any destination.
* `rescueSharesFromBlocklisted()` reverts `InsufficientShares` because the shares are already burned.
* Finally, no function in the codebase assigns `isWithdrawClaimed = false`.

Recovery requires an administrative implementation upgrade or storage migration.

The standard ERC4626 `maxRedeem()` and `maxWithdraw()` views continue to advertise the owner's full position as exitable because they do not consider the intended receiver's terminal withdrawal key.

An integration can detect the condition only by separately deriving the next period and querying the public `isWithdrawClaimed(period, receiver)` getter.

## Remediation

Do not set `isWithdrawClaimed[period][from]` after rescue has zeroed the source withdrawal shares; a repeated rescue already fails on zero amount. Independently, make `_requestWithdraw()` reject a receiver whose target-period key is terminal. Preventing rescue of the currently open next-period bucket is another safe chain break. Add the exact rescue, restoration, same-period request, claim, and re-rescue sequence as a regression.

## References

* [Scoped FirelightVault source](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol)
* [Sentora Firelight audit](https://github.com/firelight-protocol/firelight-core/blob/main/audits/2025-07-21_OpenZeppelin_Firelight.pdf)
* [Period-management fix `720d67d`](https://github.com/firelight-protocol/firelight-core/commit/720d67da9e83a18140bb96a7b95d5b3e250d1145)
* [Destination-side rescue fix `a2def47`](https://github.com/firelight-protocol/firelight-core/commit/a2def47716389330c0ebc1f2006fca5c588c9469)

## Link to Proof of Concept

<https://gist.github.com/niafreu/128850c7e944b1c8d549945d5e3b84a3>

## Proof of Concept

## Proof of Concept

The scoped V2 implementation is not deployed at the pinned block, so the test reads the live legacy proxy to authenticate the fork and deploys the exact scoped V2 implementation locally inside it.

Run:

```bash
FORK=1 \
NODE_RPC_URL="<FLARE_MAINNET_RPC_URL>" \
BN=67242818 \
HARDHAT_CHAIN_ID=14 \
npx hardhat test test/rescue_terminal_reuse.js
```

Expected result:

```
Rescue terminal key -> later withdrawal freeze
[fork] chain=14 block=67242818 hash=0xd593c82fde1fc088113b0ad3cadec464970e6fdbafdc9acb8836359aba6f7427
[fork] legacyProxy=0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3 asset=FXRP activeAssets=58288743.38103 FXRP
[setup] V2 is not deployed at the pinned block; the exact scoped implementation is deployed locally inside this fork
[positive] terminalKey=(1,0x90F79bf6EB2c4f870365E785982E1f101E93b906) claimed=true
[positive] physical=2000.0 FXRP pending=2000.0 FXRP active=0.0 FXRP
[positive] claim, withdrawal rescue to multiple destinations, and share rescue are all blocked
  ✔ A -> restoration -> reconsolidation -> B permanently freezes the remaining position
[control:rollover] poisonedPeriod=1 freshPeriod=2 claim=2000.0 FXRP
  ✔ waiting for rollover makes the reconsolidated position use a fresh key
[control:fresh-receiver] period=1 aggregateClaim=3000.0 FXRP
  ✔ a fresh receiver makes the reconsolidated position claimable

3 passing
```

The positive case proves:

* that complete rescue followed by restoration and ordinary position reconsolidation freezes exactly 2,000 FXRP without a redeposit.
* that the original rescued entitlement remains claimable
* that the later source entitlement can be recovered by no scoped path -- direct claim, re-rescue of the withdrawal to multiple destinations
* and that rescue of the shares all fail
* and that the vault retains the exact frozen amount as physical and pending assets.

I also added two negative controls to prove that waiting for rollover or selecting a fresh receiver breaks the collision. I made sure that the test works only if the source commit, Flare chain ID, and fork block match, prints the authenticated fork state and final frozen-fund ledger.

### PoC code

```js
const { expect } = require("chai");
const { execFileSync } = require("node:child_process");
const { ethers, network, upgrades } = require("hardhat");
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");

describe("Rescue terminal key -> later withdrawal freeze", function () {
  const SOURCE_COMMIT = "42f9ea5e43d88b35197b901acc2a8c24b314fa62";
  const FORK_BLOCK = 67242818;
  const LIVE_VAULT = "0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3";
  const DAY = 86400;
  const LIMIT = ethers.parseUnits("100000", 6);
  const TOTAL = ethers.parseUnits("3000", 6);
  const FIRST = ethers.parseUnits("1000", 6);
  const REMAINDER = TOTAL - FIRST;
  const fmt = (value) => `${ethers.formatUnits(value, 6)} FXRP`;

  before(async function () {
    expect(execFileSync("git", ["-C", "contracts", "rev-parse", "HEAD"], { encoding: "utf8" }).trim()).to.equal(SOURCE_COMMIT);
    expect(process.env.FORK).to.equal("1");
    expect(process.env.NODE_RPC_URL).to.be.a("string").and.not.equal("");
    expect(Number(process.env.BN)).to.equal(FORK_BLOCK);
    const metadata = await network.provider.send("hardhat_metadata");
    expect(metadata.forkedNetwork.chainId).to.equal(14);
    expect(metadata.forkedNetwork.forkBlockNumber).to.equal(FORK_BLOCK);
    const liveVault = await ethers.getContractAt("FirelightVaultPredeposits", LIVE_VAULT);
    const liveAsset = await ethers.getContractAt("IERC20Metadata", await liveVault.asset());
    console.log(`[fork] chain=14 block=${FORK_BLOCK} hash=${metadata.forkedNetwork.forkBlockHash}`);
    console.log(`[fork] legacyProxy=${LIVE_VAULT} asset=${await liveAsset.symbol()} activeAssets=${fmt(await liveVault.totalAssets())}`);
    console.log("[setup] V2 is not deployed at the pinned block; the exact scoped implementation is deployed locally inside this fork");
  });

  async function deployFixture() {
    const [admin, rescuer, blocklister, user, recipient] = await ethers.getSigners();
    const token = await (await ethers.getContractFactory("MockERC20")).deploy("Local FXRP", "FXRP", 6);
    await token.waitForDeployment();
    const initParams = ethers.AbiCoder.defaultAbiCoder().encode(
      ["address", "address", "address", "address", "address", "address", "uint256", "uint48"],
      [admin.address, admin.address, blocklister.address, admin.address, admin.address, rescuer.address, LIMIT, DAY],
    );
    const vault = await upgrades.deployProxy(
      await ethers.getContractFactory("FirelightVault"),
      [token.target, "Staked XRP", "stXRP", initParams],
      { kind: "transparent" },
    );
    await vault.waitForDeployment();
    await token.mint(user.address, TOTAL);
    await token.connect(user).approve(vault.target, TOTAL);
    await vault.connect(user).deposit(TOTAL, user.address);
    return { rescuer, blocklister, user, recipient, token, vault };
  }

  async function completeRescue(ctx) {
    const period = (await ctx.vault.currentPeriod()) + 1n;
    await ctx.vault.connect(ctx.user).redeem(FIRST, ctx.user.address, ctx.user.address);
    expect(await ctx.vault.balanceOf(ctx.user.address)).to.equal(REMAINDER);
    await ctx.vault.connect(ctx.blocklister).addToBlocklist(ctx.user.address);
    await ctx.vault.connect(ctx.rescuer).rescueWithdrawFromBlocklisted(ctx.user.address, ctx.recipient.address, [period]);
    await ctx.vault.connect(ctx.rescuer).rescueSharesFromBlocklisted(ctx.user.address, ctx.recipient.address);
    expect(await ctx.vault.balanceOf(ctx.user.address)).to.equal(0n);
    expect(await ctx.vault.balanceOf(ctx.recipient.address)).to.equal(REMAINDER);
    return period;
  }

  async function restoreAndReconsolidate(ctx) {
    await ctx.vault.connect(ctx.blocklister).removeFromBlocklist(ctx.user.address);
    await ctx.vault.connect(ctx.recipient).transfer(ctx.user.address, REMAINDER);
    expect(await ctx.vault.maxRedeem(ctx.user.address)).to.equal(REMAINDER);
  }

  it("A -> restoration -> reconsolidation -> B permanently freezes the remaining position", async function () {
    const ctx = await loadFixture(deployFixture);
    const period = await completeRescue(ctx);
    await restoreAndReconsolidate(ctx);
    expect((await ctx.vault.currentPeriod()) + 1n).to.equal(period);

    await expect(ctx.vault.connect(ctx.user).redeem(REMAINDER, ctx.user.address, ctx.user.address))
      .to.emit(ctx.vault, "WithdrawRequest")
      .withArgs(ctx.user.address, ctx.user.address, ctx.user.address, period, REMAINDER, REMAINDER);
    expect(await ctx.vault.balanceOf(ctx.user.address)).to.equal(0n);
    expect(await ctx.vault.withdrawalsOf(period, ctx.user.address)).to.equal(REMAINDER);

    await time.increase(2 * DAY);
    await ctx.vault.connect(ctx.recipient).claimWithdraw(period);
    await expect(ctx.vault.connect(ctx.user).claimWithdraw(period))
      .to.be.revertedWithCustomError(ctx.vault, "AlreadyClaimedPeriod")
      .withArgs(period);
    await ctx.vault.connect(ctx.blocklister).addToBlocklist(ctx.user.address);
    await expect(ctx.vault.connect(ctx.rescuer).rescueWithdrawFromBlocklisted(ctx.user.address, ctx.recipient.address, [period]))
      .to.be.revertedWithCustomError(ctx.vault, "AlreadyClaimedPeriod")
      .withArgs(period);
    await expect(ctx.vault.connect(ctx.rescuer).rescueSharesFromBlocklisted(ctx.user.address, ctx.recipient.address))
      .to.be.revertedWithCustomError(ctx.vault, "InsufficientShares");
    expect(await ctx.token.balanceOf(ctx.vault.target)).to.equal(REMAINDER);
    expect(await ctx.vault.pendingWithdrawAssets()).to.equal(REMAINDER);
    expect(await ctx.vault.totalAssets()).to.equal(0n);
    console.log(`[positive] terminalKey=(${period},${ctx.user.address}) claimed=true`);
    console.log(`[positive] physical=${fmt(REMAINDER)} pending=${fmt(REMAINDER)} active=${fmt(0n)}`);
    console.log("[positive] claim, withdrawal rescue to multiple destinations, and share rescue are all blocked");
  });

  it("waiting for rollover makes the reconsolidated position use a fresh key", async function () {
    const ctx = await loadFixture(deployFixture);
    const rescuedPeriod = await completeRescue(ctx);
    await time.increase(DAY);
    await restoreAndReconsolidate(ctx);
    const newPeriod = (await ctx.vault.currentPeriod()) + 1n;
    expect(newPeriod).to.equal(rescuedPeriod + 1n);
    await ctx.vault.connect(ctx.user).redeem(REMAINDER, ctx.user.address, ctx.user.address);
    await time.increase(2 * DAY);
    await ctx.vault.connect(ctx.user).claimWithdraw(newPeriod);
    expect(await ctx.token.balanceOf(ctx.user.address)).to.equal(REMAINDER);
    console.log(`[control:rollover] poisonedPeriod=${rescuedPeriod} freshPeriod=${newPeriod} claim=${fmt(REMAINDER)}`);
  });

  it("a fresh receiver makes the reconsolidated position claimable", async function () {
    const ctx = await loadFixture(deployFixture);
    const period = await completeRescue(ctx);
    await restoreAndReconsolidate(ctx);
    await ctx.vault.connect(ctx.user).redeem(REMAINDER, ctx.recipient.address, ctx.user.address);
    expect(await ctx.vault.isWithdrawClaimed(period, ctx.recipient.address)).to.equal(false);
    await time.increase(2 * DAY);
    await ctx.vault.connect(ctx.recipient).claimWithdraw(period);
    expect(await ctx.token.balanceOf(ctx.recipient.address)).to.equal(TOTAL);
    console.log(`[control:fresh-receiver] period=${period} aggregateClaim=${fmt(TOTAL)}`);
  });
});
```
