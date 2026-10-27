# 57285 sc medium incomplete signature in factory produce enables full accesstoken hijacking and direct fund theft

**Submitted on Oct 25th 2025 at 00:36:24 UTC by @pirex for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57285
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

The `Factory.produce()` function in Belong v2 contains a critical signature validation flaw that allows any attacker to front-run legitimate creators, hijack AccessToken collections, and take complete control of mint economics. The signature verification only covers metadata fields (name, symbol, contractURI, and feeNumerator) while omitting all critical deployment parameters including the intended creator address, payment token, mint pricing, transferability settings, supply caps, and collection expiry. This allows any user who obtains a valid signature to reuse it with arbitrary malicious parameters, deploy the AccessToken under their own control, permanently block the legitimate creator from deploying, and redirect all collection revenue to themselves.

### Vulnerability Details

The vulnerability stems from a severely incomplete signature that fails to authenticate the most critical aspects of AccessToken deployment. In `SignatureVerifier`, the signed payload only includes:

```solidity
keccak256(abi.encodePacked(
    metadata.name,
    metadata.symbol,
    contractURI,
    feeNumerator,
    block.chainid
))
```

However, `Factory.produce()` uses numerous additional parameters that are never verified.

Unsigned Critical Parameters:

* `creator` - Set to `msg.sender`, allows attacker to become owner
* `paymentToken` - Can be changed to worthless token or address(0)
* `mintPrice` - Can be set to 0 (free mints) or inflated
* `whitelistMintPrice` - Can be manipulated independently
* `transferable` - Can be disabled to lock tokens
* `maxTotalSupply` - Can be set to 1 or unlimited
* `collectionExpire` - Can be set to 0 (instant expiry) or removed
* No nonce or deadline for replay protection

Attack Sequence:

1. Legitimate creator signs a payload to deploy their AccessToken collection
2. Attacker observes or obtains this signature (from mempool, backend leak, or social engineering)
3. Attacker constructs a malicious `AccessTokenInfoStruct` with:
   * Same metadata (name, symbol, contractURI, feeNumerator) to match signature
   * Attacker's address as `msg.sender` (becomes creator/owner)
   * Manipulated pricing, payment token, supply, and settings
4. Attacker calls `produce()` with the reused signature
5. Signature validates successfully (only checks metadata)
6. AccessToken deploys with attacker as creator and malicious parameters
7. Legitimate creator is permanently blocked by `TokenAlreadyExists` error
8. All mint revenue flows to attacker's collection

Why This Happens:

* The signature authenticates the collection metadata but does not authorize who can deploy it.
* It does not specify or verify the economic parameters.
* `msg.sender` becomes the creator without any signature binding.
* The contract trusts unsigned user-controlled input for critical business logic.

This creates a front-running vulnerability where possession of a signature is sufficient to steal an entire collection deployment.

## Impact Details

This vulnerability enables complete takeover of AccessToken collections with severe consequences:

Unauthorized Collection Hijacking:

* Any attacker can deploy an AccessToken using another creator's signed metadata.
* The attacker becomes the collection owner and receives all revenue.
* The collection appears legitimate (uses signed metadata) but is under malicious control.
* No special privileges required — any address can call `produce()`.

Permanent Creator Lockout:

* Once attacker deploys with the signature, the name/symbol pair is registered.
* Legitimate creator's deployment attempt fails with `TokenAlreadyExists`.
* Creator has no recourse to reclaim their collection.
* The legitimate collection can never be deployed on-chain.

Complete Revenue Theft:

* Attacker controls all mint pricing and payment tokens.
* Can set `mintPrice = 0` to mint unlimited free tokens.
* Can change `paymentToken` to a worthless ERC20 they control.
* Can set `maxTotalSupply = 1` to create artificial scarcity.
* All legitimate mint revenue is lost or redirected.

Economic Manipulation:

* Attacker can disable `transferable` to lock all tokens.
* Can set `collectionExpire = 0` to make collection immediately expired.
* Can manipulate whitelist pricing independently.
* Can create deliberately broken collections to damage creator reputation.

