# Vechain | Stargate Hayabusa

## Reports by Severity

<details>

<summary>High</summary>

* \#60334 \[SC-High] Unstake permanently reverts when validator exits after delegator exit (double-decrease of effective stake)
* \#59316 \[SC-High] Off-by-One Unlocks Infinite VTHO Reward Drain from Ghost Stakes
* \#60311 \[SC-High] Double effective-stake decrement freezes unstake permanently after validator exit
* \#60081 \[SC-High] Exited delegator can continue to accrue and claim delegation rewards
* \#60298 \[SC-High] Duplicate effectiveStake decrement path bricks unstake/re-delegate
* \#60372 \[SC-High] Double-Decrement Bug — Effective Stake Underflow Permanently Locks Funds
* \#60373 \[SC-High] Incorrect effective stake decrement when validator exits causes permanent freezing of user stake
* \#59421 \[SC-High] Theft of Unclaimed Yield via Incorrect Period Range Calculation and Lack of Per-User Effective Stake Tracking
* \#60150 \[SC-High] Off-by-one in claim window lets exited delegations harvest post-exit rewards
* \#60151 \[SC-High] Double Reduction of Effective Stake can lead to Stuck Delegations.
* \#59443 \[SC-High] rithmetic Underflow in Effective Stake Accounting Causes Permanent Loss of Funds
* \#60533 \[SC-High] overlap which will lead to loss of fund
* \#60154 \[SC-High] Exited delegations can continue claiming VTHO rewards for future periods
* \#59723 \[SC-High] Double-decrease after exit + validator EXITED leads to underflow and permanent freeze
* \#59730 \[SC-High] Permanent DoS - Users Cannot Unstake After Double Exit Scenario
* \#59733 \[SC-High] Post-exit delegations can drain future rewards
* \#60592 \[SC-High] users are unable to unstake under certain conditions
* \#60069 \[SC-High] Incorrect claimable period calculation Leading to attacker keep claiming even after exiting the delegation.
* \#60049 \[SC-High] Double Effective Stake Decrement Locks Delegators : Unstake Reverts Due to Duplicate EffectiveStake Decrements in Exit Flow
* \#60426 \[SC-High] Rewards Accounting Off-By-One / Skipped/Double Period Exploit leads to "Direct loss of user funds" via incorrect reward distribution; "Theft of unclaimed yield", misallocation of VT...
* \#59752 \[SC-High] Off-by-one bug in \_claimableDelegationPeriods allows claiming yield for periods after exit
* \#59563 \[SC-High] Exited Delegators Can Claim Rewards Indefinitely After Exit
* \#60429 \[SC-High] Double-Decrease of Effective Stake Prevents Delegators from Unstaking
* \#60431 \[SC-High] Unauthorized VTHO reward claims after delegation exit
* \#59776 \[SC-High] Exited delegators can over‑claim VTHO rewards for post‑exit periods due to off‑by‑one error in \_claimableDelegationPeriods
* \#60557 \[SC-High] Double Decrement of Effective Stake in unstake() leads to DoS and Permanent Fund Lock
* \#59802 \[SC-High] Double subtraction of validator effective stake will permanently lock other delegators’ staked VET
* \#60019 \[SC-High] Off-by-one in Stargate.sol \_claimableDelegationPeriods lets exited NFTs siphon validator rewards, leading to protocol insolvency
* \#59919 \[SC-High] Loss of funds - Delegators can claim rewards for periods where they had no stake
* \#60173 \[SC-High] the Phantom Claimable Periods Can Permanently Lock The Staked VET for Ended Delegations
* \#59615 \[SC-High] Off-by-one error in period boundary check allows theft of unclaimed yield after delegation exit
* \#60506 \[SC-High] Double delegatorsEffectiveStake Decrease Permanently Prevents Single NFT from Unstaking
* \#59657 \[SC-High] Delegators Lose First Reward Period When Delegating to Pending Validators
* \#60470 \[SC-High] Double-Decrease of Validator Stake in Stargate.sol
* \#59358 \[SC-High] Off by One Error in Reward Claim Logic Allows Delegators to Steal VTHO for Periods After Delegation Ended
* \#59386 \[SC-High] Fund freeze from double stake subtraction (when validator exits)
* \#60575 \[SC-High] Double Subtraction of Delegator Effective Stake on Exit Can Freeze VET and Break Reward Distribution
* \#59756 \[SC-High] Exiting delegators' stakes can be bricked permanently by the validator signaling an exit after them in the same period
* \#59904 \[SC-High] It's possible to decrease twice delegator stake in certain conditions
* \#60027 \[SC-High] Stuck funds for the later delegators due to an edge case led to double decreasing effective stakes
* \#60004 \[SC-High] Double-Decrease Effective Stake Bug in \`unstake()\`
* \#60102 \[SC-High] Exited delegator could keep claiming rewards stealing them from active delegators which would then lead to freeze of funds
* \#59742 \[SC-High] User Funds get stucked in the contract when validators exits.
* \#60125 \[SC-High] Moving delegations from one validator to another validator will not be possible in exit case for validator 1
* \#59709 \[SC-High] Post-exit Rewards Overpayment (Theft of Unclaimed Yield) Due to Misclamped Claim Window in Stargate
* \#59361 \[SC-High] Off-by-one in \`\_claimableDelegationPeriods\` allows claimRewards() to pay for periods after delegation end — Over-claim / Theft of unclaimed yield
* \#59866 \[SC-High] The delegator's rewards in period 1 cannot be claimed
* \#60028 \[SC-High] A delegator who has requested an exit continues to accumulate rewards
* \#60586 \[SC-High] Incorrect Double Reduction of Effective Stake in Stargate.sol
* \#59665 \[SC-High] Delegators Can Claim Rewards Beyond Delegation End
* \#60169 \[SC-High] Exited Delegations Can Continue to Claim Rewards Due to Logic Fall-through in \`\_claimableDelegationPeriods\`
* \#59727 \[SC-High] Double-Decrease DoS on Exit → Permanent Unstake Revert
* \#59951 \[SC-High] In special cases \`delegatorsEffectiveStake\` may decrease twice and cause staked funds to become locked
* \#60310 \[SC-High] Incorrect Boundary Check in \`\_claimableDelegationPeriods\` Allows Claiming Rewards Beyond Delegation End Period
* \#60080 \[SC-High] Unstake Exit Requests Can Either Lock Funds or Silently Double-Deduct Effective Stake After Validator Exit
* \#59809 \[SC-High] User balances are permanently frozen in specific delegation scenarios
* \#60516 \[SC-High] Incorrect Boundary Check in \`\_claimableDelegationPeriods\` Allows Claiming Rewards Beyond Delegation End Period
* \#60192 \[SC-High] Users can claim delegation rewards after exit (endPeriod) has passed
* \#60553 \[SC-High] The delegator and the validator both exiting consecutively, could lead to underflow in the unstake() and delegate() and stuck staked VET.
* \#60265 \[SC-High] The Attacker can still claim rewards after Exiting From validator
* \#59850 \[SC-High] users funds stuck in the contract permanently
* \#60210 \[SC-High] During a validator EXIT, users will be unable to unstake due to underflow
* \#59564 \[SC-High] Double-calling \`\_updatePeriodEffectiveStake\` during the exit flow makes \`unstake\` revert, trapping staked VET.
* \#60534 \[SC-High] A delegator who signals exit and waits for the validator to finish its period can no longer withdraw in the \`unstake\` function causing permanent loss of funds
* \#60548 \[SC-High] An Exited delegator who has not \`unstaked\` or \`delegated to a validator\`, will be DOS'ed if a validator exits.
* \#60419 \[SC-High] Double Decrease of Effective Stake Leads to DoS and Permanent Loss of Funds
* \#60400 \[SC-High] Off-by-one in claimableDelegationPeriods lets claims beyond exit
* \#60282 \[SC-High] Last delegators for an exited validator may be DoSed from re-delegating or unstaking due to incorrect accounting of period effective stake
* \#59863 \[SC-High] Over-claim of delegation rewards after exit

