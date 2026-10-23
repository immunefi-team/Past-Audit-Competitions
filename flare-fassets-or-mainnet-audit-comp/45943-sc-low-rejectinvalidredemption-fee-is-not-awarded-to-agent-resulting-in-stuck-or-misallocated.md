# #45943 \[SC-Low] rejectInvalidRedemption fee is not awarded to agent, resulting in stuck or misallocated funds

**Submitted on May 22nd 2025 at 19:33:29 UTC by @magtentic for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45943
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/facets/RedemptionRequestsFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

When an agent successfully calls `rejectInvalidRedemption`, they are supposed to receive the `_executorFeeNatGWei` as compensation for providing proof of address invalidity. However, this fee is never transferred. Instead, it remains in the Asset Manager contract, effectively locking the funds or allowing unintended parties with transfer privileges to claim them.

## Vulnerability Details

The expected behavior—based on logic in `Redemptions.payOrBurnExecutorFee()`—is that either:

* the executor of a redemption receives the NAT fee, or
* if the sender is not the executor, the NAT is burned.

```
    function payOrBurnExecutorFee(
        Redemption.Request storage _request
    )
        internal
    {
        uint256 executorFeeNatWei = _request.executorFeeNatGWei * Conversion.GWEI;
        if (executorFeeNatWei > 0) {
            _request.executorFeeNatGWei = 0;
            if (msg.sender == _request.executor) {
                Transfers.transferNAT(_request.executor, executorFeeNatWei);
            } else {
                Agents.burnDirectNAT(executorFeeNatWei);
            }
        }
    }
```

However, when `rejectInvalidRedemption` is called, this fee-handling logic is not invoked. Despite the documentation in `RedemptionRequestsFacet.sol` implying that agents should be rewarded for proving invalidity (i.e., doing the work), they receive nothing:

```
    /**
     * If the redeemer provides invalid address, the agent should provide the proof of address invalidity from the
     * Flare data connector. With this, the agent's obligations are fulfilled and they can keep the underlying.
```

This means `_executorFeeNatGWei` remains in the Asset Manager contract indefinitely or until claimed by another party (e.g., if they have transfer rights), which violates the economic expectations of the agent role.

## Impact Details

* The agent, who performs the required rejection work, is not rewarded, violating protocol expectations.
* The NAT fee remains stuck in the Asset Manager contract.
* If the Asset Manager or another privileged contract can move these funds, this could result in unfair extraction of value.
* Incentive misalignment: Agents may avoid performing valid rejections if the cost of proof isn't compensated.

## Impacts

* High Severity: Theft of unclaimed yield

> The yield/fee is unclaimed due to protocol oversight and can potentially be taken by unintended parties. OR

* High Severity: Permanent freezing of unclaimed yield

> The fee is never transferred or burned, and there's no apparent recovery path.

## References

Redemptions - <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/Redemptions.sol>

## Proof of Concept

## Proof of Concept

