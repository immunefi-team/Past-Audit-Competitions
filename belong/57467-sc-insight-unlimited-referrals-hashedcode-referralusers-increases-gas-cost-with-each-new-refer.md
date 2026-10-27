# 57467 sc insight unlimited referrals hashedcode referralusers increases gas cost with each new referral making it very expensive&#x20;

**Submitted on Oct 26th 2025 at 13:41:48 UTC by @Oxlookman for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57467
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/extensions/ReferralSystemV2.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Unbounded gas consumption

{% hint style="info" %}
Brief/Intro

Growing `referrals[hashedCode].referralUsers` array will make it difficult for a referrer to refer new users as this array grows.
{% endhint %}

## Description

### Vulnerability Details

Referrer's referral array increases over time. When a new referral is being added in `_setReferralUser`, the array is iterated to check if the referral is new:

```solidity
for (uint256 i; i < len; ++i) {
    if (users[i] == referralUser) {
        // User already added; no need to add again
        inArray = true;
        break;
    }
}
if (!inArray) users.push(referralUser);
```

Since this array keeps growing with each new referral it will become very expensive for referrals to use the referrer code and at some point it will be DoSed.

### Impact Details

* The referrer will pay more gas fees as the array keeps growing, discouraging use of their referral code and causing them to lose their intended percentage.
* Users also incur very expensive transactions.

### References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/platform/extensions/ReferralSystemV2.sol#L148C8-L155C48>

## Proof of Concept

<details>

<summary>Test demonstrating increasing transaction costs with more referrals</summary>

Add this to `factort.test.ts` in the describe with referral, setup local network in your config since this test was run on the localhost.

```javascript
it('Test transaction costs increase with each new refferal', async () => {
    const { factory, signer, alice, bob } = await loadFixture(fixture);

    const hashedCode = EthCrypto.hash.keccak256([
      { type: 'address', value: alice.address },
      { type: 'address', value: factory.address },
      { type: 'uint256', value: chainId },
    ]);
    

  await factory.connect(alice).createReferralCode();

  let nftName = 'Name';
  let nftSymbol = 'AT';
  const contractURI = 'contractURI/AccessToken123';
  const price = ethers.utils.parseEther('0.01');
  const feeNumerator = 500;

  let message = EthCrypto.hash.keccak256([
    { type: 'string', value: nftName },
    { type: 'string', value: nftSymbol },
    { type: 'string', value: contractURI },
    { type: 'uint96' as any, value: feeNumerator },
    { type: 'uint256', value: chainId },
  ]);

  let signature = EthCrypto.sign(signer.privateKey, message);

  let tx = await factory.connect(bob).produce(
      {
        metadata: { name: nftName, symbol: nftSymbol },
        contractURI: contractURI,
        paymentToken: NATIVE_CURRENCY_ADDRESS,
        mintPrice: price,
        whitelistMintPrice: price,
        transferable: true,
        maxTotalSupply: BigNumber.from('1000'),
        feeNumerator: feeNumerator,
        collectionExpire: BigNumber.from('86400'),
        signature: signature,
      } as AccessTokenInfoStruct,
      hashedCode,
    )

  let receipt = await tx.wait();
  const firstGasUsed = receipt.gasUsed;
  console.log(`This is the last receipt: ${ receipt.gasUsed}`)

  let i = 0;

  while( i < 100) {
    let randomWallet = ethers.Wallet.createRandom().connect(ethers.provider);
    await alice.sendTransaction({to: await randomWallet.getAddress(), value: ethers.utils.parseEther('0.1')})

    let nftName = `Name ${ i++ }`;
    let nftSymbol = `AT ${ i++ }`;
    const contractURI = 'contractURI/AccessToken123';
    const price = ethers.utils.parseEther('0.01');
    const feeNumerator = 500;

    let message = EthCrypto.hash.keccak256([
      { type: 'string', value: nftName },
      { type: 'string', value: nftSymbol },
      { type: 'string', value: contractURI },
      { type: 'uint96' as any, value: feeNumerator },
      { type: 'uint256', value: chainId },
    ]);

    let signature = EthCrypto.sign(signer.privateKey, message);

    tx = await factory.connect(randomWallet).produce(
        {
          metadata: { name: nftName, symbol: nftSymbol },
          contractURI: contractURI,
          paymentToken: NATIVE_CURRENCY_ADDRESS,
          mintPrice: price,
          whitelistMintPrice: price,
          transferable: true,
          maxTotalSupply: BigNumber.from('1000'),
          feeNumerator: feeNumerator,
          collectionExpire: BigNumber.from('86400'),
          signature: signature,
        } as AccessTokenInfoStruct,
        hashedCode,
      )
  }

  receipt = await tx.wait();
  const diff = receipt.gasUsed.sub(firstGasUsed);
  const secondCheckpointGasUsed = receipt.gasUsed;
  console.log(`This is the last receipt gasUsed:  ${secondCheckpointGasUsed}`)

  assert(receipt.gasUsed != firstGasUsed && diff.gt(0n))
  console.log(`The gas difference is after around 100 referrals: ${diff}`)
  
  let j = 0;
  while( j < 100) {
    let randomWallet = ethers.Wallet.createRandom().connect(ethers.provider);
    await alice.sendTransaction({to: await randomWallet.getAddress(), value: ethers.utils.parseEther('0.1')})

    let nftName = `Name ${ i++ }`;
    let nftSymbol = `AT ${ i++ }`;
    const contractURI = 'contractURI/AccessToken123';
    const price = ethers.utils.parseEther('0.01');
    const feeNumerator = 500;

    let message = EthCrypto.hash.keccak256([
      { type: 'string', value: nftName },
      { type: 'string', value: nftSymbol },
      { type: 'string', value: contractURI },
      { type: 'uint96' as any, value: feeNumerator },
      { type: 'uint256', value: chainId },
    ]);

    let signature = EthCrypto.sign(signer.privateKey, message);

    tx = await factory.connect(randomWallet).produce(
        {
          metadata: { name: nftName, symbol: nftSymbol },
          contractURI: contractURI,
          paymentToken: NATIVE_CURRENCY_ADDRESS,
          mintPrice: price,
          whitelistMintPrice: price,
          transferable: true,
          maxTotalSupply: BigNumber.from('1000'),
          feeNumerator: feeNumerator,
          collectionExpire: BigNumber.from('86400'),
          signature: signature,
        } as AccessTokenInfoStruct,
        hashedCode,
      )
    j++;
  }
  receipt = await tx.wait();
  const diff2 = receipt.gasUsed.sub(firstGasUsed);
  console.log(`This is the receipt of gasUsed after 200 referrals: ${receipt.gasUsed}`)
  assert(receipt.gasUsed != secondCheckpointGasUsed && diff2 != diff)
  assert(receipt.gasUsed > secondCheckpointGasUsed && diff2 > diff)
  console.log(`The gas difference is from first transaction after around 200 referals: ${diff2}`)

});
```

</details>
