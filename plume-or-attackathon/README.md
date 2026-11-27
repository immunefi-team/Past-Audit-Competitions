# Plume | Attackathon

## Reports by Severity

<details>

<summary>Critical</summary>

* \#53022 \[SC-Critical] Funds are not properly refunded to user which calls for swap on the dex aggregator
* \#49863 \[SC-Critical] Dex Aggregator ERC20 token theft
* \#51352 \[SC-Critical] User will lose the unspent amount when executing partial swaps via 1inch
* \#53037 \[SC-Critical] Commission changes can retroactively affect user rewards
* \#53011 \[SC-Critical] Uncleaned Partial Approval Consumption in DEX Aggregator Integration Leads to Permanent DoS
* \#51847 \[SC-Critical] DoS via dust leftover in erc-20 approvals
* \#49854 \[SC-Critical] Dex Aggregator partial fill token loss
* \#51283 \[SC-Critical] Permanent Freeze of User token Due to Unhandled Partial Fill Refunds for swap via 1inch in `DexAggregatorWrapperWithPredicateProxy`
* \#52923 \[SC-Critical] Partial fill traps source token residual inside the wrapper and leaves unsafe residual allowance
* \#52178 \[SC-Critical] User will lose the unspent amount when executing partial swaps via OkxRouter
* \#52980 \[SC-Critical] Partial fills strand source tokens in the wrapper and leave dangerous residual allowances

</details>

<details>

<summary>High</summary>

* \#51946 \[SC-High] Commission Claims Fail for Removed Reward Tokens
* \#52964 \[SC-High] if a new reward token is added during a the period a validator is inactive, the validator will still earn rewards/commission for some of the duration in which they were inactive
* \#53020 \[SC-High] There are functions which when inevitably used could result in wrongly accruing yield for inactive validators, which can make the protocol insolvent
* \#52634 \[SC-High] Batch yield distribution has a mathematical flaw that enables economic manipulation
* \#50784 \[SC-High] Any arc token creator can upgrade the implementation
* \#51912 \[SC-High] Mismatched rounding rules in Reward Logic library results in two-fold loss of earnings
* \#50822 \[SC-High] Deployer can cpgrade `ArcToken` to malicious implementation and steal all user funds
* \#50477 \[SC-High] Validator loses all accrued commission when reward token is removed
* \#53034 \[SC-High] ArcTokenFactory doesn't properly handle role management which allows users to arbitrary upgrade their ArcToken's implementation
* \#50735 \[SC-High] some yield tokens will be stuck in contract due to incorrect 'lastProcessedIndex' calculation
* \#51754 \[SC-High] Double yield distribution via token transfers between distributeYieldWithLimit() calls
* \#50450 \[SC-High] Logic error in streak validation causes legitimate jackpot wins to be denied, violating reward contract expectations
* \#51558 \[SC-High] ArcToken holder can receive yield twice from `distributeYieldWithLimit`
* \#52955 \[SC-High] A commission rate checkpoint is not created when adding a validator, despite the commission rate being set leading to loss of validator commission.
* \#50860 \[SC-High] Logic Error in Jackpot Eligibility Check Leads to Systematic Theft of User Rewards
* \#52286 \[SC-High] Off-by-One Error in Jackpot Eligibility Check Leads to Denial of Legitimate Rewards
* \#51456 \[SC-High] Token creator can revoke the UPGRADER role from the factory in order to avoid upgrades
* \#52371 \[SC-High] `distributeYieldWithLimit` is vulnerable to inter-batch `balance and holders array` mutations
* \#51589 \[SC-High] TokenCreator retains upgrade rights – Fix remains insufficient - Finding #01: Immunefi Report
* \#53061 \[SC-High] Asymmetric rounding in commission (ceil for users, floor for validators) enables per-segment rounding loss; validators can amplify via frequent commission checkpoints.
* \#49616 \[SC-High] User can steal Rewards
* \#52165 \[SC-High] user can't claim reward ERC20 tokens since rewards transfer will revert
* \#52218 \[SC-High] Creator Retains DEFAULT\_ADMIN\_ROLE, Allowing Bypass of Upgrade Restrictions
* \#52499 \[SC-High] ArcToken Factory's admin cannot upgrade an ArcToken
* \#49731 \[SC-High] Theft on Re-Added Tokens
* \#52833 \[SC-High] bypass the fix of immunefi audit IMM-CRIT-01 ：Token Creator Can Upgrade ArcToken Implementation
* \#51479 \[SC-High] Inaccurate Reward Calculation Post-Validator Slashing Due to Premature Timestamp Update on Token Removal
* \#52983 \[SC-High] Validator will loose commission for the tokens which are removed from the reward tokens but they still have commission left to be claimed.
* \#52527 \[SC-High] The validator admin might claim less commission token when `ValidatorFacet.requestCommissionClaim` is called.
* \#50350 \[SC-High] StakingFacet: stakeOnBehalf allows to prevent withdraws
* \#53001 \[SC-High] Yield tokens become stuck in ArcTokenPurchase contract when distributing yield during active sales
* \#52803 \[SC-High] \_canRecoverFromCooldown is inconsistent when slash and cooldown maturity occur in the same block
* \#52464 \[SC-High] Commission rounding mismatch under payment bug
* \#52390 \[SC-High] `_validateIsToken(...)` blocks validators from claiming earned rewards from removed tokens.
* \#49939 \[SC-High] Initial timestamp mismatch might lead to users being able to spin twice in the same day
* \#50490 \[SC-High] User loses reward tokens during validator-user relationship clearing
* \#53051 \[SC-High] Unconsented stakeOnBehalf enables third-party gas-griefing DoS by bloating userValidators, breaking withdraw/claimAll
* \#53025 \[SC-High] Commission on removed tokens is unclaimable
* \#53028 \[SC-High] There is an Asymmetric Rounding issue that is can cause a Theft of Unclaimed Yield in Reward or Commission Accounting
* \#50527 \[SC-High] Attacker can steal yield during batch distribution
* \#51051 \[SC-High] Inactive Validator Reward Accrual Bypass
* \#51992 \[SC-High] Dust Accumulation in ArcToken during Yield Distribution.
* \#52865 \[SC-High] Inconsistency in how stake cooldown is handled due to off by one error
* \#52931 \[SC-High] Validators can not claim their commissions after the reward token removal.
* \#52798 \[SC-High] Integer Division Remainder Loss in Batched Yield Distribution Causes Permanent Fund Lock
* \#52995 \[SC-High] Validators lose access to historical reward tokens when tokens are removed
* \#52409 \[SC-High] Asymmetric commission rounding creates systematic accounting drift
* \#51878 \[SC-High] Timing Misalignment Between Campaign Days and Calendar Days Allows Double Spinning on High-Probability Jackpot Days
* \#49700 \[SC-High] Validator Commission can be Blocked
* \#52424 \[SC-High] There is a Retroactive Commission Miscalculation in PlumeRewardLogic
* \#52961 \[SC-High] Theft of yield from the distributor.
* \#52254 \[SC-High] ArcToken theft beyond unclaimed yield during distribution
* \#53018 \[SC-High] Owed rewards could be lost for some users for periods before slashing time due to incorrect logic.
* \#52517 \[SC-High] Missing Point-in-Time Snapshot in Batched Yield Distribution Enables Double-Claims and Permanent Fund Lock
* \#53016 \[SC-High] ArcTokenPurchase doesn't allow RWA-token owners to recover accrued yield from stored ArcTokens waiting for sale.
* \#51041 \[SC-High] Streak‑Count Misuse in Jackpot Eligibility Allows Theft of User Funds
* \#51999 \[SC-High] Logical Flaw in Validator Reactivation and addRewardToken Allows Claiming Rewards for Validators in Inactive Periods
* \#52996 \[SC-High] Users can claim rewards for newly added reward tokens even when the validator they staked for was inactive during some time interval.
* \#50252 \[SC-High] Rounding excess yield tokens become permanently stuck when last holder is yield-restricted
* \#52285 \[SC-High] Incorrect Dust Handling in Yield Distribution Leads to Permanent Fund Lock
* \#50951 \[SC-High] Inconsistent streak count usage between jackpot and raffle ticket calculations
* \#52576 \[SC-High] Flaw in Raffle::determineReward in Jackpot Prize Calculation after week 12
* \#52667 \[SC-High] commission is not added at point of adding validator hence stakers that stake before the first checkpoint would always use the current commission
* \#51961 \[SC-High] Attackers can deny commission rewards to validators by repeatedly calling forceSettleValidatorCommission()
* \#52770 \[SC-High] Unbounded Gas Consumption via stakeOnBehalf Manipulation
* \#50412 \[SC-High] Illegitimate Reward Claim After Unstake Due to Overlapping Reward Rate Checkpoints
* \#51414 \[SC-High] Attacker can drain yield by transferring tokens to other address in yield batch distributions
* \#52847 \[SC-High] No function to recover the remained yield by distributeYieldWithLimit
* \#52278 \[SC-High] Incorrect Streak Check in Jackpot Eligibility Leads to Unfair Reward Denial
* \#50167 \[SC-High] Retroactive reward drain via incomplete reward debt reset
* \#50924 \[SC-High] Validators are not able to claim their accrued commission when the reward token is removed.
* \#52104 \[SC-High] Removed reward tokens block validator commission claims
* \#50409 \[SC-High] Validator will lose comission
* \#51987 \[SC-High] Validators will be able to steal more commission from users that isn't the commission to be charged
* \#52572 \[SC-High] A legitimate arc token holder can be denied his yield.
* \#52680 \[SC-High] holders length changing when distributing limit with limit could lead to case where new holders unfairly claim yield and yield is permanently frozen
* \#53077 \[SC-High] Permanent Fund Lock Due to Flawed Remainder Logic in distributeYield
* \#52449 \[SC-High] Broken Streaks Still Pass Jackpot Eligibility in Spin Contract
* \#52986 \[SC-High] Jackpot check uses previous `streakCount` instead of current computed streak, denying jackpot on first eligible day
* \#52601 \[SC-High] In Spin::handleRandomness() jackpot eligibility uses outdated streakCount instead of updated streak
* \#52500 \[SC-High] Missing Commission Checkpoint Initialization Leads to Retroactive Commission Theft of User Rewards
* \#51324 \[SC-High] Rounding in commission accounting burns delegator rewards
* \#52787 \[SC-High] Batched yield distribution rounding in ArcToken permanently freezes unclaimed funds and misreports payouts
* \#51653 \[SC-High] Permanent loss of staker rewards after slashing when validator records are cleared
* \#53039 \[SC-High] Rewards and commissions accrued in the interval before a slash might be lost
* \#51197 \[SC-High] Arc Token owner can take upgrader role for themselves lockout the factory and upgrade the contract without the knowledge of the factory
* \#53043 \[SC-High] handleRandomness doesn't properly account for current streak which could result in the User spinning losing a Jackpot
* \#52780 \[SC-High] Timestamp Manipulation in forceSettleValidatorCommission Leads to Permanent Loss of Staker Rewards
* \#52889 \[SC-High] Inactive validators accrue rewards for new tokens
* \#52560 \[SC-High] Incorrect current streak used when calculating whether the jackpot should be awarded or not
* \#50796 \[SC-High] Jackpot eligibility uses stale streak
* \#51090 \[SC-High] malicious user can steal yields when `ArcToken.distributeYieldWithLimit` is used.
* \#52573 \[SC-High] Unconsented `stakeOnBehalf` enables **unbounded gas consumption** via `userValidators[]` growth, causing DoS at scale in `claimAll()` / `withdraw()`
* \#50519 \[SC-High] RewardsFacet: Reintroducing an old reward token will result in wrong accounting, leading to theft of yield
* \#50560 \[SC-High] Inconsistent Commission Rounding Traps User/Validator Funds
* \#51124 \[SC-High] Validator would loss commission fee if the rewards token are removed
* \#52849 \[SC-High] Claimers who claim after (slash/inactive + updateRewardPerTokenForValidator which advances `validatorLastUpdateTimes to be more than slashTimestamp`) will lose rewards for a segment
* \#52956 \[SC-High] State Inconsistency in Batched Yield Distribution Leads to Direct Theft of User Funds and Protocol Insolvency
* \#51941 \[SC-High] Token creator can revoke factory's upgrade capability, permanently blocking upgrades
* \#49710 \[SC-High] Cross-batch state manipulation in yield distribution allows double-dipping of yield funds
* \#49787 \[SC-High] Batched Yield Distribution Doesn't Account For Transfers/Purchases Between Batches
* \#51218 \[SC-High] Oracle callback timing vulnerability causes jackpot prize loss
* \#50275 \[SC-High] Eligible user loses Jackpot
* \#51530 \[SC-High] Validators can not Claim Pending Accrued Commission when Reward tokens have been removed from the isRewardToken mapping
* \#51658 \[SC-High] Yield distribution in batches let the same tokens collect rewards in multiple batches, stealing yield from other users
* \#52347 \[SC-High] Improper handling of yield distribution state in distributeYieldWithLimit() leads to revert, freezing users' yield
* \#52711 \[SC-High] In ValidatorFacet, validator cannot claims commissions of removed tokens
* \#50425 \[SC-High] Active non-slashed validators cannot claim rewards when a reward token is disabled
* \#53072 \[SC-High] Ceil-vs-Floor Rounding Mismatch Causes Systematic Underpayment and Unclaimed Yield Leakage
* \#51369 \[SC-High] Unbounded iteration gas‑dos in `_validateTokenForClaim`
* \#51994 \[SC-High] Permanent Loss of Validator Commission Upon Reward Token Removal
* \#50433 \[SC-High] Validator List Griefing: Unrestricted stakeOnBehalf allows User Asset freeze permanently
* \#51813 \[SC-High] Malicious User Can Grief Victims by Staking Them Across Many Validators Leading to Fund Freezing
* \#51452 \[SC-High] stakeOnBehalf() function enables out-of-gas DoS
* \#52944 \[SC-High] The `requestCommisionClaim` function can only claim commission on tokens that are currently reward tokens
* \#50507 \[SC-High] Non atomic yield distribution may lead to theft of yield
* \#51860 \[SC-High] Missing access control in stakeOnBehalf lets anyone bloat another user’s validator list, leading to permanent fund lock via gas-exhaustion DoS
* \#53047 \[SC-High] The jackpot eligibility check uses stale storage data instead of the freshly calculated streak.
* \#51728 \[SC-High] Users can claim rewards for inactive validator periods due to incorrect checkpoint accrual.
* \#51842 \[SC-High] Unclaimed Staker Rewards Lost When Admin Clears Validator Records Without checking Pending Rewards
* \#52439 \[SC-High] Dust Accumulation in Batched Yield Payouts Leaves Tokens Stranded
* \#53070 \[SC-High] Validator Commission Update During Max Allowed Commission Change Causes Incorrect Reward Calculations
* \#52845 \[SC-High] distributeYieldWithLimit Lacks Snapshot Between Batches, Allowing State Changes to Break Distribution and Lock Yield
* \#50916 \[SC-High] Token Creators Can Bypass Factory `Upgrade Controls` via wrong code implementation of `DEFAULT_ADMIN_ROLE` in ArcTokenFactory.sol.
* \#50713 \[SC-High] Deployer’s DEFAULT\_ADMIN\_ROLE Enables Self-Grant of UPGRADER\_ROLE, Bypassing Implementation Whitelist
* \#52649 \[SC-High] Token Creator Can Seize Upgrade Control, Bypassing Factory Whitelist and Enabling Theft of Funds
* \#52084 \[SC-High] Unstaking Before Reward Token Removal Leads to Incorrect Reward Accrual on Re-addition
* \#52588 \[SC-High] Retroactive reward accrual for newly added tokens when validator was inactive
* \#52198 \[SC-High] Balance Manipulation Between Batches Leading to Inflated Payout and DoS

