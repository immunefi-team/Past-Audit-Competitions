# 56869 sc medium hijacking deployment of accesstoken and stealing ownership to prevent further deployments

**Submitted on Oct 21st 2025 at 11:39:54 UTC by @blackgrease for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #56869
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/belongnet/checkin-contracts/blob/main/contracts/v2/platform/Factory.sol>

## Impacts

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Unintended alteration of what the NFT represents (e.g. token URI, payload, artistic content)
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Affected Files: `Factory.sol` and `AccessToken.sol`

The `Factory::produce` function creates a new `AccessToken` collection and an optional `RoyaltiesReceiver` (if `feeNumerator` > 0). The function requires a signature from an authorized signer on a hash of `NFT name, symbol, contractURI, feeNumerator` and `block.chainid`. However, the function is permissionless — anyone can call it with a valid signature. The deployment uses CREATE2 deterministic addresses and `AccessToken` is Ownable, so the caller becomes the owner of the deployed instance.

The salt hash is derived from token name and symbol; for a given name/symbol pair there cannot be duplicate `AccessToken` instances:

```solidity
//--snip--
bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol);
require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists());
//--snip
```

### The Issue

An attacker can front-run `Factory::produce` to take ownership of the `AccessToken` collection and gain full ownership of the instance. Because only one deployment exists for a given token name & symbol, the intended user's subsequent transaction will fail, causing a denial of service.

`Factory::produce` does not validate variables beyond those signed, allowing an attacker to alter values like `feeNumerator`. If `feeNumerator > 0`, this deploys a `RoyaltiesReceiverV2` which the attacker can receive profits from.

By taking over the `AccessToken` and becoming owner, an attacker can:

* prevent any other deployment of the same name/symbol pair (denial of service),
* change minting prices,
* receive royalties if `feeNumerator` was set,
* upgrade `AccessToken` to a malicious version and alter the NFT's behavior/content.

The legitimate user would need the `Signer` to sign new metadata, but the attack can be repeated, so the issue persists until fixed.

### Full problematic code (excerpt)

```solidity
//@audit-issue: can be front-run to claim ownership and prevent further deployments + steal any royoalties
 function produce(AccessTokenInfo memory accessTokenInfo, bytes32 referralCode) external returns (address nftAddress) {
        FactoryParameters memory factoryParameters = _nftFactoryParameters;

        factoryParameters.signerAddress.checkAccessTokenInfo(accessTokenInfo);

        bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol);

        require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists());

        accessTokenInfo.paymentToken = accessTokenInfo.paymentToken == address(0)   ? factoryParameters.defaultPaymentCurrency : accessTokenInfo.paymentToken;

        Implementations memory currentImplementations = _currentImplementations;

        address predictedRoyaltiesReceiver = currentImplementations.royaltiesReceiver.predictDeterministicAddress(hashedSalt, address(this));
        address predictedAccessToken = currentImplementations.accessToken.predictDeterministicAddressERC1967(hashedSalt, address(this));

        address receiver;
        _setReferralUser(referralCode, msg.sender);
        if (accessTokenInfo.feeNumerator > 0) {
            receiver = currentImplementations.royaltiesReceiver.cloneDeterministic(hashedSalt);
            require(predictedRoyaltiesReceiver == receiver, RoyaltiesReceiverAddressMismatch());
            RoyaltiesReceiverV2(payable(receiver)) 
                .initialize(
                    RoyaltiesReceiverV2.RoyaltiesReceivers(
                        msg.sender, factoryParameters.platformAddress, referrals[referralCode].creator  //@audit-issue: attacker can also receive any royalties if the `feeNumerator` was previously set
                    ),
                    Factory(address(this)),
                    referralCode
                );
        }

        nftAddress = currentImplementations.accessToken.deployDeterministicERC1967(hashedSalt);
        require(predictedAccessToken == nftAddress, AccessTokenAddressMismatch());
        AccessToken(nftAddress)
            .initialize(
                AccessToken.AccessTokenParameters({
                    factory: Factory(address(this)),
                    info: accessTokenInfo,
                    creator: msg.sender, //@audit-issue: whoever front-runs this transaction becomes the owner of AccessToken
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

        getNftInstanceInfo[hashedSalt] = accessTokenInstanceInfo;

        emit AccessTokenCreated(hashedSalt, accessTokenInstanceInfo);
    }
```

## Impact

* Disruption of legitimate operations: further `AccessToken` deployments for the same name/symbol will revert (DoS).
* Loss of royalties for the intended recipient if an attacker set `feeNumerator`.
* Business loss due to altered mint prices.
* Griefing by making NFTs inaccessible or modifying mint economics.
* Arbitrary upgrades of `AccessToken` to change NFT behavior/content.

Likelihood of exploitation: High (no execution restrictions; attacker simply front-runs a transaction).

## Mitigation

Control who can call `Factory::produce` ensuring the caller matches the intended input, or include the caller in the salt so the salt is unique per caller:

Suggested change:

```diff
-   bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol);
+   bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol, msg.sender);
```

This prevents an attacker from successfully front-running another caller for the same name/symbol.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/46e26dc7097dc866d2311ce7711f4523>

## Proof of Concept

The PoC (in the gist) demonstrates:

1. Front-running an intended owner's transaction to claim `AccessToken` ownership.
2. Confirming the attacker is owner by executing `onlyOwner` functionality.
3. Confirming that only one `AccessToken` exists per name/symbol pair by showing the legitimate user's deployment reverts.

### Conversion to Foundry and running the PoC

{% stepper %}
{% step %}

### Setup

1. Clone the GitHub repo:
   * `git clone https://github.com/belongnet/checkin-contracts.git`
     {% endstep %}

{% step %}

### Install Foundry dependencies

1. `forge install OpenZeppelin/openzeppelin-contracts-upgradeable@v5.4.0 --no-commit`
2. `forge install OpenZeppelin/openzeppelin-contracts@v5.4.0 --no-commit`
3. `npm install solady --force`
   {% endstep %}

{% step %}

### Update remappings in `foundry.toml`

Replace or update remappings as shown:

```toml
## Comment out the previous (some issue as npm dependencies did not install well)

# remappings = [
#     "@ensdomains/=node_modules/@ensdomains/",
#     "@openzeppelin/=node_modules/@openzeppelin/",
#     "eth-gas-reporter/=node_modules/eth-gas-reporter/",
#     "hardhat-deploy/=node_modules/hardhat-deploy/",
#     "hardhat/=node_modules/hardhat/",
#     "operator-filter-registry/=node_modules/operator-filter-registry/",
#     "solady/=node_modules/solady/",
# ]

## The new remappings

## Foundry Remappings
remappings = ["@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts","@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts"]
```

{% endstep %}

{% step %}

### Run the PoC

* Run with: `forge test --mt testAddressHijackingAndAccessTokenOwnershipTakeoever -vvv`
  {% endstep %}
  {% endstepper %}

> Note: In case of issues running the PoC, the gist includes a Foundry stack trace file: "PoC\_StackTrace\_AddressHijackingAndAccessTokenOwnershipTakeoever.txt"

***

End of report.
