# Flare FAssets | Mainnet Audit Comp

## Reports by Severity

<details>

<summary>High</summary>

* \#46985 \[SC-High] CollateralPool::totalCollateral can be increased to arbitrary value
* \#47060 \[SC-High] Unchecked Partial Payout on selfCloseExit Allows User Underpayment
* \#46378 \[SC-High] Unconditional F-Asset burn during partial collateral redemptions enables direct theft of user funds
* \#46437 \[SC-High] Agent can circumvent double payment challenge on XRP chain using other types of transaction (Bypass fix of #41764)
* \#46949 \[SC-High] Top-up discount miscalculation allows minting excess pool tokens via repeated small deposits in \`CollateralPool::enter\`
* \#46592 \[SC-High] The return value of redeemFromAgent/redeemFromAgentInCollateral in the selfCloseExitTo is not checked
* \#45893 \[SC-High] Agent role can stolen nat token from protocol users
* \#46121 \[SC-High] Malicious agent can manipulate the totalCollateral to cause damage to the protocol
* \#45979 \[SC-High] Agent can steal funds from FLR holders who have deposited in agent's collateral pool
* \#47108 \[SC-High] selfCloseExitTo() can cause users to receive partial payments without validation, leading to permanent asset loss
* \#46688 \[SC-High] \`claimAirdropDistribution()\` Allows Arbitrary Inflation of \`totalCollateral\`
* \#46858 \[SC-High] The agent owner can exploit a malicious rewardManager to steal tokens from the protocol
* \#47020 \[SC-High] A malicious agent can extract funds from the collateral pool by diluting the value of existing collateral providers' shares.
* \#45904 \[SC-High] Malicious agent can forge a non-payment proof despite user's valid payment and fraudulently trigger \`mintingPaymentDefault\`
* \#46953 \[SC-High] AGENTS WHO CREATE AGENTS WITH PRIOR TRANSACTIONS CAN BE INSTANTLY UNFAIRLY LIQUIDATED
* \#46282 \[SC-High] Wrong implementation of \`payout\` would lead to loss of fee share of \`AgentVault\`
* \#46541 \[SC-High] Historical Payment Transaction Exploitation Leading to Instant Agent Liquidation

</details>

<details>

<summary>Medium</summary>

* \#46929 \[SC-Medium] Incorrect required underlying value check used in mintFromFreeUnderlying function
* \#46943 \[SC-Medium] Agents can prevent user CoreVault redemptions by sandwiching them with a requestReturnFromCoreVault and a cancelReturnFromCoreVault
* \#46247 \[SC-Medium] Token transfer can revert in unstickMinting because of insufficient funds in the vault.
* \#46326 \[SC-Medium] Incorrect Minting Cap Check in Minting Process
* \#45478 \[SC-Medium] Minting Cap Check Doesn't Include \`poolFeeUBA\` in \`selfMint\` and \`mintFromUnderlying\`
* \#45550 \[SC-Medium] \[H-01] \`illegalPaymentChallenge\` is vulnerable to frontrunning by external challengers stealing the reward
* \#45554 \[SC-Medium] Fee loss during Agent's feeBIPS reduction in \`selfMint\` function
* \#45769 \[SC-Medium] Permanent blocking of Agent's fund by allowed minters
* \#46826 \[SC-Medium] Agents can game the system by ensuring they always have \`msg.value > transferFeeWei + Transfers.TRANSFER\_GAS\_ALLOWANCE\` when \`CoreVault::transferToCoreVault()\` is called.
* \#46271 \[SC-Medium] Rewards claiming functionality is broken.
* \#45910 \[SC-Medium] Changing collateral ratio makes Agents prone to liquidation
* \#46265 \[SC-Medium] Logic flaw in transferToCoreVault allows creation of zero-value redemption request
* \#45447 \[SC-Medium] Executor cannot execute minting while the agent can execute the transaction and steal executor fee
* \#45665 \[SC-Medium] \[H-02] Minting Cap Bypass via Pool Fee Exclusion during Self Mint
* \#46714 \[SC-Medium] Agent can frontrun executor to steal unclaimed executor fee in minting process
* \#45514 \[SC-Medium] Malicious agents can trap stakers by raising the exit collateral ratio
* \#47034 \[SC-Medium] check minting cap function checks on incorrect amount in mintFromFreeUnderlying function
* \#45987 \[SC-Medium] A malicious user can fill up the redemption queue with the minimum size (1 lot), making legitimate redeemers to redeem always multiple times
* \#46081 \[SC-Medium] Wrong check in \`redeemFromCoreVault\` will result in unnecessary revert
* \#46108 \[SC-Medium] Minting Cap can by bypassed while self minting
* \#45830 \[SC-Medium] Incorrect amount passed to checkMintingCap in self-minting allows bypassing of config minting cap
* \#47039 \[SC-Medium] \`poolMintFee\` is not considered for or checked against the\`mintingCapAMG\` limits.