</details>

<details>

<summary>Medium</summary>

* \#50428 \[SC-Medium] Reverting on callback increases chances of winning
* \#52841 \[SC-Medium] Token admin can DOS admin to not let admin change purchase token
* \#51684 \[SC-Medium] Unbounded Gas Consumption in removeStakerFromAllValidators Leads to Denial-of-Service, Preventing Users with Large Validator Counts from Removing Associations and Potentially Lock...
* \#51887 \[SC-Medium] safeApprove will cause revert of USDT and similar Erc20 token
* \#51666 \[SC-Medium] Inactive Validators Blocked from Claiming Accrued Commission
* \#52620 \[SC-Medium] permanently DoS to ArcTokenPurchase contract
* \#51801 \[SC-Medium] Supra callback allows for theft of gas
* \#51988 \[SC-Medium] `PlumeRewardLogic.calculateRewardsWithCheckpointsView` lacking of checking if the validator is inactive but not slashed.
* \#52719 \[SC-Medium] Inactive validators blocked from claiming commissions despite passed timelock
* \#50340 \[SC-Medium] Any ArcToken admin can block the setting/update of the purchase token indefinitely.
* \#51476 \[SC-Medium] Validators can't claim their accrued commission if they are made inactive
* \#52203 \[SC-Medium] Griefing Attack on ArcTokenPurchase.setPurchaseToken() Function via Front-Running
* \#53048 \[SC-Medium] Approval logic can break on non-standard ERC-20s (USDT-style) and leave allowances loose
* \#51909 \[SC-Medium] Inconsistent Commission Claim Logic Denies Legitimate Claims for Inactive Validators
* \#52732 \[SC-Medium] Permanent DoS of Purchase Token Change
* \#51899 \[SC-Medium] Partial Distribution of yield will fail if the totalEfficentive supply increases.
* \#49732 \[SC-Medium] Malicious Token Admin Can Permanently Block setPurchaseToken
* \#49963 \[SC-Medium] Anyone can create an ArcToken and block the setPurchaseToken() function
* \#52290 \[SC-Medium] `deposit` function in **TellerWithMultiAssetSupportPredicateProxy** is completely broken due to wrong share lock
* \#51043 \[SC-Medium] Core deposit and depositAndBridge Functionality in `TellerWithMultiAssetSupportPredicateProxy` is Non-functional Due to Flawed shareLockPeriod Logic
* \#51613 \[SC-Medium] Yield tokens can be stuck in ArcTokenPurchase , PlumeStakingRewardTreasury or other defi protocols when distributeYield is called.
* \#52075 \[SC-Medium] ArcTokenPurchase Contract is a Token Holder and may be Yield Recipient.
* \#49817 \[SC-Medium] Inactive validators are prevented to claim to eligible commission rewards
* \#52988 \[SC-Medium] deposit function DOS
* \#53035 \[SC-Medium] Share Lock Applied to Wrapper Instead of End User Breaks Transfers or Bypasses Lock
* \#52925 \[SC-Medium] USDT-like approval hygiene can block subsequent operations after partial fill leaves non-zero allowance
* \#52031 \[SC-Medium] Insufficient Access Control in Token Sales Management Leads to Permanent Griefing Attack
* \#52690 \[SC-Medium] DoS Of Smart Contracts On Bridging Functions
* \#51180 \[SC-Medium] Function is vulnerable to gas griefing
* \#51547 \[SC-Medium] Approval Race Condition with safeApprove Leads to Transaction Reverts
* \#50937 \[SC-Medium] Non-zero approve pattern causes permanent freeze of token deposits (e.g. USDT) due to ERC20 incompatibility
* \#52397 \[SC-Medium] Repeated approve without zero-reset can revert on nonstandard ERC20s, blocking deposits
* \#52026 \[SC-Medium] claimAll could revert because of unbounded gas consumptions
* \#52974 \[SC-Medium] When the approval to the `okxApprover` is not fully spent the deposit function will be blocked
* \#49705 \[SC-Medium] Two vectors for unbounded Gas Consumption due to the normal `Raffle` operations
* \#51777 \[SC-Medium] Denial of service on depositAndBridge(...) function for shareLockPeriod is non zero
* \#53021 \[SC-Medium] Deposit-and-Bridge Workflow Bricked by Immediate Share Lock: Users Cannot Bridge Immediately After Deposit
* \#52982 \[SC-Medium] Non-standard ERC20 approvals (USDT-like) cause repeat-call failures after partial fills