```
import { expectEvent, expectRevert, time } from "@openzeppelin/test-helpers";
import { AgentSettings, CollateralType } from "../../../../lib/fasset/AssetManagerTypes";
import { PaymentReference } from "../../../../lib/fasset/PaymentReference";
import { AttestationHelper } from "../../../../lib/underlying-chain/AttestationHelper";
import { filterEvents, requiredEventArgs } from "../../../../lib/utils/events/truffle";
import { BNish, HOURS, MAX_BIPS, randomAddress, toBIPS, toBN, toBNExp, toNumber, toWei, ZERO_ADDRESS } from "../../../../lib/utils/helpers";
import { AgentVaultInstance, CollateralPoolInstance, ERC20MockInstance, FAssetInstance, IIAssetManagerInstance, WNatInstance } from "../../../../typechain-truffle";
import { TestChainInfo, testChainInfo } from "../../../integration/utils/TestChainInfo";
import { impersonateContract, stopImpersonatingContract } from "../../../utils/contract-test-helpers";
import { AssetManagerInitSettings, newAssetManager } from "../../../utils/fasset/CreateAssetManager";
import { MockChain, MockChainWallet } from "../../../utils/fasset/MockChain";
import { MockFlareDataConnectorClient } from "../../../utils/fasset/MockFlareDataConnectorClient";
import { deterministicTimeIncrease, getTestFile, loadFixtureCopyVars } from "../../../utils/test-helpers";
import { TestFtsos, TestSettingsContracts, createFtsoMock, createTestAgent, createTestCollaterals, createTestContracts, createTestFtsos, createTestSettings } from "../../../utils/test-settings";
import { assertWeb3Equal } from "../../../utils/web3assertions";
import {ethers} from "ethers";

const CollateralPool = artifacts.require("CollateralPool");

contract(`Redemption.sol; ${getTestFile(__filename)}; Redemption basic tests`, async accounts => {
    const governance = accounts[10];
    let assetManagerController = accounts[11];
    let contracts: TestSettingsContracts;
    let assetManager: IIAssetManagerInstance;
    let fAsset: FAssetInstance;
    let wNat: WNatInstance;
    let usdc: ERC20MockInstance;
    let ftsos: TestFtsos;
    let settings: AssetManagerInitSettings;
    let collaterals: CollateralType[];
    let chain: MockChain;
    let chainInfo: TestChainInfo;
    let wallet: MockChainWallet;
    let flareDataConnectorClient: MockFlareDataConnectorClient;
    let attestationProvider: AttestationHelper;
    let collateralPool: CollateralPoolInstance;

    // addresses
    const agentOwner1 = accounts[20];
    const agentOwner2 = accounts[21];
    const underlyingAgent1 = "Agent1";  // addresses on mock underlying chain can be any string, as long as it is unique
    const underlyingAgent2 = "Agent2";
    const minterAddress1 = accounts[30];
    const redeemerAddress1 = accounts[40];
    const redeemerAddress2 = accounts[41];
    const executorAddress1 = accounts[45];
    const executorAddress2 = accounts[46];
    const underlyingMinter1 = "Minter1";
    const underlyingRedeemer1 = "Redeemer1";
    const underlyingRedeemer2 = "Redeemer2";
    const executorFee = toWei(0.1);

    function createAgent(owner: string, underlyingAddress: string, options?: Partial<AgentSettings>) {
        const vaultCollateralToken = options?.vaultCollateralToken ?? usdc.address;
        return createTestAgent({ assetManager, settings, chain, wallet, attestationProvider }, owner, underlyingAddress, vaultCollateralToken, options);
    }

    async function depositAndMakeAgentAvailable(agentVault: AgentVaultInstance, owner: string, fullAgentCollateral: BN = toWei(3e8)) {
        await depositCollateral(owner, agentVault, fullAgentCollateral);
        await agentVault.buyCollateralPoolTokens({ from: owner, value: fullAgentCollateral });  // add pool collateral and agent pool tokens
        await assetManager.makeAgentAvailable(agentVault.address, { from: owner });
    }

    async function depositCollateral(owner: string, agentVault: AgentVaultInstance, amount: BN, token: ERC20MockInstance = usdc) {
        await token.mintAmount(owner, amount);
        await token.approve(agentVault.address, amount, { from: owner });
        await agentVault.depositCollateral(token.address, amount, { from: owner });
    }

    async function updateUnderlyingBlock() {
        const proof = await attestationProvider.proveConfirmedBlockHeightExists(Number(settings.attestationWindowSeconds));
        await assetManager.updateCurrentBlock(proof);
        return toNumber(proof.data.requestBody.blockNumber) + toNumber(proof.data.responseBody.numberOfConfirmations);
    }

    async function mintAndRedeem(agentVault: AgentVaultInstance, chain: MockChain, underlyingMinterAddress: string, minterAddress: string, underlyingRedeemerAddress: string, redeemerAddress: string, updateBlock: boolean, approveCollateralReservation: boolean = false, agentOwner: string = agentOwner1) {
        // minter
        chain.mint(underlyingMinterAddress, toBNExp(10000, 18));
        if (updateBlock) await updateUnderlyingBlock();
        // perform minting
        const lots = 3;
        const agentInfo = await assetManager.getAgentInfo(agentVault.address);
        const crFee = await assetManager.collateralReservationFee(lots);
        const resAg = await assetManager.reserveCollateral(agentVault.address, lots, agentInfo.feeBIPS, ZERO_ADDRESS, [underlyingMinterAddress], { from: minterAddress, value: crFee });
        let crt;
        if (approveCollateralReservation) {
            const args = requiredEventArgs(resAg, 'HandshakeRequired');
            // approve reservation
            const tx1 = await assetManager.approveCollateralReservation(args.collateralReservationId, { from: agentOwner });
            crt = requiredEventArgs(tx1, 'CollateralReserved');
        } else {
            crt = requiredEventArgs(resAg, 'CollateralReserved');
        }
        const paymentAmount = crt.valueUBA.add(crt.feeUBA);
        const txHash = await wallet.addTransaction(underlyingMinterAddress, crt.paymentAddress, paymentAmount, crt.paymentReference);
        const proof = await attestationProvider.provePayment(txHash, underlyingMinterAddress, crt.paymentAddress);
        const res = await assetManager.executeMinting(proof, crt.collateralReservationId, { from: minterAddress });
        const minted = requiredEventArgs(res, 'MintingExecuted');
        // redeemer "buys" f-assets
        await fAsset.transfer(redeemerAddress, minted.mintedAmountUBA, { from: minterAddress });
        // redemption request
        const assetManagerBalance = await web3.eth.getBalance(assetManager.address);
        console.log("Balance asset manager before redeem : ", assetManagerBalance);
        const resR = await assetManager.redeem(lots, underlyingRedeemerAddress, executorAddress1, { from: redeemerAddress, value: executorFee });
        const redemptionRequests = filterEvents(resR, 'RedemptionRequested').map(e => e.args);
        const request = redemptionRequests[0];
        const assetManagerBalanceAfter = await web3.eth.getBalance(assetManager.address);
        console.log("Balance asset manager after redeem : ", ethers.utils.formatEther(assetManagerBalanceAfter));
        return request;
    }

    async function initialize() {
        const ci = chainInfo = testChainInfo.eth;
        contracts = await createTestContracts(governance);
        // save some contracts as globals
        ({ wNat } = contracts);
        usdc = contracts.stablecoins.USDC;
        // create FTSOs for nat, stablecoins and asset and set some price
        ftsos = await createTestFtsos(contracts.ftsoRegistry, ci);
        // create mock chain and attestation provider
        chain = new MockChain(await time.latest());
        wallet = new MockChainWallet(chain);
        flareDataConnectorClient = new MockFlareDataConnectorClient(contracts.fdcHub, contracts.relay, { [ci.chainId]: chain }, 'auto');
        attestationProvider = new AttestationHelper(flareDataConnectorClient, chain, ci.chainId);
        // create asset manager
        collaterals = createTestCollaterals(contracts, ci);
        settings = createTestSettings(contracts, ci, { requireEOAAddressProof: true });
        [assetManager, fAsset] = await newAssetManager(governance, assetManagerController, ci.name, ci.symbol, ci.decimals, settings, collaterals, ci.assetName, ci.assetSymbol);
        return { contracts, wNat, usdc, ftsos, chain, wallet, flareDataConnectorClient, attestationProvider, collaterals, settings, assetManager, fAsset };
    }

    beforeEach(async () => {
        ({ contracts, wNat, usdc, ftsos, chain, wallet, flareDataConnectorClient, attestationProvider, collaterals, settings, assetManager, fAsset } = await loadFixtureCopyVars(initialize));
    });

    it("mint and redeem with address validation - invalid address", async () => {
        const assetManagerStartbalance = await web3.eth.getBalance(assetManager.address);
        console.log("Start balance asset manager : ", assetManagerStartbalance);
        const agentVault = await createAgent(agentOwner1, underlyingAgent1);
        await depositAndMakeAgentAvailable(agentVault, agentOwner1);
        const agentInfo1 = await assetManager.getAgentInfo(agentVault.address);
        const request = await mintAndRedeem(agentVault, chain, underlyingMinter1, minterAddress1, "MY_INVALID_ADDRESS", redeemerAddress1, true);
        const agentInfo2 = await assetManager.getAgentInfo(agentVault.address);
        assert.equal(Number(agentInfo2.freeCollateralLots), Number(agentInfo1.freeCollateralLots) - 2);
        const proof = await attestationProvider.proveAddressValidity(request.paymentAddress);
        assert.isFalse(proof.data.responseBody.isValid);
        const res = await assetManager.rejectInvalidRedemption(proof, request.requestId, { from: agentOwner1 });
        expectEvent(res, 'RedemptionRejected', { requestId: request.requestId, redemptionAmountUBA: request.valueUBA });
        const agentInfo3 = await assetManager.getAgentInfo(agentVault.address);
        assert.equal(Number(agentInfo3.freeCollateralLots), Number(agentInfo1.freeCollateralLots));
        const assetManagerEndbalance = await web3.eth.getBalance(assetManager.address);
        console.log("End balance asset manager : ",  ethers.utils.formatEther(assetManagerEndbalance));
    });
});
```