</details>

<details>

<summary>Low</summary>

* \#45897 \[SC-Low] Executor Fee Lost in \`rejectInvalidRedemption()\` Due to Missing Handling Logic
* \#47033 \[SC-Low] Incorrect calculation of total available amount in core vault in a certain case when a user redeems from the core vault
* \#47082 \[SC-Low] Zero collateral payout despite burned fAssets
* \#47106 \[SC-Low] Collateral Reservation Fee distribution uses current poolFeeShareBips instead of value stored during during time of collateral reservation
* \#46993 \[SC-Low] Malicious agent with large capital can abuse \`cancelReturnFromCoreVault\` to block access to core vault liquidity during high redemption demand
* \#47053 \[SC-Low] \`transferToCoreVault()\` allows agents to have unbacked synthetic assets by extracting underlying value without burning
* \#47010 \[SC-Low] \`CollateralPool::donateNat\` manipulation enables arbitrary pool‐token value inflation and fee‐debt evasion
* \#46442 \[SC-Low] Agent collateral pool is vulnerable to inflation attack
* \#46486 \[SC-Low] Faulty logic in \`transferToCoreVault\` makes users pay more for the refund transaction than the amount being refunded.
* \#46758 \[SC-Low] Collateral Reservation Fee Calculation Inconsistent with Actual Reserved Value
* \#46520 \[SC-Low] ETH loss on \`selfCloseExitTo\` when redeeming to collateral
* \#46847 \[SC-Low] executor fee is not paid or burned in \`rejectInvalidRedemption\`
* \#46836 \[SC-Low] buybackAgentCollateral will revert due to overflow
* \#46068 \[SC-Low] selfCloseExitTo is lack of slippage protect
* \#46071 \[SC-Low] Ultra-low amount of total shares in collateral pool
* \#46886 \[SC-Low] \`destroyAgent()\` functionality can easily be bricked due to Frontrunning Attack
* \#46462 \[SC-Low] Malicious collateral provider can steal funds from agent collateral pool by donating a large amount of native token to the pool (inflation attack)
* \#46681 \[SC-Low] malicious actor can prevent agent from being destroyed
* \#46976 \[SC-Low] Agent Destruction Can Permanently Lock Unclaimed Transfer Fees
* \#46119 \[SC-Low] Incorrect \`msg.Value\` check in \`CoreVault\` Transfer
* \#46984 \[SC-Low] Incomplete Token Supply Check After Token Share Recalculation in \`\_selfCloseExitTo\`
* \#45533 \[SC-Low] Incorrect gas allowance comparison in CoreVault transfer function leads to user fund loss
* \#46643 \[SC-Low] \`destroyAgent\` in \`AgentsCreateDestroy\` is prone to DOS
* \#45604 \[SC-Low] User Overpayment in \`transferToCoreVault\` Fee Handling
* \#45379 \[SC-Low] Frontrunning Vulnerability in createAgentVault Suffix Reservation
* \#46969 \[SC-Low] Inconsistent Use of poolFeeShareBIPS Between Collateral Reservation and Distribution
* \#46320 \[SC-Low] Executor fee will be stuck in the contract when rejectInvalidRedemption is called
* \#46930 \[SC-Low] \`depositNat()\` in \`CollateralPool\` Fails to Notify Asset Manager, By not calling the \`updateCollateral\`
* \#45499 \[SC-Low] Malicious user can prevent agent to be destroyed and lock up his funds
* \#46838 \[SC-Low] Agent Destruction Can Be Blocked by Malicious Collateral Pool Entries
* \#45943 \[SC-Low] rejectInvalidRedemption fee is not awarded to agent, resulting in stuck or misallocated funds
* \#46924 \[SC-Low] Last user may exit with almost all of his values, but he'll purposefully leave a small 1e18 or a little more to grief \`destroy()\`
* \#46587 \[SC-Low] Overpayment loss in \`transferToCoreVault\` due to incorrect refund condition
* \#45336 \[SC-Low] Malicious Agent could repeatedly create and destroy vaults reserving different suffixes and grief other agents
* \#45439 \[SC-Low] Empty String Allowed as Pool Token Suffix in \_reserveAndValidatePoolTokenSuffix