</details>

<details>

<summary>Low</summary>

* \#51519 \[SC-Low] `_unstake` does not validate users remaing stake
* \#52843 \[SC-Low] The zero address cannot be whitelisted, which means during restrictions minting and burning cannot work
* \#52129 \[SC-Low] `previewYieldDistribution` Reverts Instead of Returning Zero When No Tokens Are in Circulation
* \#49915 \[SC-Low] Misleading Event Emission in `createWhitelistRestrictions` Function in `RestrictionsFactory` contract
* \#52796 \[SC-Low] Whitelist Restriction in ArcToken Blocks All Minting and Burning
* \#52327 \[SC-Low] Unfair Yield Distribution Due to Last Holder Bias
* \#51596 \[SC-Low] Unsafe uint256 to uint8 Downcast Causes Integer Overflow Leading to Unauthorized Jackpot Payouts After Week 255
* \#52915 \[SC-Low] Yield are transferred before eligibility check potentially leading to freezing of funds
* \#50721 \[SC-Low] Winners cannot claim Prizes until all winners have been drawn in `Raffle::claimPrize`
* \#50399 \[SC-Low] Broken access control in particular contract functions due lack of pause/unpause functionality
* \#51979 \[SC-Low] `getAccruedCommission` returns outdated accrued commission
* \#52710 \[SC-Low] Mint/Burn are blocked when whitelist restrictions are enabled
* \#50082 \[SC-Low] Protocol lets validators operate with dust amounts, making attacks risk-free
* \#50195 \[SC-Low] Unfair Yield Distribution Due to Remainder Allocation to Last Holder
* \#52393 \[SC-Low] Burns blocked by both-sides whitelist with zero-address exclusion when restrictions are enabled
* \#52422 \[SC-Low] Using the current time in getEffectiveRewardRateAt will result in incorrect reward calculation for an entire duration of a time segment
* \#52446 \[SC-Low] Withdrawing Unsold Tokens Desynchronizes Sale Accounting
* \#51816 \[SC-Low] Yield distribution can be front-run to steal rounding remainder as last holder
* \#50839 \[SC-Low] Last Holder Always Gets More Yield
* \#51836 \[SC-Low] Contract Cannot Be Paused Despite Inheriting Pausable
* \#53063 \[SC-Low] `maxValidatorPercentage` can be used to DOS protocol staking
* \#51320 \[SC-Low] Malicious teller parameter allow event data manipulation
* \#53038 \[SC-Low] distributeYield can be frontrun to sandwich rewards. We can force ourselves to be the last holder and get unfairly big bonuses
* \#52277 \[SC-Low] Race Condition in Streak Calculation Leads to Unfair Streak Reset for Users Spinning Near UTC Day Change
* \#51146 \[SC-Low] getMaxNumberOfTokens returns wrong max number of tokens available to buy
* \#51510 \[SC-Low] Bypass of `maxValidatorPercentage` allows a validator to exceed the decentralisation cap
* \#52896 \[SC-Low] “Pause” gate is present but no way to pause
* \#51850 \[SC-Low] upgradeToken(...) can not initialize an upgraded token because the `data` variable of upgradeToAndCall() is hardcoded to empty string
* \#52489 \[SC-Low] When users perform unstake operations in batches, it may cause some funds to be frozen for an additional period of time.
* \#51989 \[SC-Low] Event:: RestrictionsCreated always emits msg.sender as Owner
* \#50624 \[SC-Low] There is a Missing Emergency Pause in Predicate Proxy
* \#50977 \[SC-Low] TellerWithMultiAssetSupportPredicateProxy contract cannot be emergency paused
* \#52314 \[SC-Low] Unsold Token Withdrawal Causes Permanent Inventory Mismatch
* \#50343 \[SC-Low] Cooldown reset vulnerability
* \#52113 \[SC-Low] `StakingFacet.unstake(uint16 validatorId, uint256 amount)` can be abused to bypass `$.minStakeAmount`
* \#50694 \[SC-Low] Spins occuring close to midnight lead to users streaks being unfairly broken due to VRF callback delay
* \#50040 \[SC-Low] Missing Pause Controls, ETH Refund Flaws, and Miscalculated Shares Enable Fund Loss and Protocol Inconsistency in depositAndBridge
* \#51567 \[SC-Low] Contract Cannot Be Paused: Missing Public pause and unpause Functions
* \#51034 \[SC-Low] Sales information is lost when enabling token
* \#52998 \[SC-Low] Minor delays from oracle can unfairly reset users streak
* \#51129 \[SC-Low] BoringVault proxies do not support smart contract wallets
* \#50783 \[SC-Low] Validator percentage cap does not work properly
* \#52339 \[SC-Low] Loss of Daily Streak and Jackpot Eligibility Due to Supra Generator Callback Delay and On-Callback Time Usage in `Spin.sol`
* \#53015 \[SC-Low] Raffle Does Not Invalidate Used Tickets, Breaking Fairness
* \#50922 \[SC-Low] Unstaking partially will extend the cooldown time for previously unstaked amount too
* \#51132 \[SC-Low] `TellerWithMultiAssetSupportPredicateProxy` cannot be paused/unpaused
* \#50889 \[SC-Low] ArcTokenPurchase::withdrawUnsoldArcTokens() Fails to Reduce totalAmountForSale, Leaving Availability Counters Wrong
* \#52519 \[SC-Low] Missing Eligibility Check Before Fund Transfer in distributeYield Leads to Permanent Loss of Yield Tokens
* \#50415 \[SC-Low] getMaxNumberOfTokens() returns wrong value when ArcTokens are withdrawn
* \#50225 \[SC-Low] User can bypass `minStakeAmount` checking.
* \#52979 \[SC-Low] WhitelistRestrictions unintentionally disables mint and burn when transfers are restricted
* \#51122 \[SC-Low] `ArcTokenPurchase#enableToken` can reset the `amountSold` to 0
* \#51776 \[SC-Low] Streak System Breaks Despite Timely User Action Due to Delayed Supra Oracle Callback
* \#52901 \[SC-Low] Wrapped week index can mis-price jackpot table after long uptime
* \#49647 \[SC-Low] Pausable Functions are Not Exposed
* \#52202 \[SC-Low] Failure to Invalidate Winning Tickets Allows Multiple Wins from Single Entry
* \#52990 \[SC-Low] `uint8` truncation and missing cap on week index can return wrong/zero jackpot amounts (Low — Contract fails to deliver promised returns)
* \#49623 \[SC-Low] Unstaking allows going below minimum stake
* \#50487 \[SC-Low] Cross-Campaign Jackpot Denial Due to State Pollution
* \#52891 \[SC-Low] Staking and unstaking immediately an amount little less than the original staked amount leaves dust stake amounts in the system.
* \#51951 \[SC-Low] A Global Blocking Check in claimPrize Prevents Individual Winner Claims Until All Winners Are Drawn
* \#49941 \[SC-Low] Permanent Freezing of Yield Tokens Due to Flawed Check in Distribution Logic
* \#51910 \[SC-Low] Inconsistent yield token transfer logic causes permanent loss of yield in distributeYield()
* \#51391 \[SC-Low] enableToken Function Overwrites amountSold to Zero Causing Permanent Loss of Sales History
* \#50022 \[SC-Low] Missing admin `Pause/Unpause` functions in `TellerWithMultiAssetSupportPredicateProxy` contract
* \#52186 \[SC-Low] Incorrect reward calculation for slashed validators due to single segment time handling
* \#52027 \[SC-Low] WhitelistRestrictions.sol: Mint & Burn Operations Blocked When Transfers Disabled
* \#53059 \[SC-Low] Reward rate checkpoints are used but are never set
* \#51286 \[SC-Low] Event `RestrictionsCreated` uses wrong owner
* \#51771 \[SC-Low] Unsafe downcast of uint256 to uint8 will lead to Silent overflow
* \#52436 \[SC-Low] getAccruedCommission() could return an inaccurate value
* \#51746 \[SC-Low] depositAndBridge(...) function of TellerWithMultiAssetSupportPredicateProxy.sol can not be paused
* \#51980 \[SC-Low] Unstake cooldown period is mistakenly reset on each claim, resulting in temporary frozen funds
* \#52669 \[SC-Low] Token minting is blocked for whitelisted addresses when transfersAllowed is false
* \#52041 \[SC-Low] In ArcToken Attacker Can Reposition to Last Holder and Capture Entire Yield Remainder
* \#52706 \[SC-Low] Multi-Quantity Prize Claims Revert Until All Winners Are Drawn, Freezing Early Winners
* \#51882 \[SC-Low] Unnecessary Claiming Restriction in Raffle Contract Prevents Winners from Claiming Prizes Until All Winners Are Drawn
* \#50551 \[SC-Low] Staked dust positions are not properly prevented
* \#51525 \[SC-Low] Unfair Yield Distribution to Last Holder Due to Flawed Dust Handling
* \#51070 \[SC-Low] Winning Raffle Ticket can be re-used to maintain unfair advantage over other players in `Raffle`
* \#51201 \[SC-Low] Contracts Without Payable Entry Points cannot withdraw nor claim rewards
* \#51929 \[SC-Low] Deactivating `isTransferAllowed` indirectly DOSes minting/burning functionality
* \#49698 \[SC-Low] Coordinated Validator Attack Delays Slashing and Enables Commission Theft
* \#50436 \[SC-Low] `voteToSlashValidator` prevents malicious inactive validators to be slashed.
* \#53056 \[SC-Low] Native-withdraw to msg.sender only → non-payable contract stakers cannot withdraw (permanent funds lock)
* \#51970 \[SC-Low] Spin streak computation relies on oracle callback time; any third-party delay can reset the user’s streak and block jackpot eligibility.
* \#52750 \[SC-Low] Percentage Limit Bypass via Unstaking from Other Validators
* \#52794 \[SC-Low] remainingForSale not updated after withdrawUnsoldArcTokens will cause following buy revert
* \#52870 \[SC-Low] Cooldown Extension Logic May Lead to Locked Funds
* \#50745 \[SC-Low] Single Cooldown Entry Design Causes Timer Reset on Multiple Unstakes Leading to Extended Lock Periods
* \#51296 \[SC-Low] ArcTokenPurchase Withdrawal Breaks View Functions
* \#53069 \[SC-Low] Dynamic Cooldown Interval Changes Cause Unexpected Fund Lockup Extensions
* \#51451 \[SC-Low] Token Freezing via Whitelist Restriction Bypass
* \#51162 \[SC-Low] Missing Pause Control Implementation in TellerWithMultiAssetSupportPredicateProxy
* \#51863 \[SC-Low] Lack of Winning Ticket Removal in handleWinnerSelection Leads to Unfair Prize Distribution and Economic Exploitation
* \#50402 \[SC-Low] Single rate assumption ignores checkpoints in slashed case
* \#51502 \[SC-Low] Enabling Transfer Restrictions Permanently Blocks Minting and Burning
* \#51455 \[SC-Low] Inflated `earned()` / UI rewards when validator stake is zero due to missing `totalStaked` guard in view logic
* \#52976 \[SC-Low] Turning on transfer restriction permanently blocks minting and burning
* \#52948 \[SC-Low] Jackpot Reward Rejected at Exact Threshold
* \#52890 \[SC-Low] No-Recipient Yield Distribution Locks Yield Tokens on ArcToken (effTotal==0)
* \#52241 \[SC-Low] Unexposed Pauseable Functionality
* \#50914 \[SC-Low] Bypass of Minimum Stake Enforcement via Partial Unstake
* \#50120 \[SC-Low] ArcTokens cannot be burned or minted when transfers are restricted
* \#51276 \[SC-Low] ArcTokenPurchase: Re-enabling Active Token Sales Causes Accounting Corruption and Token Loss
* \#51412 \[SC-Low] Token admin can withdraw the token from the purchase contract making the token balance to be less than the totalAmountForSale
* \#51457 \[SC-Low] getAccruedCommission() reverts when token was removed instead of returning the accrued commission
* \#51713 \[SC-Low] Missing Minimum Stake Validation in Unstake Operations
* \#51966 \[SC-Low] `totalAmountClaimable` reverts instead of returning the claimable reward for historical tokens
* \#52312 \[SC-Low] Cooldown coalescing bug: Unintended cooldown extension for prior unstakes
* \#52589 \[SC-Low] In distribute yield function if there are no legitimate users i.e no restricted users the funds will remain stuck
* \#50493 \[SC-Low] Immutable Proxy→Implementation Mapping in RestrictionsFactory Breaks Upgrade Logic
* \#51802 \[SC-Low] Temporary freeze of rewards is possible if efficientSupply == 0

