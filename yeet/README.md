# Yeet

## Reports by Severity

<details>

<summary>Critical</summary>

* \#41695 \[SC-Critical] StakeV2 leaks user tokens as rewards and eventually will become insolvent.
* \#42725 \[SC-Critical] startUnstake() Reduces Total Supply, but StakingToken Balance in contract Remains Constant, Leading to Inflated accumulatedDeptRewardsYeet()
* \#41365 \[SC-Critical] Vested tokens are counted as accumulated revenue
* \#41521 \[SC-Critical] Unstaked tokens incorrectly counted as rewards during vesting period
* \#41549 \[SC-Critical] users funds can get lost when the executeRewardDistributionYeet function invoked after users unstake
* \#41938 \[SC-Critical] Unstake process manipulation and reward distribution vulnerability
* \#42527 \[SC-Critical] Critical Balance/Supply Desynchronization Leading to Protocol Insolvency and Loss of User Funds
* \#41981 \[SC-Critical] Loss of user funds during unstaking, while under the lockup period
* \#42723 \[SC-Critical] Unstaked Tokens Included in Excess Reward Calculation Can Cause DoS for Unstaking Users
* \#42020 \[SC-Critical] Inaccurate calculation in \`accumulatedDeptRewardsYeet()\` causes double counting of vesting tokens as excess, leading to permanent loss of user funds
* \#42152 \[SC-Critical] \`StakeV2::accumulatedDeptRewardsYeet\` fails to account for pending vesting withdrawals which could cause contract insolvency
* \#41286 \[SC-Critical] \`accumulatedDeptRewardsYeet()\` accounts for tokens under unstaking process
* \#41215 \[SC-Critical] StakeV2: Inconsistencies in totalSupply computation, can lead to protocol insolvency
* \#41345 \[SC-Critical] Calculation of accumulatedDeptRewardsYeet is incorrect lead to user lost of fund
* \#42382 \[SC-Critical] Calling \`StakeV2::executeRewardDistributionYeet\` by manager during an ongoing unstaking period for stakers can result in them being unable to unstake permanently
* \#42443 \[SC-Critical] Vested \`$YEET\` are susceptible of being impossible to unstake
* \#42469 \[SC-Critical] Incorrect computation of excess rewards leads to permanent freezing of user funds
* \#42518 \[SC-Critical] Incorrect handling of total staked funds will lead to protocol insolvency
* \#42682 \[SC-Critical] Loss of funds during the reward distribution in executeRewardDistributionYeet() of StakeV2 contract
* \#41894 \[SC-Critical] Incorrect calculation of deposited rewards yeet leads to Staker's not being able to get their staked amount back
* \#42623 \[SC-Critical] Potential Loss of Staked Tokens During Unstaking, Incorrect calculation of excess tokens in\`accumulatedDeptRewardsYeet\`
* \#41911 \[SC-Critical] Unstake amount can be zapped before user withdrawal
* \#41559 \[SC-Critical] Incorrect Calculation of Accumulated Rewards Due to Unstaked Tokens
* \#42581 \[SC-Critical] Miscalculated Balances Lead to Protocol Insolvency
* \#41831 \[SC-Critical] Miscalculation of excess rewards via external token transfers leads to contract insolvency and incomplete withdrawals
* \#41524 \[SC-Critical] Incorrect Reward Calculation in accumulatedDeptRewardsYeet() Function Leads to Loss of User Funds During Vesting Period
* \#41456 \[SC-Critical] \`executeRewardDistributionYeet\` will count user withdraws as rewards
* \#41974 \[SC-Critical] Reducing \`totalSupply\` in \`startUnstake\` leads to protocol insolvency
* \#42345 \[SC-Critical] Theft of User Funds in executeRewardDistributionYeet During Vesting Period
* \#42123 \[SC-Critical] Insufficient Token Reservation in \`startUnstake\` Leads to Permanent Freezing of Vested Funds
* \#41289 \[SC-Critical] StakeV2 Contract Insolvency Issue
* \#41487 \[SC-Critical] Updates totalSupply before transferring the tokens which causes calculating more reward tokens