</details>

<details>

<summary>Insight</summary>

* \#45978 \[SC-Insight] Failed Transactions Trigger Invalid Double Payment Challenges Causing Loss of Funds for Legitimate Agents
* \#47094 \[SC-Insight] Missing Event Emission in \`AgentVault\` and \`CollateralPoolToken\` Factory Contracts
* \#45377 \[SC-Insight] Missing pause modifier in \`beforeCollateralWithdrawal\` allows collateral theft during a pause
* \#47150 \[SC-Insight] XRP Deposit Authorization Griefing Attack on Minting Process
* \#46220 \[SC-Insight] Missing Documented Function in the CollateralPool Contract
* \#45450 \[SC-Insight] Outdated underlying chain data lead to shortened minting windows or DoS when minting fAssets
* \#45674 \[SC-Insight] \`executeMinting()\` allows impersonation of minter during chain-reorg due to deterministic \`crtId\` and lack of minter binding
* \#46702 \[SC-Insight] \`executeMinting()\` Enables Cross-Contract Reentrancy to Manipulate Collateral Pool Pricing
* \#46721 \[SC-Insight] Inconsistencies for agentTimelockedOperationWindowSeconds value checks between SettingsInitializer.sol::\_validateSettings and SettingsManagementFacet.sol::setAgentTimelockedOpera...
* \#45961 \[SC-Insight] \`selfMint()\` Can Lead to Permanent Loss of Agents' Funds During Emergency Pause
* \#46848 \[SC-Insight] Minters can grief agents by deliberately fragmenting the agent's redemption ticket queue with minimal size tickets, preventing or delaying large transfers to core vault
* \#45357 \[SC-Insight] Increase in the usedTokens array
* \#46210 \[SC-Insight] Incorrect timestamp comparison in function "beforeCollateralWithdrawal" allows agent to withdraw at last second without being challenged
* \#46534 \[SC-Insight] Missing Validation to Prevent Self-Assignment of Work Address
* \#45956 \[SC-Insight] EOA only on smart contract chains bypassed on ETH
* \#46218 \[SC-Insight] Documentation-Implementation Discrepancy in Agent Vault Access Control
* \#45485 \[SC-Insight] Comments above \`reserveCollateral\` indicate collateral reservation fee is burned, which is not the case
* \#46999 \[SC-Insight] Absence of event emission in critical functions
* \#47087 \[SC-Insight] CollateralTypesFacet.sol::deprecateCollateralType allows to break CollateralTypes.sol::initialize invariant because it allows to deprecate all token collateral vaults leading to ...
* \#46546 \[SC-Insight] Accounting Mismatches in AgentVault.sol Due to Non-Standard ERC20 Tokens
* \#45368 \[SC-Insight] Corruptible Upgradability Pattern
* \#46122 \[SC-Insight] Incorrect Minimum Lots Validation in CoreVault Redemption
* \#47116 \[SC-Insight] Undocumented Redemption Pool Fee Share potentially leading to confusion
* \#45864 \[SC-Insight] Minter's underlying token can get stuck if the agent calls mintingDefault before the minter’s transaction is recorded on the underlying blockchain.
* \#45813 \[SC-Insight] Missing \`setAutoClaiming\` Function
* \#47121 \[SC-Insight] Incorrect documentation on pool Top-up feature
* \#45309 \[SC-Insight] Gas Optimization in \`\_burnForAtNow\` Function for efficient balance retrieval
* \#46266 \[SC-Insight] Cannot use a pool token suffix of MAX\_SUFFIX\_LEN
* \#46241 \[SC-Insight] Misleading definition in Core-Vault documentation (“CV operators submit proof”)
* \#45949 \[SC-Insight] Mismatch between doc and implementation for \`confirmationByOthersAfterSeconds\` minimum on XRP
* \#46493 \[SC-Insight] ADDRESS\_STORAGE\_POSITION is not ERC7201 compliant
* \#46311 \[SC-Insight] Unbacked Redemptions Due to Donation- Attack on CoreVault Can Freeze Agent Collateral
* \#46198 \[SC-Insight] Redemption Blocked if Agent Refuses to Confirm Core Vault Payment
* \#45405 \[SC-Insight] Insufficient Documentation for Governance-Controlled Functions and Critical Parameters in 'CoreVaultManager.sol'
* \#45772 \[SC-Insight] NatSpec Mismatch in CoreVault Redemption Logic
* \#47091 \[SC-Insight] \`setWorkAddress()\` enables front-running attacks to hijack work addresses
* \#47159 \[SC-Insight] Lack of Access Control on \`triggerInstructions()\` Allows Unauthorized Transfers Post-Deletion
* \#45685 \[SC-Insight] Incorrect comments in finishRedemptionWithoutPayment
* \#45517 \[SC-Insight] Partial Documentation for Self-Close Exit Fee Handling and Redemption Workflow in 'CollateralPool.sol'
* \#45574 \[SC-Insight] Redundant Per‑Item Upper Bound Check in \`validateLiquidationFactors\`
* \#46092 \[SC-Insight] AgentVault::destroy mismatch between comment documentation and contract behavior
* \#45310 \[SC-Insight] \`IWNat(address(token)).governanceVotePower().undelegate()\` is redundant after \`undelegateGovernance()\`
* \#46677 \[SC-Insight] Wrong comment in \_getFAssetRequiredToNotSpoilCR
* \#46982 \[SC-Insight] Spread calculation discrepancy allows wildly divergent prices to be accepted
* \#46771 \[SC-Insight] Incorrect Collateral Ratio Check Due to Rounding Error
* \#45731 \[SC-Insight] Off-by-One Logic in Escrow End Timestamp Calculation May Cause Unintended Escrow Delay

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#46929 \[SC-Medium] Incorrect required underlying value check used in mintFromFreeUnderlying function
* \#46985 \[SC-High] CollateralPool::totalCollateral can be increased to arbitrary value
* \#45897 \[SC-Low] Executor Fee Lost in \`rejectInvalidRedemption()\` Due to Missing Handling Logic
* \#46943 \[SC-Medium] Agents can prevent user CoreVault redemptions by sandwiching them with a requestReturnFromCoreVault and a cancelReturnFromCoreVault
* \#47033 \[SC-Low] Incorrect calculation of total available amount in core vault in a certain case when a user redeems from the core vault
* \#47060 \[SC-High] Unchecked Partial Payout on selfCloseExit Allows User Underpayment
* \#47082 \[SC-Low] Zero collateral payout despite burned fAssets
* \#45978 \[SC-Insight] Failed Transactions Trigger Invalid Double Payment Challenges Causing Loss of Funds for Legitimate Agents
* \#47094 \[SC-Insight] Missing Event Emission in \`AgentVault\` and \`CollateralPoolToken\` Factory Contracts
* \#47106 \[SC-Low] Collateral Reservation Fee distribution uses current poolFeeShareBips instead of value stored during during time of collateral reservation
* \#46247 \[SC-Medium] Token transfer can revert in unstickMinting because of insufficient funds in the vault.
* \#46993 \[SC-Low] Malicious agent with large capital can abuse \`cancelReturnFromCoreVault\` to block access to core vault liquidity during high redemption demand
* \#47053 \[SC-Low] \`transferToCoreVault()\` allows agents to have unbacked synthetic assets by extracting underlying value without burning
* \#46326 \[SC-Medium] Incorrect Minting Cap Check in Minting Process
* \#47010 \[SC-Low] \`CollateralPool::donateNat\` manipulation enables arbitrary pool‐token value inflation and fee‐debt evasion
* \#45377 \[SC-Insight] Missing pause modifier in \`beforeCollateralWithdrawal\` allows collateral theft during a pause
* \#47150 \[SC-Insight] XRP Deposit Authorization Griefing Attack on Minting Process
* \#46378 \[SC-High] Unconditional F-Asset burn during partial collateral redemptions enables direct theft of user funds
* \#46220 \[SC-Insight] Missing Documented Function in the CollateralPool Contract
* \#45450 \[SC-Insight] Outdated underlying chain data lead to shortened minting windows or DoS when minting fAssets
* \#46437 \[SC-High] Agent can circumvent double payment challenge on XRP chain using other types of transaction (Bypass fix of #41764)
* \#45478 \[SC-Medium] Minting Cap Check Doesn't Include \`poolFeeUBA\` in \`selfMint\` and \`mintFromUnderlying\`
* \#46442 \[SC-Low] Agent collateral pool is vulnerable to inflation attack
* \#45550 \[SC-Medium] \[H-01] \`illegalPaymentChallenge\` is vulnerable to frontrunning by external challengers stealing the reward
* \#46486 \[SC-Low] Faulty logic in \`transferToCoreVault\` makes users pay more for the refund transaction than the amount being refunded.
* \#45554 \[SC-Medium] Fee loss during Agent's feeBIPS reduction in \`selfMint\` function
* \#45674 \[SC-Insight] \`executeMinting()\` allows impersonation of minter during chain-reorg due to deterministic \`crtId\` and lack of minter binding
* \#46702 \[SC-Insight] \`executeMinting()\` Enables Cross-Contract Reentrancy to Manipulate Collateral Pool Pricing
* \#45769 \[SC-Medium] Permanent blocking of Agent's fund by allowed minters
* \#46721 \[SC-Insight] Inconsistencies for agentTimelockedOperationWindowSeconds value checks between SettingsInitializer.sol::\_validateSettings and SettingsManagementFacet.sol::setAgentTimelockedOpera...
* \#46758 \[SC-Low] Collateral Reservation Fee Calculation Inconsistent with Actual Reserved Value
* \#46949 \[SC-High] Top-up discount miscalculation allows minting excess pool tokens via repeated small deposits in \`CollateralPool::enter\`
* \#46520 \[SC-Low] ETH loss on \`selfCloseExitTo\` when redeeming to collateral
* \#45961 \[SC-Insight] \`selfMint()\` Can Lead to Permanent Loss of Agents' Funds During Emergency Pause
* \#46826 \[SC-Medium] Agents can game the system by ensuring they always have \`msg.value > transferFeeWei + Transfers.TRANSFER\_GAS\_ALLOWANCE\` when \`CoreVault::transferToCoreVault()\` is called.
* \#46848 \[SC-Insight] Minters can grief agents by deliberately fragmenting the agent's redemption ticket queue with minimal size tickets, preventing or delaying large transfers to core vault
* \#45357 \[SC-Insight] Increase in the usedTokens array
* \#46847 \[SC-Low] executor fee is not paid or burned in \`rejectInvalidRedemption\`
* \#46836 \[SC-Low] buybackAgentCollateral will revert due to overflow
* \#46210 \[SC-Insight] Incorrect timestamp comparison in function "beforeCollateralWithdrawal" allows agent to withdraw at last second without being challenged
* \#46271 \[SC-Medium] Rewards claiming functionality is broken.
* \#46592 \[SC-High] The return value of redeemFromAgent/redeemFromAgentInCollateral in the selfCloseExitTo is not checked
* \#46534 \[SC-Insight] Missing Validation to Prevent Self-Assignment of Work Address
* \#45956 \[SC-Insight] EOA only on smart contract chains bypassed on ETH
* \#46218 \[SC-Insight] Documentation-Implementation Discrepancy in Agent Vault Access Control
* \#45485 \[SC-Insight] Comments above \`reserveCollateral\` indicate collateral reservation fee is burned, which is not the case
* \#46068 \[SC-Low] selfCloseExitTo is lack of slippage protect
* \#46071 \[SC-Low] Ultra-low amount of total shares in collateral pool
* \#45910 \[SC-Medium] Changing collateral ratio makes Agents prone to liquidation
* \#46886 \[SC-Low] \`destroyAgent()\` functionality can easily be bricked due to Frontrunning Attack
* \#45893 \[SC-High] Agent role can stolen nat token from protocol users
* \#46265 \[SC-Medium] Logic flaw in transferToCoreVault allows creation of zero-value redemption request
* \#46121 \[SC-High] Malicious agent can manipulate the totalCollateral to cause damage to the protocol
* \#46462 \[SC-Low] Malicious collateral provider can steal funds from agent collateral pool by donating a large amount of native token to the pool (inflation attack)
* \#46999 \[SC-Insight] Absence of event emission in critical functions
* \#47087 \[SC-Insight] CollateralTypesFacet.sol::deprecateCollateralType allows to break CollateralTypes.sol::initialize invariant because it allows to deprecate all token collateral vaults leading to ...
* \#46546 \[SC-Insight] Accounting Mismatches in AgentVault.sol Due to Non-Standard ERC20 Tokens
* \#45368 \[SC-Insight] Corruptible Upgradability Pattern
* \#46681 \[SC-Low] malicious actor can prevent agent from being destroyed
* \#46122 \[SC-Insight] Incorrect Minimum Lots Validation in CoreVault Redemption
* \#45979 \[SC-High] Agent can steal funds from FLR holders who have deposited in agent's collateral pool
* \#47116 \[SC-Insight] Undocumented Redemption Pool Fee Share potentially leading to confusion
* \#46976 \[SC-Low] Agent Destruction Can Permanently Lock Unclaimed Transfer Fees
* \#46119 \[SC-Low] Incorrect \`msg.Value\` check in \`CoreVault\` Transfer
* \#45447 \[SC-Medium] Executor cannot execute minting while the agent can execute the transaction and steal executor fee
* \#46984 \[SC-Low] Incomplete Token Supply Check After Token Share Recalculation in \`\_selfCloseExitTo\`
* \#45533 \[SC-Low] Incorrect gas allowance comparison in CoreVault transfer function leads to user fund loss
* \#45864 \[SC-Insight] Minter's underlying token can get stuck if the agent calls mintingDefault before the minter’s transaction is recorded on the underlying blockchain.
* \#47108 \[SC-High] selfCloseExitTo() can cause users to receive partial payments without validation, leading to permanent asset loss
* \#46643 \[SC-Low] \`destroyAgent\` in \`AgentsCreateDestroy\` is prone to DOS
* \#45813 \[SC-Insight] Missing \`setAutoClaiming\` Function
* \#46688 \[SC-High] \`claimAirdropDistribution()\` Allows Arbitrary Inflation of \`totalCollateral\`
* \#47121 \[SC-Insight] Incorrect documentation on pool Top-up feature
* \#45665 \[SC-Medium] \[H-02] Minting Cap Bypass via Pool Fee Exclusion during Self Mint
* \#45309 \[SC-Insight] Gas Optimization in \`\_burnForAtNow\` Function for efficient balance retrieval
* \#45604 \[SC-Low] User Overpayment in \`transferToCoreVault\` Fee Handling
* \#46714 \[SC-Medium] Agent can frontrun executor to steal unclaimed executor fee in minting process
* \#46858 \[SC-High] The agent owner can exploit a malicious rewardManager to steal tokens from the protocol
* \#45379 \[SC-Low] Frontrunning Vulnerability in createAgentVault Suffix Reservation
* \#46266 \[SC-Insight] Cannot use a pool token suffix of MAX\_SUFFIX\_LEN
* \#46241 \[SC-Insight] Misleading definition in Core-Vault documentation (“CV operators submit proof”)
* \#45949 \[SC-Insight] Mismatch between doc and implementation for \`confirmationByOthersAfterSeconds\` minimum on XRP
* \#46493 \[SC-Insight] ADDRESS\_STORAGE\_POSITION is not ERC7201 compliant
* \#47020 \[SC-High] A malicious agent can extract funds from the collateral pool by diluting the value of existing collateral providers' shares.
* \#45514 \[SC-Medium] Malicious agents can trap stakers by raising the exit collateral ratio
* \#46311 \[SC-Insight] Unbacked Redemptions Due to Donation- Attack on CoreVault Can Freeze Agent Collateral
* \#46198 \[SC-Insight] Redemption Blocked if Agent Refuses to Confirm Core Vault Payment
* \#45405 \[SC-Insight] Insufficient Documentation for Governance-Controlled Functions and Critical Parameters in 'CoreVaultManager.sol'
* \#47034 \[SC-Medium] check minting cap function checks on incorrect amount in mintFromFreeUnderlying function
* \#45772 \[SC-Insight] NatSpec Mismatch in CoreVault Redemption Logic
* \#47091 \[SC-Insight] \`setWorkAddress()\` enables front-running attacks to hijack work addresses
* \#47159 \[SC-Insight] Lack of Access Control on \`triggerInstructions()\` Allows Unauthorized Transfers Post-Deletion
* \#45685 \[SC-Insight] Incorrect comments in finishRedemptionWithoutPayment
* \#45517 \[SC-Insight] Partial Documentation for Self-Close Exit Fee Handling and Redemption Workflow in 'CollateralPool.sol'
* \#45574 \[SC-Insight] Redundant Per‑Item Upper Bound Check in \`validateLiquidationFactors\`
* \#46092 \[SC-Insight] AgentVault::destroy mismatch between comment documentation and contract behavior
* \#45310 \[SC-Insight] \`IWNat(address(token)).governanceVotePower().undelegate()\` is redundant after \`undelegateGovernance()\`
* \#46677 \[SC-Insight] Wrong comment in \_getFAssetRequiredToNotSpoilCR
* \#46982 \[SC-Insight] Spread calculation discrepancy allows wildly divergent prices to be accepted
* \#46771 \[SC-Insight] Incorrect Collateral Ratio Check Due to Rounding Error
* \#45731 \[SC-Insight] Off-by-One Logic in Escrow End Timestamp Calculation May Cause Unintended Escrow Delay
* \#45987 \[SC-Medium] A malicious user can fill up the redemption queue with the minimum size (1 lot), making legitimate redeemers to redeem always multiple times
* \#46081 \[SC-Medium] Wrong check in \`redeemFromCoreVault\` will result in unnecessary revert
* \#46969 \[SC-Low] Inconsistent Use of poolFeeShareBIPS Between Collateral Reservation and Distribution
* \#46320 \[SC-Low] Executor fee will be stuck in the contract when rejectInvalidRedemption is called
* \#46108 \[SC-Medium] Minting Cap can by bypassed while self minting
* \#45830 \[SC-Medium] Incorrect amount passed to checkMintingCap in self-minting allows bypassing of config minting cap
* \#45904 \[SC-High] Malicious agent can forge a non-payment proof despite user's valid payment and fraudulently trigger \`mintingPaymentDefault\`
* \#46930 \[SC-Low] \`depositNat()\` in \`CollateralPool\` Fails to Notify Asset Manager, By not calling the \`updateCollateral\`
* \#45499 \[SC-Low] Malicious user can prevent agent to be destroyed and lock up his funds
* \#46838 \[SC-Low] Agent Destruction Can Be Blocked by Malicious Collateral Pool Entries
* \#45943 \[SC-Low] rejectInvalidRedemption fee is not awarded to agent, resulting in stuck or misallocated funds
* \#47039 \[SC-Medium] \`poolMintFee\` is not considered for or checked against the\`mintingCapAMG\` limits.
* \#46953 \[SC-High] AGENTS WHO CREATE AGENTS WITH PRIOR TRANSACTIONS CAN BE INSTANTLY UNFAIRLY LIQUIDATED
* \#46924 \[SC-Low] Last user may exit with almost all of his values, but he'll purposefully leave a small 1e18 or a little more to grief \`destroy()\`
* \#46587 \[SC-Low] Overpayment loss in \`transferToCoreVault\` due to incorrect refund condition
* \#45336 \[SC-Low] Malicious Agent could repeatedly create and destroy vaults reserving different suffixes and grief other agents
* \#46282 \[SC-High] Wrong implementation of \`payout\` would lead to loss of fee share of \`AgentVault\`
* \#45439 \[SC-Low] Empty String Allowed as Pool Token Suffix in \_reserveAndValidatePoolTokenSuffix
* \#46541 \[SC-High] Historical Payment Transaction Exploitation Leading to Instant Agent Liquidation

</details>