</details>

<details>

<summary>Insight</summary>

* \#50677 \[SC-Insight] Redundant code in `DexAggregatorWrapperWithPredicateProxy` impairs readability and potentially increases gas costs
* \#50393 \[SC-Insight] Unused admin state variable increases deployment and storage costs.
* \#50392 \[SC-Insight] Phantom commission burn
* \#51918 \[SC-Insight] Redundant zero address checks for router address
* \#52468 \[SC-Insight] DoS in Batch Yield Distribution Due to Cross-Batch State Inconsistency
* \#49798 \[SC-Insight] Invalid Holder Set Initialization Bypasses Modular Restrictions, Corrupting Yield Distribution
* \#52837 \[SC-Insight] Gas-heavy repeated binary search increases reward-calculation gas costs
* \#49893 \[SC-Insight] `Raffle.sol` implementation logic allows direct PLUME transfers but has no withdraw locking funds permanently
* \#50470 \[SC-Insight] Inefficient Design in `distributeYieldWithLimit:;ArcToken` Creates Unnecessary Gas Consumption
* \#49835 \[SC-Insight] Dex Aggregator unused ETH loss
* \#49671 \[SC-Insight] Wrong emission in Stake
* \#50187 \[SC-Insight] YieldBlacklistRestrictions Uses Slot 0 Instead of Unstructured Storage, Risking Slot Collision
* \#51001 \[SC-Insight] Inaccurate share calculation in emitted event for non-bridge deposits
* \#50060 \[SC-Insight] Scattered Module Processing Pattern in `ArcToken._update` Function
* \#50312 \[SC-Insight] Validator can steal user rewards due to a lack of cooldown when validator increases commission
* \#51288 \[SC-Insight] Validators commission can be permanently lost
* \#51100 \[SC-Insight] Gas Inefficiency in Prize Removal Logic
* \#50027 \[SC-Insight] Missing Validation of OKX Swap Output Token in function `_okxHelper()`
* \#50691 \[SC-Insight] No validator limit can lead to DoS
* \#49876 \[SC-Insight] Lack of refund on admin-canceled spin requests leads to permanent loss of funds
* \#49800 \[SC-Insight] Yield distribution could encounter an unexpected revert
* \#51707 \[SC-Insight] Gas inefficiency due to redundant \_validateValidatorExists() modifier in requestCommissionClaim()
* \#49868 \[SC-Insight] `Raffle.sol` does not enforce `Prize.endTimeStamp` allowing user and admin interactions with expired Prizes
* \#52303 \[SC-Insight] Incorrect Yield Distribution Event Emission
* \#50506 \[SC-Insight] `StakingFacet` missing event emission on any unstaking operations
* \#49726 \[SC-Insight] There is a redundant zero address check in the ValidatorFacet.sol that is obsolete and could never be true
* \#49668 \[SC-Insight] Validator status function emit misleading event
* \#50973 \[SC-Insight] Incorrect Parameter Type in setJackpotProbabilities
* \#50380 \[SC-Insight] Redundant Use of `allowedImplementations` Mapping in Factory Contracts (createToken and createWhitelistRestrictions in ArcTokenFactory and RestrictionsFactory respectively)
* \#49954 \[SC-Insight] `Raffle::editPrizes` lacks logic to make prizes immutable once winner selection starts or users join breaking user trust.
* \#50887 \[SC-Insight] ArcoTokenPurchase::PurchaseMade Event Mislabels Payment Amount as “pricePaid” Instead Of Unit Price
* \#50168 \[SC-Insight] Unused and duplicated functions should be removed from RewardsFacet and StakingFacet
* \#50461 \[SC-Insight] Incorrect deposit event receiver logged in bridge functions of `DexAggregatorWrapperWithPredicateProxy.sol`
* \#51920 \[SC-Insight] Unnecessary second hand of `if` check in `calculateRewardsWithCheckpointsView`
* \#52248 \[SC-Insight] Lack of initialization check in staking allows users to stake without reward token configured, causing permanent loss of yield
* \#53071 \[SC-Insight] `_okxHelper` function incompatible with the `UNISWAP_V3_SWAP_TO_WITH_PERMIT_SELECTOR`
* \#51171 \[SC-Insight] Redundant Storage Reads and Unnecessary Checks in Reward Rate Checkpoint Logic Lead to Inefficient Gas Usage
* \#50284 \[SC-Insight] Incorrect ERC7201 Storage Implementation in Core Factory Contracts
* \#51651 \[SC-Insight] Redundant Array Access in removeStakerFromValidator
* \#51493 \[SC-Insight] Misleading View Function Documentation
* \#50675 \[SC-Insight] Re-Entrant ETH Refund Can Emit Mismatched shares in Deposit event
* \#52937 \[SC-Insight] Redundant Raffle Ticket Balance Check
* \#52918 \[SC-Insight] Redundant Check for AllWinnersDrawn Error
* \#52137 \[SC-Insight] Silent Override of Non-Global Module Implementation Causes Stored State and Event Log Inconsistency
* \#49919 \[SC-Insight] Unstake function does not unstake all as mentioned in the NatSpec
* \#49932 \[SC-Insight] There are five separate but similar implementations of a binary search that can be condensed into one function
* \#49639 \[SC-Insight] Gas Inefficiency in Loop Storage Reads \_processMaturedCooldowns
* \#49738 \[SC-Insight] Active users in prize pool loose invested raffle tickets when `Raffle::removePrize()` is called.
* \#49768 \[SC-Insight] Missing input validation in `Raffle::editPrize` breaks functionality
* \#51083 \[SC-Insight] claimAll() only loops over active reward tokens and ignores historical tokens
* \#50212 \[SC-Insight] Validators without staked funds can control slashing decisions leading to protocol insolvency
* \#52221 \[SC-Insight] Hardcoded Supra subscription wallet can freeze Spin
* \#50949 \[SC-Insight] No check if raffle actually has enough funds
* \#50632 \[SC-Insight] Critical Timestamp Parsing Bug in getYear() of DateTime contract
* \#49626 \[SC-Insight] Modulo Bias in Winner Selection in Raffle
* \#51712 \[SC-Insight] Yield distribution will revert if global module doesn't implement IYieldRestrictions
* \#52960 \[SC-Insight] Incosistent withdrawable amount calculations
* \#51814 \[SC-Insight] `checkpoint.cumulativeIndex` returned in the `getRewardRateCheckpoint` function will be zero
* \#51159 \[SC-Insight] High Gas: Iterative Date Calculations in DateTime.sol

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#51946 \[SC-High] Commission Claims Fail for Removed Reward Tokens
* \#52964 \[SC-High] if a new reward token is added during a the period a validator is inactive, the validator will still earn rewards/commission for some of the duration in which they were inactive
* \#51519 \[SC-Low] `_unstake` does not validate users remaing stake
* \#53020 \[SC-High] There are functions which when inevitably used could result in wrongly accruing yield for inactive validators, which can make the protocol insolvent
* \#52843 \[SC-Low] The zero address cannot be whitelisted, which means during restrictions minting and burning cannot work
* \#53022 \[SC-Critical] Funds are not properly refunded to user which calls for swap on the dex aggregator
* \#52129 \[SC-Low] `previewYieldDistribution` Reverts Instead of Returning Zero When No Tokens Are in Circulation
* \#50428 \[SC-Medium] Reverting on callback increases chances of winning
* \#49915 \[SC-Low] Misleading Event Emission in `createWhitelistRestrictions` Function in `RestrictionsFactory` contract
* \#52634 \[SC-High] Batch yield distribution has a mathematical flaw that enables economic manipulation
* \#52841 \[SC-Medium] Token admin can DOS admin to not let admin change purchase token
* \#52796 \[SC-Low] Whitelist Restriction in ArcToken Blocks All Minting and Burning
* \#50784 \[SC-High] Any arc token creator can upgrade the implementation
* \#52327 \[SC-Low] Unfair Yield Distribution Due to Last Holder Bias
* \#51912 \[SC-High] Mismatched rounding rules in Reward Logic library results in two-fold loss of earnings
* \#51596 \[SC-Low] Unsafe uint256 to uint8 Downcast Causes Integer Overflow Leading to Unauthorized Jackpot Payouts After Week 255
* \#50822 \[SC-High] Deployer can cpgrade `ArcToken` to malicious implementation and steal all user funds
* \#50477 \[SC-High] Validator loses all accrued commission when reward token is removed
* \#52915 \[SC-Low] Yield are transferred before eligibility check potentially leading to freezing of funds
* \#53034 \[SC-High] ArcTokenFactory doesn't properly handle role management which allows users to arbitrary upgrade their ArcToken's implementation
* \#50735 \[SC-High] some yield tokens will be stuck in contract due to incorrect 'lastProcessedIndex' calculation
* \#50721 \[SC-Low] Winners cannot claim Prizes until all winners have been drawn in `Raffle::claimPrize`
* \#50399 \[SC-Low] Broken access control in particular contract functions due lack of pause/unpause functionality
* \#51684 \[SC-Medium] Unbounded Gas Consumption in removeStakerFromAllValidators Leads to Denial-of-Service, Preventing Users with Large Validator Counts from Removing Associations and Potentially Lock...
* \#51754 \[SC-High] Double yield distribution via token transfers between distributeYieldWithLimit() calls
* \#51979 \[SC-Low] `getAccruedCommission` returns outdated accrued commission
* \#52710 \[SC-Low] Mint/Burn are blocked when whitelist restrictions are enabled
* \#49863 \[SC-Critical] Dex Aggregator ERC20 token theft
* \#50082 \[SC-Low] Protocol lets validators operate with dust amounts, making attacks risk-free
* \#50450 \[SC-High] Logic error in streak validation causes legitimate jackpot wins to be denied, violating reward contract expectations
* \#51887 \[SC-Medium] safeApprove will cause revert of USDT and similar Erc20 token
* \#51558 \[SC-High] ArcToken holder can receive yield twice from `distributeYieldWithLimit`
* \#52955 \[SC-High] A commission rate checkpoint is not created when adding a validator, despite the commission rate being set leading to loss of validator commission.
* \#50860 \[SC-High] Logic Error in Jackpot Eligibility Check Leads to Systematic Theft of User Rewards
* \#50195 \[SC-Low] Unfair Yield Distribution Due to Remainder Allocation to Last Holder
* \#52286 \[SC-High] Off-by-One Error in Jackpot Eligibility Check Leads to Denial of Legitimate Rewards
* \#51456 \[SC-High] Token creator can revoke the UPGRADER role from the factory in order to avoid upgrades
* \#52371 \[SC-High] `distributeYieldWithLimit` is vulnerable to inter-batch `balance and holders array` mutations
* \#52393 \[SC-Low] Burns blocked by both-sides whitelist with zero-address exclusion when restrictions are enabled
* \#51589 \[SC-High] TokenCreator retains upgrade rights – Fix remains insufficient - Finding #01: Immunefi Report
* \#52422 \[SC-Low] Using the current time in getEffectiveRewardRateAt will result in incorrect reward calculation for an entire duration of a time segment
* \#53061 \[SC-High] Asymmetric rounding in commission (ceil for users, floor for validators) enables per-segment rounding loss; validators can amplify via frequent commission checkpoints.
* \#50677 \[SC-Insight] Redundant code in `DexAggregatorWrapperWithPredicateProxy` impairs readability and potentially increases gas costs
* \#49616 \[SC-High] User can steal Rewards
* \#51666 \[SC-Medium] Inactive Validators Blocked from Claiming Accrued Commission
* \#52446 \[SC-Low] Withdrawing Unsold Tokens Desynchronizes Sale Accounting
* \#52165 \[SC-High] user can't claim reward ERC20 tokens since rewards transfer will revert
* \#52218 \[SC-High] Creator Retains DEFAULT\_ADMIN\_ROLE, Allowing Bypass of Upgrade Restrictions
* \#51816 \[SC-Low] Yield distribution can be front-run to steal rounding remainder as last holder
* \#52499 \[SC-High] ArcToken Factory's admin cannot upgrade an ArcToken
* \#49731 \[SC-High] Theft on Re-Added Tokens
* \#52620 \[SC-Medium] permanently DoS to ArcTokenPurchase contract
* \#50839 \[SC-Low] Last Holder Always Gets More Yield
* \#51801 \[SC-Medium] Supra callback allows for theft of gas
* \#51836 \[SC-Low] Contract Cannot Be Paused Despite Inheriting Pausable
* \#52833 \[SC-High] bypass the fix of immunefi audit IMM-CRIT-01 ：Token Creator Can Upgrade ArcToken Implementation
* \#51479 \[SC-High] Inaccurate Reward Calculation Post-Validator Slashing Due to Premature Timestamp Update on Token Removal
* \#53063 \[SC-Low] `maxValidatorPercentage` can be used to DOS protocol staking
* \#51352 \[SC-Critical] User will lose the unspent amount when executing partial swaps via 1inch
* \#51320 \[SC-Low] Malicious teller parameter allow event data manipulation
* \#52983 \[SC-High] Validator will loose commission for the tokens which are removed from the reward tokens but they still have commission left to be claimed.
* \#53038 \[SC-Low] distributeYield can be frontrun to sandwich rewards. We can force ourselves to be the last holder and get unfairly big bonuses
* \#51988 \[SC-Medium] `PlumeRewardLogic.calculateRewardsWithCheckpointsView` lacking of checking if the validator is inactive but not slashed.
* \#52527 \[SC-High] The validator admin might claim less commission token when `ValidatorFacet.requestCommissionClaim` is called.
* \#50350 \[SC-High] StakingFacet: stakeOnBehalf allows to prevent withdraws
* \#53037 \[SC-Critical] Commission changes can retroactively affect user rewards
* \#52277 \[SC-Low] Race Condition in Streak Calculation Leads to Unfair Streak Reset for Users Spinning Near UTC Day Change
* \#52719 \[SC-Medium] Inactive validators blocked from claiming commissions despite passed timelock
* \#50340 \[SC-Medium] Any ArcToken admin can block the setting/update of the purchase token indefinitely.
* \#53001 \[SC-High] Yield tokens become stuck in ArcTokenPurchase contract when distributing yield during active sales
* \#51146 \[SC-Low] getMaxNumberOfTokens returns wrong max number of tokens available to buy
* \#52803 \[SC-High] \_canRecoverFromCooldown is inconsistent when slash and cooldown maturity occur in the same block
* \#51510 \[SC-Low] Bypass of `maxValidatorPercentage` allows a validator to exceed the decentralisation cap
* \#52464 \[SC-High] Commission rounding mismatch under payment bug
* \#52896 \[SC-Low] “Pause” gate is present but no way to pause
* \#53011 \[SC-Critical] Uncleaned Partial Approval Consumption in DEX Aggregator Integration Leads to Permanent DoS
* \#51476 \[SC-Medium] Validators can't claim their accrued commission if they are made inactive
* \#52390 \[SC-High] `_validateIsToken(...)` blocks validators from claiming earned rewards from removed tokens.
* \#52203 \[SC-Medium] Griefing Attack on ArcTokenPurchase.setPurchaseToken() Function via Front-Running
* \#51850 \[SC-Low] upgradeToken(...) can not initialize an upgraded token because the `data` variable of upgradeToAndCall() is hardcoded to empty string
* \#49939 \[SC-High] Initial timestamp mismatch might lead to users being able to spin twice in the same day
* \#52489 \[SC-Low] When users perform unstake operations in batches, it may cause some funds to be frozen for an additional period of time.
* \#51989 \[SC-Low] Event:: RestrictionsCreated always emits msg.sender as Owner
* \#50393 \[SC-Insight] Unused admin state variable increases deployment and storage costs.
* \#53048 \[SC-Medium] Approval logic can break on non-standard ERC-20s (USDT-style) and leave allowances loose
* \#50490 \[SC-High] User loses reward tokens during validator-user relationship clearing
* \#50624 \[SC-Low] There is a Missing Emergency Pause in Predicate Proxy
* \#53051 \[SC-High] Unconsented stakeOnBehalf enables third-party gas-griefing DoS by bloating userValidators, breaking withdraw/claimAll
* \#51909 \[SC-Medium] Inconsistent Commission Claim Logic Denies Legitimate Claims for Inactive Validators
* \#53025 \[SC-High] Commission on removed tokens is unclaimable
* \#50977 \[SC-Low] TellerWithMultiAssetSupportPredicateProxy contract cannot be emergency paused
* \#53028 \[SC-High] There is an Asymmetric Rounding issue that is can cause a Theft of Unclaimed Yield in Reward or Commission Accounting
* \#52314 \[SC-Low] Unsold Token Withdrawal Causes Permanent Inventory Mismatch
* \#50343 \[SC-Low] Cooldown reset vulnerability
* \#50527 \[SC-High] Attacker can steal yield during batch distribution
* \#51051 \[SC-High] Inactive Validator Reward Accrual Bypass
* \#50392 \[SC-Insight] Phantom commission burn
* \#52113 \[SC-Low] `StakingFacet.unstake(uint16 validatorId, uint256 amount)` can be abused to bypass `$.minStakeAmount`
* \#50694 \[SC-Low] Spins occuring close to midnight lead to users streaks being unfairly broken due to VRF callback delay
* \#51992 \[SC-High] Dust Accumulation in ArcToken during Yield Distribution.
* \#52732 \[SC-Medium] Permanent DoS of Purchase Token Change
* \#50040 \[SC-Low] Missing Pause Controls, ETH Refund Flaws, and Miscalculated Shares Enable Fund Loss and Protocol Inconsistency in depositAndBridge
* \#52865 \[SC-High] Inconsistency in how stake cooldown is handled due to off by one error
* \#52931 \[SC-High] Validators can not claim their commissions after the reward token removal.
* \#51847 \[SC-Critical] DoS via dust leftover in erc-20 approvals
* \#52798 \[SC-High] Integer Division Remainder Loss in Batched Yield Distribution Causes Permanent Fund Lock
* \#51567 \[SC-Low] Contract Cannot Be Paused: Missing Public pause and unpause Functions
* \#52995 \[SC-High] Validators lose access to historical reward tokens when tokens are removed
* \#52409 \[SC-High] Asymmetric commission rounding creates systematic accounting drift
* \#51034 \[SC-Low] Sales information is lost when enabling token
* \#51878 \[SC-High] Timing Misalignment Between Campaign Days and Calendar Days Allows Double Spinning on High-Probability Jackpot Days
* \#52998 \[SC-Low] Minor delays from oracle can unfairly reset users streak
* \#49700 \[SC-High] Validator Commission can be Blocked
* \#52424 \[SC-High] There is a Retroactive Commission Miscalculation in PlumeRewardLogic
* \#51899 \[SC-Medium] Partial Distribution of yield will fail if the totalEfficentive supply increases.
* \#51129 \[SC-Low] BoringVault proxies do not support smart contract wallets
* \#52961 \[SC-High] Theft of yield from the distributor.
* \#52254 \[SC-High] ArcToken theft beyond unclaimed yield during distribution
* \#53018 \[SC-High] Owed rewards could be lost for some users for periods before slashing time due to incorrect logic.
* \#52517 \[SC-High] Missing Point-in-Time Snapshot in Batched Yield Distribution Enables Double-Claims and Permanent Fund Lock
* \#53016 \[SC-High] ArcTokenPurchase doesn't allow RWA-token owners to recover accrued yield from stored ArcTokens waiting for sale.
* \#51041 \[SC-High] Streak‑Count Misuse in Jackpot Eligibility Allows Theft of User Funds
* \#50783 \[SC-Low] Validator percentage cap does not work properly
* \#49732 \[SC-Medium] Malicious Token Admin Can Permanently Block setPurchaseToken
* \#52339 \[SC-Low] Loss of Daily Streak and Jackpot Eligibility Due to Supra Generator Callback Delay and On-Callback Time Usage in `Spin.sol`
* \#53015 \[SC-Low] Raffle Does Not Invalidate Used Tickets, Breaking Fairness
* \#50922 \[SC-Low] Unstaking partially will extend the cooldown time for previously unstaked amount too
* \#49963 \[SC-Medium] Anyone can create an ArcToken and block the setPurchaseToken() function
* \#51999 \[SC-High] Logical Flaw in Validator Reactivation and addRewardToken Allows Claiming Rewards for Validators in Inactive Periods
* \#52290 \[SC-Medium] `deposit` function in **TellerWithMultiAssetSupportPredicateProxy** is completely broken due to wrong share lock
* \#49854 \[SC-Critical] Dex Aggregator partial fill token loss
* \#52996 \[SC-High] Users can claim rewards for newly added reward tokens even when the validator they staked for was inactive during some time interval.
* \#50252 \[SC-High] Rounding excess yield tokens become permanently stuck when last holder is yield-restricted
* \#52285 \[SC-High] Incorrect Dust Handling in Yield Distribution Leads to Permanent Fund Lock
* \#51132 \[SC-Low] `TellerWithMultiAssetSupportPredicateProxy` cannot be paused/unpaused
* \#50951 \[SC-High] Inconsistent streak count usage between jackpot and raffle ticket calculations
* \#50889 \[SC-Low] ArcTokenPurchase::withdrawUnsoldArcTokens() Fails to Reduce totalAmountForSale, Leaving Availability Counters Wrong
* \#52519 \[SC-Low] Missing Eligibility Check Before Fund Transfer in distributeYield Leads to Permanent Loss of Yield Tokens
* \#52576 \[SC-High] Flaw in Raffle::determineReward in Jackpot Prize Calculation after week 12
* \#52667 \[SC-High] commission is not added at point of adding validator hence stakers that stake before the first checkpoint would always use the current commission
* \#50415 \[SC-Low] getMaxNumberOfTokens() returns wrong value when ArcTokens are withdrawn
* \#51961 \[SC-High] Attackers can deny commission rewards to validators by repeatedly calling forceSettleValidatorCommission()
* \#52770 \[SC-High] Unbounded Gas Consumption via stakeOnBehalf Manipulation
* \#50225 \[SC-Low] User can bypass `minStakeAmount` checking.
* \#50412 \[SC-High] Illegitimate Reward Claim After Unstake Due to Overlapping Reward Rate Checkpoints
* \#52979 \[SC-Low] WhitelistRestrictions unintentionally disables mint and burn when transfers are restricted
* \#51414 \[SC-High] Attacker can drain yield by transferring tokens to other address in yield batch distributions
* \#51283 \[SC-Critical] Permanent Freeze of User token Due to Unhandled Partial Fill Refunds for swap via 1inch in `DexAggregatorWrapperWithPredicateProxy`
* \#51122 \[SC-Low] `ArcTokenPurchase#enableToken` can reset the `amountSold` to 0
* \#51776 \[SC-Low] Streak System Breaks Despite Timely User Action Due to Delayed Supra Oracle Callback
* \#51043 \[SC-Medium] Core deposit and depositAndBridge Functionality in `TellerWithMultiAssetSupportPredicateProxy` is Non-functional Due to Flawed shareLockPeriod Logic
* \#51613 \[SC-Medium] Yield tokens can be stuck in ArcTokenPurchase , PlumeStakingRewardTreasury or other defi protocols when distributeYield is called.
* \#51918 \[SC-Insight] Redundant zero address checks for router address
* \#52901 \[SC-Low] Wrapped week index can mis-price jackpot table after long uptime
* \#49647 \[SC-Low] Pausable Functions are Not Exposed
* \#52847 \[SC-High] No function to recover the remained yield by distributeYieldWithLimit
* \#52202 \[SC-Low] Failure to Invalidate Winning Tickets Allows Multiple Wins from Single Entry
* \#52990 \[SC-Low] `uint8` truncation and missing cap on week index can return wrong/zero jackpot amounts (Low — Contract fails to deliver promised returns)
* \#52278 \[SC-High] Incorrect Streak Check in Jackpot Eligibility Leads to Unfair Reward Denial
* \#52075 \[SC-Medium] ArcTokenPurchase Contract is a Token Holder and may be Yield Recipient.
* \#50167 \[SC-High] Retroactive reward drain via incomplete reward debt reset
* \#50924 \[SC-High] Validators are not able to claim their accrued commission when the reward token is removed.
* \#52104 \[SC-High] Removed reward tokens block validator commission claims
* \#50409 \[SC-High] Validator will lose comission
* \#49817 \[SC-Medium] Inactive validators are prevented to claim to eligible commission rewards
* \#51987 \[SC-High] Validators will be able to steal more commission from users that isn't the commission to be charged
* \#52572 \[SC-High] A legitimate arc token holder can be denied his yield.
* \#49623 \[SC-Low] Unstaking allows going below minimum stake
* \#52468 \[SC-Insight] DoS in Batch Yield Distribution Due to Cross-Batch State Inconsistency
* \#52680 \[SC-High] holders length changing when distributing limit with limit could lead to case where new holders unfairly claim yield and yield is permanently frozen
* \#50487 \[SC-Low] Cross-Campaign Jackpot Denial Due To State Pollution
* \#52891 \[SC-Low] Staking and unstaking immediately an amount little less than the original staked amount leaves dust stake amounts in the system.
* \#49798 \[SC-Insight] Invalid Holder Set Initialization Bypasses Modular Restrictions, Corrupting Yield Distribution
* \#51951 \[SC-Low] A Global Blocking Check in claimPrize Prevents Individual Winner Claims Until All Winners Are Drawn
* \#49941 \[SC-Low] Permanent Freezing of Yield Tokens Due To Flawed Check in Distribution Logic
* \#53077 \[SC-High] Permanent Fund Lock Due To Flawed Remainder Logic in distributeYield
* \#52449 \[SC-High] Broken Streaks Still Pass Jackpot Eligibility in Spin Contract
* \#52837 \[SC-Insight] Gas-heavy repeated binary search increases reward-calculation gas costs
* \#52986 \[SC-High] Jackpot check uses previous `streakCount` instead of current computed streak, denying jackpot on first eligible day
* \#52601 \[SC-High] In Spin::handleRandomness() jackpot eligibility uses outdated streakCount instead of updated streak
* \#51910 \[SC-Low] Inconsistent yield token transfer logic causes permanent loss of yield in distributeYield()
* \#49893 \[SC-Insight] `Raffle.sol` implementation logic allows direct PLUME transfers but has no withdraw locking funds permanently
* \#52500 \[SC-High] Missing Commission Checkpoint Initialization Leads to Retroactive Commission Theft of User Rewards
* \#51391 \[SC-Low] enableToken Function Overwrites amountSold to Zero Causing Permanent Loss of Sales History
* \#51324 \[SC-High] Rounding in commission accounting burns delegator rewards
* \#50022 \[SC-Low] Missing admin `Pause/Unpause` functions in `TellerWithMultiAssetSupportPredicateProxy` contract
* \#50470 \[SC-Insight] Inefficient Design in `distributeYieldWithLimit:;ArcToken` Creates Unnecessary Gas Consumption
* \#52186 \[SC-Low] Incorrect reward calculation for slashed validators due to single segment time handling
* \#52787 \[SC-High] Batched yield distribution rounding in ArcToken permanently freezes unclaimed funds and misreports payouts
* \#49835 \[SC-Insight] Dex Aggregator unused ETH loss
* \#49671 \[SC-Insight] Wrong emission in Stake
* \#50187 \[SC-Insight] YieldBlacklistRestrictions Uses Slot 0 Instead Of Unstructured Storage, Risking Slot Collision
* \#51653 \[SC-High] Permanent loss of staker rewards after slashing when validator records are cleared
* \#51001 \[SC-Insight] Inaccurate share calculation in emitted event for non-bridge deposits
* \#50060 \[SC-Insight] Scattered Module Processing Pattern in `ArcToken._update` Function
* \#52027 \[SC-Low] WhitelistRestrictions.sol: Mint & Burn Operations Blocked When Transfers Disabled
* \#53039 \[SC-High] Rewards and commissions accrued in the interval before a slash might be lost
* \#53059 \[SC-Low] Reward rate checkpoints are used but are never set
* \#52988 \[SC-Medium] deposit function DOS
* \#53035 \[SC-Medium] Share Lock Applied to Wrapper Instead Of End User Breaks Transfers or Bypasses Lock
* \#51197 \[SC-High] Arc Token owner can take upgrader role for themselves lockout the factory and upgrade the contract without the knowledge of the factory
* \#51286 \[SC-Low] Event `RestrictionsCreated` uses wrong owner
* \#53043 \[SC-High] handleRandomness doesn't properly account for current streak which could result in the User spinning losing a Jackpot
* \#51771 \[SC-Low] Unsafe downcast of uint256 to uint8 will lead to Silent overflow
* \#52436 \[SC-Low] getAccruedCommission() could return an inaccurate value
* \#52780 \[SC-High] Timestamp Manipulation in forceSettleValidatorCommission Leads to Permanent Loss of Staker Rewards
* \#50312 \[SC-Insight] Validator can steal user rewards due to a lack of cooldown when validator increases commission
* \#51288 \[SC-Insight] Validators commission can be permanently lost
* \#52889 \[SC-High] Inactive validators accrue rewards for new tokens
* \#51746 \[SC-Low] depositAndBridge(...) function of TellerWithMultiAssetSupportPredicateProxy.sol can not be paused
* \#51980 \[SC-Low] Unstake cooldown period is mistakenly reset on each claim, resulting in temporary frozen funds
* \#52560 \[SC-High] Incorrect current streak used when calculating whether the jackpot should be awarded or not
* \#52669 \[SC-Low] Token minting is blocked for whitelisted addresses when transfersAllowed is false
* \#50796 \[SC-High] Jackpot eligibility uses stale streak
* \#52041 \[SC-Low] In ArcToken Attacker Can Reposition to Last Holder and Capture Entire Yield Remainder
* \#51090 \[SC-High] malicious user can steal yields when `ArcToken.distributeYieldWithLimit` is used.
* \#52573 \[SC-High] Unconsented `stakeOnBehalf` enables **unbounded gas consumption** via `userValidators[]` growth, causing DoS at scale in `claimAll()` / `withdraw()`
* \#52706 \[SC-Low] Multi-Quantity Prize Claims Revert Until All Winners Are Drawn, Freezing Early Winners
* \#51100 \[SC-Insight] Gas Inefficiency in Prize Removal Logic
* \#50519 \[SC-High] RewardsFacet: Reintroducing an old reward token will result in wrong accounting, leading to theft of yield
* \#51882 \[SC-Low] Unnecessary Claiming Restriction in Raffle Contract Prevents Winners from Claiming Prizes Until All Winners Are Drawn
* \#50560 \[SC-High] Inconsistent Commission Rounding Traps User/Validator Funds
* \#50027 \[SC-Insight] Missing Validation of OKX Swap Output Token in function `_okxHelper()`
* \#50691 \[SC-Insight] No validator limit can lead to DoS
* \#50551 \[SC-Low] Staked dust positions are not properly prevented
* \#51525 \[SC-Low] Unfair Yield Distribution to Last Holder Due to Flawed Dust Handling
* \#51124 \[SC-High] Validator would loss commission fee if the rewards token are removed
* \#51070 \[SC-Low] Winning Raffle Ticket can be re-used to maintain unfair advantage over other players in `Raffle`
* \#51201 \[SC-Low] Contracts Without Payable Entry Points cannot withdraw nor claim rewards
* \#51929 \[SC-Low] Deactivating `isTransferAllowed` indirectly DOSes minting/burning functionality
* \#49698 \[SC-Low] Coordinated Validator Attack Delays Slashing and Enables Commission Theft
* \#50436 \[SC-Low] `voteToSlashValidator` prevents malicious inactive validators to be slashed.
* \#49876 \[SC-Insight] Lack of refund on admin-canceled spin requests leads to permanent loss of funds
* \#49800 \[SC-Insight] Yield distribution could encounter an unexpected revert
* \#52849 \[SC-High] Claimers who claim after (slash/inactive + updateRewardPerTokenForValidator which advances `validatorLastUpdateTimes to be more than slashTimestamp`) will lose rewards for a segment
* \#51707 \[SC-Insight] Gas inefficiency due to redundant \_validateValidatorExists() modifier in requestCommissionClaim()
* \#53056 \[SC-Low] Native-withdraw to msg.sender only → non-payable contract stakers cannot withdraw (permanent funds lock)
* \#51970 \[SC-Low] Spin streak computation relies on oracle callback time; any third-party delay can reset the user’s streak and block jackpot eligibility.
* \#52956 \[SC-High] State Inconsistency in Batched Yield Distribution Leads to Direct Theft of User Funds and Protocol Insolvency
* \#52923 \[SC-Critical] Partial fill traps source token residual inside the wrapper and leaves unsafe residual allowance
* \#52925 \[SC-Medium] USDT-like approval hygiene can block subsequent operations after partial fill leaves non-zero allowance
* \#49868 \[SC-Insight] `Raffle.sol` does not enforce `Prize.endTimeStamp` allowing user and admin interactions with expired Prizes
* \#51941 \[SC-High] Token creator can revoke factory's upgrade capability, permanently blocking upgrades
* \#52750 \[SC-Low] Percentage Limit Bypass via Unstaking from Other Validators
* \#52794 \[SC-Low] remainingForSale not updated after withdrawUnsoldArcTokens will cause following buy revert
* \#52303 \[SC-Insight] Incorrect Yield Distribution Event Emission
* \#50506 \[SC-Insight] `StakingFacet` missing event emission on any unstaking operations
* \#49710 \[SC-High] Cross-batch state manipulation in yield distribution allows double-dipping of yield funds
* \#49726 \[SC-Insight] There is a redundant zero address check in the ValidatorFacet.sol that is obsolete and could never be true
* \#49668 \[SC-Insight] Validator status function emit misleading event
* \#52031 \[SC-Medium] Insufficient Access Control in Token Sales Management Leads to Permanent Griefing Attack
* \#50973 \[SC-Insight] Incorrect Parameter Type in setJackpotProbabilities
* \#49787 \[SC-High] Batched Yield Distribution Doesn't Account For Transfers/Purchases Between Batches
* \#50380 \[SC-Insight] Redundant Use of `allowedImplementations` Mapping in Factory Contracts (createToken and createWhitelistRestrictions in ArcTokenFactory and RestrictionsFactory respectively)
* \#49954 \[SC-Insight] `Raffle::editPrizes` lacks logic to make prizes immutable once winner selection starts or users join breaking user trust.
* \#52870 \[SC-Low] Cooldown Extension Logic May Lead to Locked Funds
* \#50745 \[SC-Low] Single Cooldown Entry Design Causes Timer Reset on Multiple Unstakes Leading to Extended Lock Periods
* \#51296 \[SC-Low] ArcTokenPurchase Withdrawal Breaks View Functions
* \#50887 \[SC-Insight] ArcoTokenPurchase::PurchaseMade Event Mislabels Payment Amount as “pricePaid” Instead Of Unit Price
* \#51218 \[SC-High] Oracle callback timing vulnerability causes jackpot prize loss
* \#50275 \[SC-High] Eligible user loses Jackpot
* \#52178 \[SC-Critical] User will lose the unspent amount when executing partial swaps via OkxRouter
* \#53069 \[SC-Low] Dynamic Cooldown Interval Changes Cause Unexpected Fund Lockup Extensions
* \#50168 \[SC-Insight] Unused and duplicated functions should be removed from RewardsFacet and StakingFacet
* \#51451 \[SC-Low] Token Freezing via Whitelist Restriction Bypass
* \#51530 \[SC-High] Validators can not Claim Pending Accrued Commission when Reward tokens have been removed from the isRewardToken mapping
* \#51162 \[SC-Low] Missing Pause Control Implementation in TellerWithMultiAssetSupportPredicateProxy
* \#50461 \[SC-Insight] Incorrect deposit event receiver logged in bridge functions of `DexAggregatorWrapperWithPredicateProxy.sol`
* \#51920 \[SC-Insight] Unnecessary second hand of `if` check in `calculateRewardsWithCheckpointsView`
* \#52248 \[SC-Insight] Lack of initialization check in staking allows users to stake without reward token configured, causing permanent loss of yield
* \#51658 \[SC-High] Yield distribution in batches let the same tokens collect rewards in multiple batches, stealing yield from other users
* \#52347 \[SC-High] Improper handling of yield distribution state in distributeYieldWithLimit() leads to revert, freezing users' yield
* \#52711 \[SC-High] In ValidatorFacet, validator cannot claims commissions of removed tokens
* \#50425 \[SC-High] Active non-slashed validators cannot claim rewards when a reward token is disabled
* \#53071 \[SC-Insight] `_okxHelper` function incompatible with the `UNISWAP_V3_SWAP_TO_WITH_PERMIT_SELECTOR`
* \#53072 \[SC-High] Ceil-vs-Floor Rounding Mismatch Causes Systematic Underpayment and Unclaimed Yield Leakage
* \#51171 \[SC-Insight] Redundant Storage Reads and Unnecessary Checks in Reward Rate Checkpoint Logic Lead to Inefficient Gas Usage
* \#50284 \[SC-Insight] Incorrect ERC7201 Storage Implementation in Core Factory Contracts
* \#51651 \[SC-Insight] Redundant Array Access in removeStakerFromValidator
* \#52690 \[SC-Medium] DoS Of Smart Contracts On Bridging Functions
* \#51863 \[SC-Low] Lack of Winning Ticket Removal in handleWinnerSelection Leads to Unfair Prize Distribution and Economic Exploitation
* \#51180 \[SC-Medium] Function is vulnerable to gas griefing
* \#51493 \[SC-Insight] Misleading View Function Documentation
* \#50675 \[SC-Insight] Re-Entrant ETH Refund Can Emit Mismatched shares in Deposit event
* \#51369 \[SC-High] Unbounded iteration gas‑dos in `_validateTokenForClaim`
* \#51994 \[SC-High] Permanent Loss of Validator Commission Upon Reward Token Removal
* \#50402 \[SC-Low] Single rate assumption ignores checkpoints in slashed case
* \#50433 \[SC-High] Validator List Griefing: Unrestricted stakeOnBehalf allows User Asset freeze permanently
* \#51813 \[SC-High] Malicious User Can Grief Victims by Staking Them Across Many Validators Leading to Fund Freezing
* \#51547 \[SC-Medium] Approval Race Condition with safeApprove Leads to Transaction Reverts
* \#50937 \[SC-Medium] Non-zero approve pattern causes permanent freeze of token deposits (e.g. USDT) due to ERC20 incompatibility
* \#51502 \[SC-Low] Enabling Transfer Restrictions Permanently Blocks Minting and Burning
* \#51452 \[SC-High] stakeOnBehalf() function enables out-of-gas DoS
* \#51455 \[SC-Low] Inflated `earned()` / UI rewards when validator stake is zero due to missing `totalStaked` guard in view logic
* \#52976 \[SC-Low] Turning on transfer restriction permanently blocks minting and burning
* \#52397 \[SC-Medium] Repeated approve without zero-reset can revert on nonstandard ERC20s, blocking deposits
* \#52948 \[SC-Low] Jackpot Reward Rejected at Exact Threshold
* \#52890 \[SC-Low] No-Recipient Yield Distribution Locks Yield Tokens on ArcToken (effTotal==0)
* \#52026 \[SC-Medium] claimAll could revert because of unbounded gas consumptions
* \#52937 \[SC-Insight] Redundant Raffle Ticket Balance Check
* \#52918 \[SC-Insight] Redundant Check For AllWinnersDrawn Error
* \#52137 \[SC-Insight] Silent Override of Non-Global Module Implementation Causes Stored State and Event Log Inconsistency
* \#52944 \[SC-High] The `requestCommisionClaim` function can only claim commission on tokens that are currently reward tokens
* \#52974 \[SC-Medium] When the approval to the `okxApprover` is not fully spent the deposit function will be blocked
* \#49919 \[SC-Insight] Unstake function does not unstake all as mentioned in the NatSpec
* \#49932 \[SC-Insight] There are five separate but similar implementations of a binary search that can be condensed into one function
* \#49705 \[SC-Medium] Two vectors for unbounded Gas Consumption due to the normal `Raffle` operations
* \#50507 \[SC-High] Non atomic yield distribution may lead to theft of yield
* \#49639 \[SC-Insight] Gas Inefficiency in Loop Storage Reads \_processMaturedCooldowns
* \#49738 \[SC-Insight] Active users in prize pool loose invested raffle tickets when `Raffle::removePrize()` is called.
* \#49768 \[SC-Insight] Missing input validation in `Raffle::editPrize` breaks functionality
* \#51083 \[SC-Insight] claimAll() only loops over active reward tokens and ignores historical tokens
* \#50212 \[SC-Insight] Validators without staked funds can control slashing decisions leading to protocol insolvency
* \#52221 \[SC-Insight] Hardcoded Supra subscription wallet can freeze Spin
* \#50949 \[SC-Insight] No check if raffle actually has enough funds
* \#50632 \[SC-Insight] Critical Timestamp Parsing Bug in getYear() of DateTime contract
* \#49626 \[SC-Insight] Modulo Bias in Winner Selection in Raffle
* \#51712 \[SC-Insight] Yield distribution will revert if global module doesn't implement IYieldRestrictions
* \#52960 \[SC-Insight] Incosistent withdrawable amount calculations
* \#51814 \[SC-Insight] `checkpoint.cumulativeIndex` returned in the `getRewardRateCheckpoint` function will be zero
* \#51159 \[SC-Insight] High Gas: Iterative Date Calculations in DateTime.sol

</details>

(End of report list)
