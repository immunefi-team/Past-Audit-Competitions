# 57671 sc high royaltiesreceiverv2 shares referralshare uses dynamic values which may result in failure to release funds properly&#x20;

**Submitted on Oct 28th 2025 at 02:30:23 UTC by @i0x1982us for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57671
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/RoyaltiesReceiverV2.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

### Brief / Intro

`referralShare` decreases as `referralCode` is used more often, resulting in a larger `platformShare`. This may result in the contract not having enough balance to pay, causing `releaseAll` / `release` to fail and rewards to be frozen.

### Vulnerability Details

`RoyaltiesReceiverV2.shares()` is used to calculate the shares corresponding to `creator/referral/platform`.

* `creator` = `factory.royaltiesParameters().amountToCreator`
* `referral` = `factory.getReferralRate(creator, referralCode, royaltiesParameters.amountToPlatform)`
* `platform` = `factory.royaltiesParameters().amountToCreator` - `referral`

All three values are variable, especially `referral`, which gets smaller as `usedCode[referralUser][code]` increases.

This leads to a problem: if `referral` is released first, and then shares get smaller, the contract balance reserved/expected for distribution will not match the new shares, potentially causing releases to revert due to insufficient balance.

Example values (from the report):

* amountToCreator = 80%
* amountToPlatform = 20%
* usedToPercentage\[2] = 30%
* usedToPercentage\[3] = 10%
* usedCode\[referralUser]\[code] = 2 (referralCode used 2 times)

So:

* creator = 80%
* referral = 20% \* 30% = 6%
* platform = 20% - 6% = 14%

Royalties balance = 1000

{% stepper %}
{% step %}

### Step

1. Call releaseAll()

* creator gets = 1000 \* 80% = 800
* platform gets = 1000 \* 14% = 140
* referral gets = 1000 \* 6% = 60
* Royalties\_balance = 0
  {% endstep %}

{% step %}

### Step

2. `factory.produce(some_creator, some_referral_code)`

* `usedCode[referralUser][code]` increases from 2 to 3
* referral becomes = amountToPlatform \* usedToPercentage\[3] = 20% \* 10% = 2%
* platform becomes = 18%
  {% endstep %}

{% step %}

### Step

3. New reward 1000 is sent to RoyaltiesReceiverV2 and `releaseAll()` is called:

* creator get = 2000 \* 80% - 800 = 800
* platform get = 2000 \* 18% - 140 = 220 (revert: not enough balance; 200 left)
* This causes a revert and prevents distribution of funds.
  {% endstep %}
  {% endstepper %}

Note: `factory.setFactoryParameters()` may also cause similar problems as above by changing referral percentages or other parameters after prior releases.

### Impact Details

Rewards are not distributed properly and may be locked until there is enough balance for another reward. Donated rewards may also be partially split and locked.

## Proof of Concept

The following test demonstrates the issue. In this example `setFactoryParameters()` is used instead of modifying `usedCode[referralUser][code]`, but the effect is the same: referral percentage is reduced from `6%` to `2%`, platform percentage increases from `14%` to `18%`, and a subsequent distribution reverts with insufficient balance.

Add this to `accesstoken.test.ts`:

```ts
it('dynamic_shares_break_release', async () => {
  const {
    creator,
    signer,
    royaltiesReceiverEth,
    accessTokenEth,
    erc20Example,
    factory,
    owner,
    referral,
    charlie,
    pete,
    referralCode,
  } = await loadFixture(fixture);

  let message = EthCrypto.hash.keccak256([
    { type: 'address', value: charlie.address },
    { type: 'uint256', value: 0 },
    { type: 'string', value: NFT_721_BASE_URI },
    { type: 'bool', value: false },
    { type: 'uint256', value: chainId },
  ]);

  let signature = EthCrypto.sign(signer.privateKey, message);

  const startBalanceOwner = await owner.getBalance();
  const startBalanceCreator = await creator.getBalance();
  const startBalanceReferral = await referral.getBalance();

  await accessTokenEth.connect(charlie).mintStaticPrice(
    charlie.address,
    [
      {
        tokenId: 0,
        tokenUri: NFT_721_BASE_URI,
        whitelisted: false,
        signature,
      } as StaticPriceParametersStruct,
    ],
    NATIVE_CURRENCY_ADDRESS,
    ethPurchasePrice,
    {
      value: ethPurchasePrice,
    },
  );

  const endBalanceOwner = await owner.getBalance();
  const endBalanceCreator = await creator.getBalance();
  const endBalanceReferral = await referral.getBalance();

  const fullFees = ethPurchasePrice.div(BigNumber.from('100'));
  const feesToReferralCreator = await factory.getReferralRate(creator.address, referralCode, fullFees);
  const feesToPlatform = fullFees.sub(feesToReferralCreator);

  expect(endBalanceOwner.sub(startBalanceOwner)).to.be.equal(feesToPlatform);
  expect(endBalanceReferral.sub(startBalanceReferral)).to.be.equal(feesToReferralCreator);
  expect(endBalanceCreator.sub(startBalanceCreator)).to.be.equal(
    ethPurchasePrice.mul(BigNumber.from('99')).div(BigNumber.from('100')),
  );

  let tx = {
    from: owner.address,
    to: royaltiesReceiverEth.address,
    value: ethers.utils.parseEther('1'),
    gasLimit: 1000000,
  };

  const platformAddress = (await factory.nftFactoryParameters()).platformAddress;

  await owner.sendTransaction(tx);

  let creatorBalanceBefore = await creator.getBalance();
  let platformBalanceBefore = await owner.getBalance();
  let referralBalanceBefore = await referral.getBalance();

  await royaltiesReceiverEth.connect(pete).releaseAll(royaltiesReceiverEth.NATIVE_CURRENCY_ADDRESS());

  expect(await royaltiesReceiverEth.totalReleased(royaltiesReceiverEth.NATIVE_CURRENCY_ADDRESS())).to.eq(
    ethers.utils.parseEther('1'),
  );
  expect(
    await royaltiesReceiverEth.released(royaltiesReceiverEth.NATIVE_CURRENCY_ADDRESS(), creator.address),
  ).to.eq(ethers.utils.parseEther('0.8'));
  expect(await royaltiesReceiverEth.released(royaltiesReceiverEth.NATIVE_CURRENCY_ADDRESS(), owner.address)).to.eq(
    ethers.utils.parseEther('0.14'),
  );
  expect(
    await royaltiesReceiverEth.released(royaltiesReceiverEth.NATIVE_CURRENCY_ADDRESS(), referral.address),
  ).to.eq(ethers.utils.parseEther('0.06'));

  let creatorBalanceAfter = await creator.getBalance();
  let platformBalanceAfter = await owner.getBalance();
  let referralBalanceAfter = await referral.getBalance();

  expect(creatorBalanceAfter.sub(creatorBalanceBefore)).to.be.equal(ethers.utils.parseEther('0.8'));
  expect(platformBalanceAfter.sub(platformBalanceBefore)).to.be.equal(ethers.utils.parseEther('0.14'));
  expect(referralBalanceAfter.sub(referralBalanceBefore)).to.be.equal(ethers.utils.parseEther('0.06'));

  creatorBalanceBefore = await erc20Example.balanceOf(creator.address);
  platformBalanceBefore = await erc20Example.balanceOf(platformAddress);
  referralBalanceBefore = await erc20Example.balanceOf(referral.address);

  await erc20Example.connect(owner).mint(royaltiesReceiverEth.address, ethers.utils.parseEther('1'));

  await royaltiesReceiverEth.connect(pete).releaseAll(erc20Example.address);

  expect(await royaltiesReceiverEth.totalReleased(erc20Example.address)).to.eq(ethers.utils.parseEther('1'));
  expect(await royaltiesReceiverEth.released(erc20Example.address, creator.address)).to.eq(
    ethers.utils.parseEther('0.8'),
  );
  expect(await royaltiesReceiverEth.released(erc20Example.address, owner.address)).to.eq(
    ethers.utils.parseEther('0.14'),
  );
  expect(await royaltiesReceiverEth.released(erc20Example.address, referral.address)).to.eq(
    ethers.utils.parseEther('0.06'),
  );

  creatorBalanceAfter = await erc20Example.balanceOf(creator.address);
  platformBalanceAfter = await erc20Example.balanceOf(platformAddress);
  referralBalanceAfter = await erc20Example.balanceOf(referral.address);

  expect(creatorBalanceAfter.sub(creatorBalanceBefore)).to.be.equal(ethers.utils.parseEther('0.8'));
  expect(platformBalanceAfter.sub(platformBalanceBefore)).to.be.equal(ethers.utils.parseEther('0.14'));
  expect(referralBalanceAfter.sub(referralBalanceBefore)).to.be.equal(ethers.utils.parseEther('0.06'));


  //@audit change referralPercentages
  referralPercentages[2] = 1000;
  await factory.setFactoryParameters(factoryParams, royalties, implementations, referralPercentages);


  let tx2 = {
    from: owner.address,
    to: royaltiesReceiverEth.address,
    value: ethers.utils.parseEther('1'),
    gasLimit: 1000000,
  };

  await owner.sendTransaction(tx2);

  await royaltiesReceiverEth.connect(pete).releaseAll(royaltiesReceiverEth.NATIVE_CURRENCY_ADDRESS());
  expect(
    await royaltiesReceiverEth.released(royaltiesReceiverEth.NATIVE_CURRENCY_ADDRESS(), creator.address),
  ).to.eq(ethers.utils.parseEther('1.6'));      
});    
```

Console output when running the test:

```console
yarn test --grep  "dynamic_shares_break_release"

     dynamic_shares_break_release:
 Error: VM Exception while processing transaction: reverted with an unrecognized custom error (return data: 0xb12d13eb)
    at RoyaltiesReceiverV2.safeTransferETH (solady/src/utils/SafeTransferLib.sol:108)
    at RoyaltiesReceiverV2._release (contracts/v2/periphery/RoyaltiesReceiverV2.sol:216)
    at RoyaltiesReceiverV2.releaseAll (contracts/v2/periphery/RoyaltiesReceiverV2.sol:150)
    at <UnrecognizedContract>.<unknown> (0x0c756cf2873119eb13be4f4de64b6110a1015729)
    at EdrProviderWrapper.request (node_modules/hardhat/src/internal/hardhat-network/provider/provider.ts:359:41)
    at async EthersProviderWrapper.send (node_modules/@nomiclabs/hardhat-ethers/src/internal/ethers-provider-wrapper.ts:13:20)
```

## References

Target file: <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/RoyaltiesReceiverV2.sol>

***

(End of report)