Platform Trust Damage:

* Creators lose confidence in the platform's security.
* Legitimate collections cannot be deployed reliably.
* Platform revenue is reduced (stolen collections pay attacker, not platform).
* Reputation damage from widespread collection hijacking.

Scalability:

* Attack can be automated to front-run all new AccessToken deployments.
* Multiple collections can be hijacked simultaneously.
* Works on any AccessToken regardless of intended parameters.
* No on-chain detection or prevention mechanism exists.

Financial Impact Example:

* Creator signs deployment for premium collection (1 ETH mint price, 10K supply).
* Expected creator revenue: 10,000 ETH.
* Attacker front-runs with same signature but sets mintPrice = 0.
* Attacker mints entire supply for free.
* Creator receives: 0 ETH.
* Platform receives: 0 ETH.
* Attacker profit: 10,000 tokens worth of potential value.

This qualifies as Critical severity under Immunefi criteria: "Unauthorized minting of NFTs" and "Direct theft of any user funds, whether at-rest or in-motion."

## References

Affected Contracts:

* `contracts/v2/platform/Factory.sol` - `produce()` function with incomplete validation
* `contracts/v2/utils/SignatureVerifier.sol` - Missing signature bindings
* `contracts/v2/Structures.sol` - `AccessTokenInfoStruct` with unsigned critical fields
* `contracts/v2/nft/AccessToken.sol` - Deployed with attacker-controlled parameters

Key Functions:

* `Factory.produce()` - Entry point for collection deployment
* `SignatureVerifier.checkAccessTokenInfo()` - Incomplete signature verification
* `AccessToken` constructor - Accepts unsigned parameters from attacker

## Proof of Concept

A complete proof of concept demonstrates the attack by showing that a signature created for a legitimate deployment can be reused by an attacker to hijack the collection with completely different parameters.

Full Git Diff (configuration & test changes for PoC)

Configuration changes for testing environment:

```diff
diff --git a/hardhat.config.ts b/hardhat.config.ts
index 5023728..c5096b5 100644
--- a/hardhat.config.ts
+++ b/hardhat.config.ts
@@ -20,6 +20,8 @@ if (process.env.LEDGER_ADDRESS) {
   ledgerAccounts = [process.env.LEDGER_ADDRESS];
 }
 
+const mainnetForkBlock = process.env.MAINNET_FORK_BLOCK ? Number(process.env.MAINNET_FORK_BLOCK) : undefined;
+
 const config: HardhatUserConfig = {
   solidity: {
     compilers: [
@@ -37,10 +39,12 @@ const config: HardhatUserConfig = {
     networks: {
       hardhat: {
         forking: {
-        url: process.env.INFURA_ID_PROJECT
-          ? `https://mainnet.infura.io/v3/${process.env.INFURA_ID_PROJECT}`
-          : `https://eth.llamarpc.com`,
-        blockNumber: 23490636,
+        url: process.env.MAINNET_RPC_URL
+          ? process.env.MAINNET_RPC_URL
+          : process.env.INFURA_ID_PROJECT
+            ? `https://mainnet.infura.io/v3/${process.env.INFURA_ID_PROJECT}`
+            : `https://eth.llamarpc.com`,
+        ...(mainnetForkBlock ? { blockNumber: mainnetForkBlock } : {}),
         },
         accounts: { accountsBalance: '10000000000000000000000000' },
       },