</details>

<details>

<summary>High</summary>

* \#42598 \[SC-High] When claiming rewards from \`StakeV2\` left-over debt is sent to \`StakeV2\` instead of the user
* \#41280 \[SC-High] Permanent freezing of yield due to incorrect reward handling in \`StakeV2\` claim functions
* \#42039 \[SC-High] When calling \`StakeV2::claimRewardsInNative()\` surplus $YEET are send to the StakeV2 contract instead of the user
* \#42525 \[SC-High] Misallocation of leftover token1 in StakeV2.claimRewardsInToken0
* \#42732 \[SC-High] Incomplete token return whena user claim his rewards leads to rewards fund loss
* \#42113 \[SC-High] yeetOut function in Zapper.sol sends tokens back to StakeV2 contract instead of user
* \#41432 \[SC-High] Attacker can DoS \`StakeV2\`'s rewards distribution by repeatedly inflating Zapper's approval for whitelisted Kodiak Vault tokens
* \#42532 \[SC-High] Compound function in MoneyBrinter can lead to loss of yield
* \#41633 \[SC-High] Users might lose some of the rewards they’re supposed to get.
* \#42158 \[SC-High] Users can DoS \`Zapper::zapIn\` functionality for a token
* \#42292 \[SC-High] Zapper: Wrong Convertion of Assets in zapOut Functions Leads to Partial Loss of Staking Rewards
* \#41528 \[SC-High] When claiming rewards in native Bera via \`StakeV2.claimRewardsInNative\`, excess \`token0Debt\` or/and \`token1Debt\` is not returned to the kodiak vault but stuck in \`StakeV2\` contract.
* \#41640 \[SC-High] Stuck Rewards in StakeV2 Contract Due to Improper Handling of Leftover Tokens
* \#42718 \[SC-High] zapOut methods in zapper contract incorrectly use \_msgSender() instead of receiver when sending back remainder tokens
* \#42214 \[SC-High] Leftover \`WBERA\` and \`YEET\` sent to \`StakeV2\` instead of to user who is claiming rewards
* \#41644 \[SC-High] \`\_clearUserDebt\` in zapOut function sends the remaining tokens to \`msg.sender\` instead of receiver.
* \#42189 \[SC-High] User rewards incorrectly transferred to \`StakeV2\` instead of claimant
* \#41907 \[SC-High] Unused debt is not send to Reward Claimer
* \#42548 \[SC-High] Remaining token0 and token1 sent from Zapper to StakeV2 will be permanently locked in StakeV2 forever.
* \#41647 \[SC-High] Unused tokens after zapping can be stuck and not entitled to users
* \#41875 \[SC-High] Permanent Lock of User Funds in StakeV2 Due to Incorrect token Debt Handling

</details>

<details>

<summary>Medium</summary>

* \#41895 \[SC-Medium] Potential loss of token0, token1 in the MoneyBrinter contract
* \#41788 \[SC-Medium] High - Yield theft because of compound function design
* \#42355 \[SC-Medium] Compounding can be sandwich attacked
* \#42333 \[SC-Medium] compound MoneyBrinter.sol can be sandwiched to extract value from other depositors
* \#42710 \[SC-Medium] Modulo opation introduces bias during the winning yeet calculation
* \#42553 \[SC-Medium] Sandwich attack on \`MoneyBrinter\_compound\` allows extracting rewards intended for LPs
* \#41270 \[SC-Medium] Harvest timing exploit enables theft of unclaimed yield
* \#41624 \[SC-Medium] Reward sandwich is possible in \`MoneyBrinter\` vault by frontrunning \`compound\`.
* \#42602 \[SC-Medium] Some of the Compounded Reward Island token can be stolen by sandwiching the compound() function call
* \#41638 \[SC-Medium] Sandwich Attack on \`compound()\` Function Allows Value Extraction from Honest Depositors
* \#41526 \[SC-Medium] MoneyBrinter::compound can be vulnerable to sandwich attacks