</details>

<details>

<summary>Medium</summary>

* \#60241 \[SC-Medium] Permanent freezing of staked funds caused by accumulation with zero rewards
* \#59570 \[SC-Medium] Access Control Bypass in unstake() Leads to Permanent Freezing of Funds
* \#59997 \[SC-Medium] \`claimRewards\` Fails to Update State for Zero-Value Periods, Causing Permanent Fund Freeze in \`unstake\`
* \#60466 \[SC-Medium] MaxClaimablePeriodsExceeded Lock — Zero-Reward Backlog Permanently Locks NFTs
* \#60539 \[SC-Medium] Critical Withdraw DoS: Zero-Reward Validators Cause Permanent User Fund Lock via Broken Reward-Claim Logic

</details>

<details>

<summary>Low</summary>

* \#60318 \[SC-Low] Zero-Cost Boost Bypass for New Levels
* \#60079 \[SC-Low] Critical Historical State Corruption via Stale Checkpoints Leads to Permanent Loss of Future Yield
* \#60597 \[SC-Low] \`hasRequestedExit\` Returns True for not just Requested Exits but also Delegations That Are Already Exited
* \#59795 \[SC-Low] Free Boosts for Levels Added After V3
* \#60386 \[SC-Low] Missing setter for boostPricePerBlock after adding new NFT levels can allow users to bypass intended staking boost
* \#60289 \[SC-Low] Misconfigured Level With maturityBlocks = 0 Allows Skip of Maturity Requirements and Backrun Minting
* \#60578 \[SC-Low] Zero Boost Fee for Newly Added Levels Lets Users Skip Maturity for Free and Avoid Paying Intended VTHO Boost Cost
* \#59814 \[SC-Low] StargateNFT.sol::addLevel function not implement updateLevelBoostPricePerBlock
* \#59841 \[SC-Low] The newly added level cannot have its boost price set because the \`updateLevelBoostPricePerBlock\` function is not exposed
* \#60259 \[SC-Low] Malicious User can bypass maturity period for Newly added levels
* \#60171 \[SC-Low] Levels Added After Deployment Lack Boost Price Initialization, Resulting in Free Boosting
* \#60593 \[SC-Low] No Mechanism to Set \`boostPricePerBlock\` for Levels Added After Initialization