```

Other diffs omitted for brevity (see original PoC for full changes).

PoC Test modifications (key test added to `test/v2/platform/factory.test.ts`):

* Creates a signature that only covers metadata (name, symbol, contractURI, feeNumerator)
* Reuses the signature to deploy an AccessToken with attacker-controlled parameters (payment token, mint prices, transferability, supply, expiry)
* Verifies the attacker becomes the creator, parameters are accepted, and legitimate creator is blocked by `TokenAlreadyExists`

Key test changes (excerpt):

```diff
+    it('allows attacker to hijack AccessToken parameters via reused signature', async () => {
+      const { factory, alice, bob, erc20Example, signer } = await loadFixture(fixture);
+
+      // ... set up metadata and legitimate parameters ...
+
+      // Create signature that only covers metadata
+      const message = hashAccessTokenInfo(nftName, nftSymbol, contractURI, feeNumerator, chainId);
+      const signature = EthCrypto.sign(signer.privateKey, message);
+
+      // Legitimate creator's intended parameters
+      const legitimateInfo: AccessTokenInfoStruct = { ... signature, intended params ... };
+
+      // Attacker reuses signature with malicious parameters
+      const attackerInfo: AccessTokenInfoStruct = { ... signature, malicious params ... };
+
+      // Attacker (bob) front-runs and deploys with hijacked parameters
+      await expect(factory.connect(bob).produce(attackerInfo, ethers.constants.HashZero)).to.emit(
+        factory,
+        'AccessTokenCreated',
+      );
+
+      // Verify attacker now owns the collection and params were accepted
+
+      // Verify legitimate creator (alice) is now permanently blocked
+      await expect(
+        factory.connect(alice).produce(legitimateInfo, ethers.constants.HashZero),
+      ).to.be.revertedWithCustomError(factory, 'TokenAlreadyExists');
+    });
```

To Reproduce (steps)

{% stepper %}
{% step %}
Install dependencies:

```bash
pnpm install
```

{% endstep %}

{% step %}
Run the PoC test:

```bash
MAINNET_RPC_URL=https://ethereum.publicnode.com \
LEDGER_ADDRESS=0x1111111111111111111111111111111111111111 \
PK=0x2222222222222222222222222222222222222222222222222222222222222222 \
pnpm test test/v2/platform/factory.test.ts --grep "hijack AccessToken parameters"
```

{% endstep %}
{% endstepper %}

Test Results:

* The test passes and demonstrates the signature covering only metadata can be reused to deploy an AccessToken with arbitrary attacker-controlled parameters.
* Attacker becomes the creator, all malicious parameters are accepted, and the legitimate creator is blocked.

This conclusively proves that any signature can be reused to deploy AccessToken collections with arbitrary malicious parameters under attacker control.

## Recommended Mitigation

1. Include all critical parameters in signature (example using abi.encodePacked shown):

```solidity
keccak256(abi.encodePacked(
    metadata.name,
    metadata.symbol,
    contractURI,
    creator,                    // Add this
    paymentToken,               // Add this
    mintPrice,                  // Add this
    whitelistMintPrice,         // Add this
    transferable,               // Add this
    maxTotalSupply,             // Add this
    collectionExpire,           // Add this
    feeNumerator,
    nonce,                      // Add this
    deadline,                   // Add this
    block.chainid
))
```

2. Verify msg.sender matches signed creator:

```solidity
require(msg.sender == accessTokenInfo.creator, "Unauthorized creator");
```

3. Implement replay protection:

```solidity
mapping(address => mapping(uint256 => bool)) public usedNonces;

function produce(AccessTokenInfo calldata info, bytes32 referralCode) external {
    require(!usedNonces[info.creator][info.nonce], "Nonce already used");
    usedNonces[info.creator][info.nonce] = true;
    // ... rest of function
}
```

4. Add deadline validation:

```solidity
require(block.timestamp <= info.deadline, "Signature expired");
```

5. Use EIP-712 structured data signatures (preferred):

```solidity
// Implement proper typed structured data hashing
bytes32 structHash = keccak256(abi.encode(
    ACCESS_TOKEN_TYPEHASH,
    keccak256(bytes(metadata.name)),
    keccak256(bytes(metadata.symbol)),
    keccak256(bytes(contractURI)),
    creator,
    paymentToken,
    mintPrice,
    whitelistMintPrice,
    transferable,
    maxTotalSupply,
    collectionExpire,
    feeNumerator,
    nonce,
    deadline
));
```

Severity: CRITICAL (per attack impact and Immunefi criteria — unauthorized minting and direct theft of funds)

***

Note: All links and code snippets are preserved as in the original report. No navigation or unrelated UI elements were included.