</details>

<details>

<summary>Low</summary>

* \#41841 \[SC-Low] Risk of Reward Loss and Gain Manipulation Due to Untimely Claims and Reward Cap Adjustments
* \#42166 \[SC-Low] Modification of MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR Leads to Unjust Loss of Promised Rewards for Users
* \#41377 \[SC-Low] Retroactive Reward Cap Manipulation Allows Theft/Loss of Unclaimed Yield
* \#42008 \[SC-Low] Incorrect Application of MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR on Historical Epochs
* \#41283 \[SC-Low] Contract fails to deliver promised returns, due to changed \`MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR\`
* \#42462 \[SC-Low] Potential loss of unclaimed rewards due to updating setting \`MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR\`
* \#42604 \[SC-Low] \`MoneyBrinter\` vault does not conform to ERC4626
* \#42539 \[SC-Low] Incorrect \`maxWithdraw()\` returns lead to user failed withdrawals of returned maximum amount
* \#41886 \[SC-Low] Full or Large WBERA reward collects can be blocked by small amounts
* \#41635 \[SC-Low] MoneyBrinter contract is EIP-4626 incompliant
* \#42407 \[SC-Low] Updating MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR impacts unclaimed rewards of past epochs
* \#41823 \[SC-Low] Changing the reward settings has a retroactive impact
* \#41511 \[SC-Low] The contract calculates the \`minimumYeetPoint\` using the Pot going to the winner instead of the whole Pot.
* \#41664 \[SC-Low] Users may receive fewer rewards due to the change in reward limits

</details>

<details>

<summary>Insight</summary>