</details>

<details>

<summary>Insight</summary>

* \#59244 \[SC-Insight] Missing Event emission on critical state change
* \#60450 \[SC-Insight] Code optimizations and enhancemets for efficient gas usage in several functions
* \#60527 \[SC-Insight] DelegationExitRequested event emits inconsistent exit period values
* \#59844 \[SC-Insight] Incorrect and misleading events when adding levels in \`StargateNFT\`
* \#59411 \[SC-Insight] Inconsistency in \`migrateTokenManager\` in terms of the permitted caller
* \#60525 \[SC-Insight] LevelCirculatingSupplyUpdated not emitted during supply changes
* \#60335 \[SC-Insight] Missing or misleading code comments causes confusion and may lead to unnecessary code changes
* \#60149 \[SC-Insight] \[REVISED] Missing input validation in \`addLevels\` can break multiple staking tier invariant in \`StartgateNFT\`
* \#59993 \[SC-Insight] Unnecessary Call to Get Balance In \`MintingLogic::boostOnBehalfOf()\`
* \#60023 \[SC-Insight] Unchecked address(0) Validator in unstake()

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#60318 \[SC-Low] Zero-Cost Boost Bypass for New Levels
* \#60334 \[SC-High] Unstake permanently reverts when validator exits after delegator exit (double-decrease of effective stake)
* \#60241 \[SC-Medium] Permanent freezing of staked funds caused by accumulation with zero rewards
* \#59244 \[SC-Insight] Missing Event emission on critical state change
* \#59316 \[SC-High] Off-by-One Unlocks Infinite VTHO Reward Drain from Ghost Stakes
* \#60311 \[SC-High] Double effective-stake decrement freezes unstake permanently after validator exit
* \#60079 \[SC-Low] Critical Historical State Corruption via Stale Checkpoints Leads to Permanent Loss of Future Yield
* \#60081 \[SC-High] Exited delegator can continue to accrue and claim delegation rewards
* \#60298 \[SC-High] Duplicate effectiveStake decrement path bricks unstake/re-delegate
* \#60372 \[SC-High] Double-Decrement Bug — Effective Stake Underflow Permanently Locks Funds
* \#60373 \[SC-High] Incorrect effective stake decrement when validator exits causes permanent freezing of user stake
* \#59421 \[SC-High] Theft of Unclaimed Yield via Incorrect Period Range Calculation and Lack of Per-User Effective Stake Tracking
* \#60150 \[SC-High] Off-by-one in claim window lets exited delegations harvest post-exit rewards
* \#60151 \[SC-High] Double Reduction of Effective Stake can lead to Stuck Delegations.
* \#59443 \[SC-High] rithmetic Underflow in Effective Stake Accounting Causes Permanent Loss of Funds
* \#60533 \[SC-High] overlap which will lead to loss of fund
* \#60154 \[SC-High] Exited delegations can continue claiming VTHO rewards for future periods
* \#59570 \[SC-Medium] Access Control Bypass in unstake() Leads to Permanent Freezing of Funds
* \#59997 \[SC-Medium] \`claimRewards\` Fails to Update State for Zero-Value Periods, Causing Permanent Fund Freeze in \`unstake\`
* \#59723 \[SC-High] Double-decrease after exit + validator EXITED leads to underflow and permanent freeze
* \#59730 \[SC-High] Permanent DoS - Users Cannot Unstake After Double Exit Scenario
* \#59733 \[SC-High] Post-exit delegations can drain future rewards
* \#60592 \[SC-High] users are unable to unstake under certain conditions
* \#60450 \[SC-Insight] Code optimizations and enhancemets for efficient gas usage in several functions
* \#60069 \[SC-High] Incorrect claimable period calculation Leading to attacker keep claiming even after exiting the delegation.
* \#60049 \[SC-High] Double Effective Stake Decrement Locks Delegators : Unstake Reverts Due to Duplicate EffectiveStake Decrements in Exit Flow
* \#60426 \[SC-High] Rewards Accounting Off-By-One / Skipped/Double Period Exploit leads to "Direct loss of user funds" via incorrect reward distribution; "Theft of unclaimed yield", misallocation of VT...
* \#59752 \[SC-High] Off-by-one bug in \_claimableDelegationPeriods allows claiming yield for periods after exit
* \#59563 \[SC-High] Exited Delegators Can Claim Rewards Indefinitely After Exit
* \#60429 \[SC-High] Double-Decrease of Effective Stake Prevents Delegators from Unstaking
* \#60431 \[SC-High] Unauthorized VTHO reward claims after delegation exit
* \#60597 \[SC-Low] \`hasRequestedExit\` Returns True for not just Requested Exits but also Delegations That Are Already Exited
* \#59776 \[SC-High] Exited delegators can over‑claim VTHO rewards for post‑exit periods due to off‑by‑one error in \_claimableDelegationPeriods
* \#60557 \[SC-High] Double Decrement of Effective Stake in unstake() leads to DoS and Permanent Fund Lock
* \#59795 \[SC-Low] Free Boosts for Levels Added After V3
* \#59802 \[SC-High] Double subtraction of validator effective stake will permanently lock other delegators’ staked VET
* \#60019 \[SC-High] Off-by-one in Stargate.sol \_claimableDelegationPeriods lets exited NFTs siphon validator rewards, leading to protocol insolvency
* \#59919 \[SC-High] Loss of funds - Delegators can claim rewards for periods where they had no stake
* \#60173 \[SC-High] the Phantom Claimable Periods Can Permanently Lock The Staked VET for Ended Delegations
* \#60466 \[SC-Medium] MaxClaimablePeriodsExceeded Lock — Zero-Reward Backlog Permanently Locks NFTs
* \#60386 \[SC-Low] Missing setter for boostPricePerBlock after adding new NFT levels can allow users to bypass intended staking boost
* \#59615 \[SC-High] Off-by-one error in period boundary check allows theft of unclaimed yield after delegation exit
* \#60289 \[SC-Low] Misconfigured Level With maturityBlocks = 0 Allows Skip of Maturity Requirements and Backrun Minting
* \#60506 \[SC-High] Double delegatorsEffectiveStake Decrease Permanently Prevents Single NFT from Unstaking
* \#60527 \[SC-Insight] DelegationExitRequested event emits inconsistent exit period values
* \#59657 \[SC-High] Delegators Lose First Reward Period When Delegating to Pending Validators
* \#60470 \[SC-High] Double-Decrease of Validator Stake in Stargate.sol
* \#59358 \[SC-High] Off by One Error in Reward Claim Logic Allows Delegators to Steal VTHO for Periods After Delegation Ended
* \#59386 \[SC-High] Fund freeze from double stake subtraction (when validator exits)
* \#60575 \[SC-High] Double Subtraction of Delegator Effective Stake on Exit Can Freeze VET and Break Reward Distribution
* \#60578 \[SC-Low] Zero Boost Fee for Newly Added Levels Lets Users Skip Maturity for Free and Avoid Paying Intended VTHO Boost Cost
* \#59756 \[SC-High] Exiting delegators' stakes can be bricked permanently by the validator signaling an exit after them in the same period
* \#59844 \[SC-Insight] Incorrect and misleading events when adding levels in \`StargateNFT\`
* \#59904 \[SC-High] It's possible to decrease twice delegator stake in certain conditions
* \#60027 \[SC-High] Stuck funds for the later delegators due to an edge case led to double decreasing effective stakes
* \#60004 \[SC-High] Double-Decrease Effective Stake Bug in \`unstake()\`
* \#60102 \[SC-High] Exited delegator could keep claiming rewards stealing them from active delegators which would then lead to freeze of funds
* \#59742 \[SC-High] User Funds get stucked in the contract when validators exits.
* \#60125 \[SC-High] Moving delegations from one validator to another validator will not be possible in exit case for validator 1
* \#59709 \[SC-High] Post-exit Rewards Overpayment (Theft of Unclaimed Yield) Due to Misclamped Claim Window in Stargate
* \#59361 \[SC-High] Off-by-one in \`\_claimableDelegationPeriods\` allows claimRewards() to pay for periods after delegation end — Over-claim / Theft of unclaimed yield
* \#59866 \[SC-High] The delegator's rewards in period 1 cannot be claimed
* \#60028 \[SC-High] A delegator who has requested an exit continues to accumulate rewards
* \#60586 \[SC-High] Incorrect Double Reduction of Effective Stake in Stargate.sol
* \#59665 \[SC-High] Delegators Can Claim Rewards Beyond Delegation End
* \#60169 \[SC-High] Exited Delegations Can Continue to Claim Rewards Due to Logic Fall-through in \`\_claimableDelegationPeriods\`
* \#59814 \[SC-Low] StargateNFT.sol::addLevel function not implement updateLevelBoostPricePerBlock
* \#59727 \[SC-High] Double-Decrease DoS on Exit → Permanent Unstake Revert
* \#59951 \[SC-High] In special cases \`delegatorsEffectiveStake\` may decrease twice and cause staked funds to become locked
* \#59841 \[SC-Low] The newly added level cannot have its boost price set because the \`updateLevelBoostPricePerBlock\` function is not exposed
* \#60310 \[SC-High] Incorrect Boundary Check in \`\_claimableDelegationPeriods\` Allows Claiming Rewards Beyond Delegation End Period
* \#60259 \[SC-Low] Malicious User can bypass maturity period for Newly added levels
* \#60080 \[SC-High] Unstake Exit Requests Can Either Lock Funds or Silently Double-Deduct Effective Stake After Validator Exit
* \#59411 \[SC-Insight] Inconsistency in \`migrateTokenManager\` in terms of the permitted caller
* \#60171 \[SC-Low] Levels Added After Deployment Lack Boost Price Initialization, Resulting in Free Boosting
* \#59809 \[SC-High] User balances are permanently frozen in specific delegation scenarios
* \#60525 \[SC-Insight] LevelCirculatingSupplyUpdated not emitted during supply changes
* \#60516 \[SC-High] Incorrect Boundary Check in \`\_claimableDelegationPeriods\` Allows Claiming Rewards Beyond Delegation End Period
* \#60335 \[SC-Insight] Missing or misleading code comments causes confusion and may lead to unnecessary code changes
* \#60149 \[SC-Insight] \[REVISED] Missing input validation in \`addLevels\` can break multiple staking tier invariant in \`StartgateNFT\`
* \#60192 \[SC-High] Users can claim delegation rewards after exit (endPeriod) has passed
* \#60593 \[SC-Low] No Mechanism to Set \`boostPricePerBlock\` for Levels Added After Initialization
* \#60553 \[SC-High] The delegator and the validator both exiting consecutively, could lead to underflow in the unstake() and delegate() and stuck staked VET.
* \#60265 \[SC-High] The Attacker can still claim rewards after Exiting From validator
* \#59850 \[SC-High] users funds stuck in the contract permanently
* \#59993 \[SC-Insight] Unnecessary Call to Get Balance In \`MintingLogic::boostOnBehalfOf()\`
* \#60210 \[SC-High] During a validator EXIT, users will be unable to unstake due to underflow
* \#59564 \[SC-High] Double-calling \`\_updatePeriodEffectiveStake\` during the exit flow makes \`unstake\` revert, trapping staked VET.
* \#60534 \[SC-High] A delegator who signals exit and waits for the validator to finish its period can no longer withdraw in the \`unstake\` function causing permanent loss of funds
* \#60539 \[SC-Medium] Critical Withdraw DoS: Zero-Reward Validators Cause Permanent User Fund Lock via Broken Reward-Claim Logic
* \#60548 \[SC-High] An Exited delegator who has not \`unstaked\` or \`delegated to a validator\`, will be DOS'ed if a validator exits.
* \#60419 \[SC-High] Double Decrease of Effective Stake Leads to DoS and Permanent Loss of Funds
* \#60400 \[SC-High] Off-by-one in claimableDelegationPeriods lets claims beyond exit
* \#60282 \[SC-High] Last delegators for an exited validator may be DoSed from re-delegating or unstaking due to incorrect accounting of period effective stake
* \#60023 \[SC-Insight] Unchecked address(0) Validator in unstake()
* \#59863 \[SC-High] Over-claim of delegation rewards after exit

</details>