* \#41741 \[SC-Insight] Improper Input Validation in zapInNative Leads to Theft of Residual Funds
* \#41876 \[SC-Insight] User may receive boosted values which are non-concave
* \#41758 \[SC-Insight] The code comment to \`BOOSTRAP\_PHASE\_DURATION\` is incorrect
* \#41890 \[SC-Insight] MoneyBrinter vault does not consider Farm's staking cap
* \#41699 \[SC-Insight] Silent Transfer Failures in Native Token Handling
* \#41885 \[SC-Insight] Bypass token whitelist
* \#41952 \[SC-Insight] Reduce storage costs by eliminating stakedTimes in StakeV2::startUnstake
* \#42711 \[SC-Insight] Incorrect Index Handling in \`unstake\` and \`rageQuit\` Leading to Potential Fund Loss
* \#41873 \[SC-Insight] Protocol fee loss due to incorrect fee calculation in MoneyBrinter.sol
* \#42637 \[SC-Insight] When there is sufficient liquidity for executing reward distribution, token swapping should be skipped to avoid slippage loss
* \#41689 \[SC-Insight] Blacklisting a Kodiak vault unintentionally whitelists a previously blacklisted token
* \#41359 \[SC-Insight] Remove Manager of Address 0 is irrelevant and will never be reached
* \#41145 \[SC-Insight] Incorrect Inheritance of Ownership in \`Manager\` Contract Leading to Inconsistent Use of \`Ownable2Step\`
* \#42439 \[SC-Insight] Insight Report for StakeV2 contract
* \#41488 \[SC-Insight] In \`StakeV2.sol\` there exists a critical flaw that allows adversaries to earn more rewards than should be possible for a period of having staked minimal tokens.
* \#41542 \[SC-Insight] The 20% charged as a \`yeetback\` is not considered as part of \`addYeetVolume\` and \`boostedValue\`
* \#42538 \[SC-Insight] Incorrect value in events emitted in StakeV2
* \#41639 \[SC-Insight] Cross-Vault Reward Arbitrage in StakeV2 Allows Yield Theft
* \#41374 \[SC-Insight] Incorrect NFT Boost Value in Lookup Array
* \#41688 \[SC-Insight] Code can be optimized to to save a significant amount of gas.
* \#41419 \[SC-Insight] Miscalculation of \`maxClaimable\` variable leads to users being able to claim too many or too few reward tokens
* \#42351 \[SC-Insight] Yeetback complex rewards system
* \#41570 \[SC-Insight] Yeet Protocol - Code Insights Report
* \#42487 \[SC-Insight] Redundant Slippage Check in \`compound\` Function
* \#41765 \[SC-Insight] Storage slots only set in constructor should be declared \`immutable\`
* \#41682 \[SC-Insight] Code can be optimized to use save a lot of gas.
* \#41659 \[SC-Insight] Previous owner still hold manager role after ownership transfer
* \#41132 \[SC-Insight] NFT Boost Lookup values not adhere to docs
* \#42388 \[SC-Insight] Discrepancy between number of Yeetback winners in contract and documentation
* \#41707 \[SC-Insight] Code differs from documentation in \`Reward::getClaimableAmount\` function
* \#41272 \[SC-Insight] Unnecessary precision loss due to division before multiplication in \`getDistribution()\`
* \#41766 \[SC-Insight] In \`Yeet.sol\`, storage slots only set in constructor should be declared \`immutable\`.
* \#42127 \[SC-Insight] Redundant Fee Calculation in addYeetback() function
* \#42033 \[SC-Insight] MoneyBrinter contract does not consider farm's pausing status
* \#41949 \[SC-Insight] Optimize StakeV2::startUnstake with \`unchecked\` block to reduce gas costs
* \#41291 \[SC-Insight] Winner Selection Vulnerability in Yeetback Contract Allows Multiple Reward for the Same Lucky Winner
* \#41256 \[SC-Insight] Contradictory Documentation and actual function
* \#41660 \[SC-Insight] Yeet will be permanently DOSED if the entropyProvider runs out of randome numbers or gets blacklisted
* \#41492 \[SC-Insight] Incorrect Reward Value Emitted in \`executeRewardDistributionYeet\` Function
* \#41672 \[SC-Insight] Permanent Loss Risk of User Funds Due to Inflexible Function Design in Claim()

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#41695 \[SC-Critical] StakeV2 leaks user tokens as rewards and eventually will become insolvent.
* \#42598 \[SC-High] When claiming rewards from \`StakeV2\` left-over debt is sent to \`StakeV2\` instead of the user
* \#41895 \[SC-Medium] Potential loss of token0, token1 in the MoneyBrinter contract
* \#41788 \[SC-Medium] High - Yield theft because of compound function design
* \#42725 \[SC-Critical] startUnstake() Reduces Total Supply, but StakingToken Balance in contract Remains Constant, Leading to Inflated accumulatedDeptRewardsYeet()
* \#41841 \[SC-Low] Risk of Reward Loss and Gain Manipulation Due to Untimely Claims and Reward Cap Adjustments
* \#41741 \[SC-Insight] Improper Input Validation in zapInNative Leads to Theft of Residual Funds
* \#41876 \[SC-Insight] User may receive boosted values which are non-concave
* \#41758 \[SC-Insight] The code comment to \`BOOSTRAP\_PHASE\_DURATION\` is incorrect
* \#41280 \[SC-High] Permanent freezing of yield due to incorrect reward handling in \`StakeV2\` claim functions
* \#41365 \[SC-Critical] Vested tokens are counted as accumulated revenue
* \#41890 \[SC-Insight] MoneyBrinter vault does not consider Farm's staking cap
* \#41521 \[SC-Critical] Unstaked tokens incorrectly counted as rewards during vesting period
* \#41549 \[SC-Critical] users funds can get lost when the executeRewardDistributionYeet function invoked after users unstake
* \#41699 \[SC-Insight] Silent Transfer Failures in Native Token Handling
* \#41938 \[SC-Critical] Unstake process manipulation and reward distribution vulnerability
* \#42527 \[SC-Critical] Critical Balance/Supply Desynchronization Leading to Protocol Insolvency and Loss of User Funds
* \#41885 \[SC-Insight] Bypass token whitelist
* \#41952 \[SC-Insight] Reduce storage costs by eliminating stakedTimes in StakeV2::startUnstake
* \#41981 \[SC-Critical] Loss of user funds during unstaking, while under the lockup period
* \#42723 \[SC-Critical] Unstaked Tokens Included in Excess Reward Calculation Can Cause DoS for Unstaking Users
* \#42355 \[SC-Medium] Compounding can be sandwich attacked
* \#42039 \[SC-High] When calling \`StakeV2::claimRewardsInNative()\` surplus $YEET are send to the StakeV2 contract instead of the user
* \#42711 \[SC-Insight] Incorrect Index Handling in \`unstake\` and \`rageQuit\` Leading to Potential Fund Loss
* \#41873 \[SC-Insight] Protocol fee loss due to incorrect fee calculation in MoneyBrinter.sol
* \#42525 \[SC-High] Misallocation of leftover token1 in StakeV2.claimRewardsInToken0
* \#42020 \[SC-Critical] Inaccurate calculation in \`accumulatedDeptRewardsYeet()\` causes double counting of vesting tokens as excess, leading to permanent loss of user funds
* \#42732 \[SC-High] Incomplete token return whena user claim his rewards leads to rewards fund loss
* \#42637 \[SC-Insight] When there is sufficient liquidity for executing reward distribution, token swapping should be skipped to avoid slippage loss
* \#42113 \[SC-High] yeetOut function in Zapper.sol sends tokens back to StakeV2 contract instead of user
* \#42152 \[SC-Critical] \`StakeV2::accumulatedDeptRewardsYeet\` fails to account for pending vesting withdrawals which could cause contract insolvency
* \#42166 \[SC-Low] Modification of MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR Leads to Unjust Loss of Promised Rewards for Users
* \#41689 \[SC-Insight] Blacklisting a Kodiak vault unintentionally whitelists a previously blacklisted token
* \#41377 \[SC-Low] Retroactive Reward Cap Manipulation Allows Theft/Loss of Unclaimed Yield
* \#42008 \[SC-Low] Incorrect Application of MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR on Historical Epochs
* \#41283 \[SC-Low] Contract fails to deliver promised returns, due to changed \`MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR\`
* \#41286 \[SC-Critical] \`accumulatedDeptRewardsYeet()\` accounts for tokens under unstaking process
* \#42462 \[SC-Low] Potential loss of unclaimed rewards due to updating setting \`MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR\`
* \#41359 \[SC-Insight] Remove Manager of Address 0 is irrelevant and will never be reached
* \#41215 \[SC-Critical] StakeV2: Inconsistencies in totalSupply computation, can lead to protocol insolvency
* \#41345 \[SC-Critical] Calculation of accumulatedDeptRewardsYeet is incorrect lead to user lost of fund
* \#41145 \[SC-Insight] Incorrect Inheritance of Ownership in \`Manager\` Contract Leading to Inconsistent Use of \`Ownable2Step\`
* \#42333 \[SC-Medium] compound MoneyBrinter.sol can be sandwiched to extract value from other depositors
* \#42382 \[SC-Critical] Calling \`StakeV2::executeRewardDistributionYeet\` by manager during an ongoing unstaking period for stakers can result in them being unable to unstake permanently
* \#41432 \[SC-High] Attacker can DoS \`StakeV2\`'s rewards distribution by repeatedly inflating Zapper's approval for whitelisted Kodiak Vault tokens
* \#42439 \[SC-Insight] Insight Report for StakeV2 contract
* \#41488 \[SC-Insight] In \`StakeV2.sol\` there exists a critical flaw that allows adversaries to earn more rewards than should be possible for a period of having staked minimal tokens.
* \#42443 \[SC-Critical] Vested \`$YEET\` are susceptible of being impossible to unstake
* \#42469 \[SC-Critical] Incorrect computation of excess rewards leads to permanent freezing of user funds
* \#41542 \[SC-Insight] The 20% charged as a \`yeetback\` is not considered as part of \`addYeetVolume\` and \`boostedValue\`
* \#42518 \[SC-Critical] Incorrect handling of total staked funds will lead to protocol insolvency
* \#42604 \[SC-Low] \`MoneyBrinter\` vault does not conform to ERC4626
* \#42532 \[SC-High] Compound function in MoneyBrinter can lead to loss of yield
* \#41633 \[SC-High] Users might lose some of the rewards they’re supposed to get.
* \#42682 \[SC-Critical] Loss of funds during the reward distribution in executeRewardDistributionYeet() of StakeV2 contract
* \#41894 \[SC-Critical] Incorrect calculation of deposited rewards yeet leads to Staker's not being able to get their staked amount back
* \#42623 \[SC-Critical] Potential Loss of Staked Tokens During Unstaking, Incorrect calculation of excess tokens in\`accumulatedDeptRewardsYeet\`
* \#42538 \[SC-Insight] Incorrect value in events emitted in StakeV2
* \#41911 \[SC-Critical] Unstake amount can be zapped before user withdrawal
* \#42539 \[SC-Low] Incorrect \`maxWithdraw()\` returns lead to user failed withdrawals of returned maximum amount
* \#41559 \[SC-Critical] Incorrect Calculation of Accumulated Rewards Due to Unstaked Tokens
* \#42710 \[SC-Medium] Modulo opation introduces bias during the winning yeet calculation
* \#42581 \[SC-Critical] Miscalculated Balances Lead to Protocol Insolvency
* \#41831 \[SC-Critical] Miscalculation of excess rewards via external token transfers leads to contract insolvency and incomplete withdrawals
* \#41524 \[SC-Critical] Incorrect Reward Calculation in accumulatedDeptRewardsYeet() Function Leads to Loss of User Funds During Vesting Period
* \#42553 \[SC-Medium] Sandwich attack on \`MoneyBrinter\_compound\` allows extracting rewards intended for LPs
* \#41270 \[SC-Medium] Harvest timing exploit enables theft of unclaimed yield
* \#41624 \[SC-Medium] Reward sandwich is possible in \`MoneyBrinter\` vault by frontrunning \`compound\`.
* \#41886 \[SC-Low] Full or Large WBERA reward collects can be blocked by small amounts
* \#42158 \[SC-High] Users can DoS \`Zapper::zapIn\` functionality for a token
* \#41456 \[SC-Critical] \`executeRewardDistributionYeet\` will count user withdraws as rewards
* \#41974 \[SC-Critical] Reducing \`totalSupply\` in \`startUnstake\` leads to protocol insolvency
* \#42292 \[SC-High] Zapper: Wrong Convertion of Assets in zapOut Functions Leads to Partial Loss of Staking Rewards
* \#42345 \[SC-Critical] Theft of User Funds in executeRewardDistributionYeet During Vesting Period
* \#42123 \[SC-Critical] Insufficient Token Reservation in \`startUnstake\` Leads to Permanent Freezing of Vested Funds
* \#42602 \[SC-Medium] Some of the Compounded Reward Island token can be stolen by sandwiching the compound() function call
* \#41635 \[SC-Low] MoneyBrinter contract is EIP-4626 incompliant
* \#42407 \[SC-Low] Updating MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR impacts unclaimed rewards of past epochs
* \#41528 \[SC-High] When claiming rewards in native Bera via \`StakeV2.claimRewardsInNative\`, excess \`token0Debt\` or/and \`token1Debt\` is not returned to the kodiak vault but stuck in \`StakeV2\` contract.
* \#41823 \[SC-Low] Changing the reward settings has a retroactive impact
* \#41638 \[SC-Medium] Sandwich Attack on \`compound()\` Function Allows Value Extraction from Honest Depositors
* \#41639 \[SC-Insight] Cross-Vault Reward Arbitrage in StakeV2 Allows Yield Theft
* \#41640 \[SC-High] Stuck Rewards in StakeV2 Contract Due to Improper Handling of Leftover Tokens
* \#42718 \[SC-High] zapOut methods in zapper contract incorrectly use \_msgSender() instead of receiver when sending back remainder tokens
* \#42214 \[SC-High] Leftover \`WBERA\` and \`YEET\` sent to \`StakeV2\` instead of to user who is claiming rewards
* \#41374 \[SC-Insight] Incorrect NFT Boost Value in Lookup Array
* \#41688 \[SC-Insight] Code can be optimized to to save a significant amount of gas.
* \#41644 \[SC-High] \`\_clearUserDebt\` in zapOut function sends the remaining tokens to \`msg.sender\` instead of receiver.
* \#41419 \[SC-Insight] Miscalculation of \`maxClaimable\` variable leads to users being able to claim too many or too few reward tokens
* \#42189 \[SC-High] User rewards incorrectly transferred to \`StakeV2\` instead of claimant
* \#41907 \[SC-High] Unused debt is not send to Reward Claimer
* \#42548 \[SC-High] Remaining token0 and token1 sent from Zapper to StakeV2 will be permanently locked in StakeV2 forever.
* \#41647 \[SC-High] Unused tokens after zapping can be stuck and not entitled to users
* \#41875 \[SC-High] Permanent Lock of User Funds in StakeV2 Due to Incorrect token Debt Handling
* \#42351 \[SC-Insight] Yeetback complex rewards system
* \#41570 \[SC-Insight] Yeet Protocol - Code Insights Report
* \#42487 \[SC-Insight] Redundant Slippage Check in \`compound\` Function
* \#41765 \[SC-Insight] Storage slots only set in constructor should be declared \`immutable\`
* \#41682 \[SC-Insight] Code can be optimized to use save a lot of gas.
* \#41659 \[SC-Insight] Previous owner still hold manager role after ownership transfer
* \#41132 \[SC-Insight] NFT Boost Lookup values not adhere to docs
* \#42388 \[SC-Insight] Discrepancy between number of Yeetback winners in contract and documentation
* \#41707 \[SC-Insight] Code differs from documentation in \`Reward::getClaimableAmount\` function
* \#41272 \[SC-Insight] Unnecessary precision loss due to division before multiplication in \`getDistribution()\`
* \#41766 \[SC-Insight] In \`Yeet.sol\`, storage slots only set in constructor should be declared \`immutable\`.
* \#42127 \[SC-Insight] Redundant Fee Calculation in addYeetback() function
* \#42033 \[SC-Insight] MoneyBrinter contract does not consider farm's pausing status
* \#41949 \[SC-Insight] Optimize StakeV2::startUnstake with \`unchecked\` block to reduce gas costs
* \#41291 \[SC-Insight] Winner Selection Vulnerability in Yeetback Contract Allows Multiple Reward for the Same Lucky Winner
* \#41256 \[SC-Insight] Contradictory Documentation and actual function
* \#41511 \[SC-Low] The contract calculates the \`minimumYeetPoint\` using the Pot going to the winner instead of the whole Pot.
* \#41660 \[SC-Insight] Yeet will be permanently DOSED if the entropyProvider runs out of randome numbers or gets blacklisted
* \#41289 \[SC-Critical] StakeV2 Contract Insolvency Issue
* \#41526 \[SC-Medium] MoneyBrinter::compound can be vulnerable to sandwich attacks
* \#41492 \[SC-Insight] Incorrect Reward Value Emitted in \`executeRewardDistributionYeet\` Function
* \#41672 \[SC-Insight] Permanent Loss Risk of User Funds Due to Inflexible Function Design in Claim()
* \#41664 \[SC-Low] Users may receive fewer rewards due to the change in reward limits
* \#41487 \[SC-Critical] Updates totalSupply before transferring the tokens which causes calculating more reward tokens

</details>
