# Alchemix V3

## Reports by Severity

<details>

<summary>Critical</summary>

* \#57774 \[SC-Critical] Redemption Earmark Mechanism Can Be Permanently Blocked via Single-Block Earmark Calls
* \#58793 \[SC-Critical] Repayment Fee Overpayment from Global Collateral Pool
* \#57093 \[SC-Critical] Potential Locked Funds Due to Partial Redeem Shortfall and miss calculation, lead to user loss their myt token forever.
* \#58276 \[SC-Critical] Uncapped \`feeInYield\` in \`\_resolveRepaymentFee\` allows for collateral theft from other depositors
* \#58518 \[SC-Critical] Liquidation will steal Repayment Fee from Innocent Users Funds
* \#57122 \[SC-Critical] Mismatch Between Capped Fee and Returned Fee in \`\_resolveRepaymentFee\`
* \#58531 \[SC-Critical] QueryGraph function Zero-Return Bug causing Tracking Earmarking Failure Over Progressive Block Intervals
* \#58288 \[SC-Critical] Incorrect Fee Payment Logic Leads to Underpayment
* \#58306 \[SC-Critical] Repayment Fee Not Adjusted for Insufficient Collateral
* \#57360 \[SC-Critical] Unreconciled repayment fee transfer enables MYT overpayment and TVL inflation
* \#56435 \[SC-Critical] AlchemistV3: repayment‑only liquidation pays liquidator from pool (fee leak) → theft of unclaimed yield
* \#58688 \[SC-Critical] \`AlchemistV3::\_liquidate\` can steal other users’ collateral
* \#58320 \[SC-Critical] Incorrect Fee Return Value in \_resolveRepaymentFee Enables Fund Theft Under Extreme Conditions
* \#58683 \[SC-Critical] There is an issue in earmarked debt eeduction in the repay() can causes a permanent fund freeze
* \#58447 \[SC-Critical] Unfair Collateral Loss Through Socialized Redemption Costs
* \#58573 \[SC-Critical] AlchemistV3 Repayment Fee Cross-Account Theft Vulnerability
* \#58413 \[SC-Critical] Attacker/user can prevent Earmark from updating the earnmarkweight causing the transmuter action to repay det gradually to fail for all users
* \#56732 \[SC-Critical] Incorrect boundary condition in queryGraph leads to systematic under-earmarking and transmuter redemption fund loss
* \#56385 \[SC-Critical] Repayment fee can be paid from the pool even when the account has no collateral left
* \#57590 \[SC-Critical] Double-counted Transmuter cover in \`redeem()\` allows overstated redemptions and potential over-withdraw/over-borrow
* \#58724 \[SC-Critical] Partial Redemption Burns Full Position — Accounting Desynchronization and Potential Underpayment in Transmuter.claimRedemption()
* \#56794 \[SC-Critical] Liquidators can be overpaid due to accounting error
* \#58507 \[SC-Critical] Repayment fee after forceRepay could result in socialized loss during global undercollateralization
* \#56965 \[SC-Critical] AlchemistV3 handling of added Transmuter coverage includes an error that enables an attacker to cause protocol insolvency
* \#56519 \[SC-Critical] Unchecked repayment fee transfer in \`\_liquidate\` pays liquidators from other users’ collateral
* \#58280 \[SC-Critical] Repayment's fee is charged from other users causing the contract to fail when the myt total balance of a user cannot cover the fee
* \#57053 \[SC-Critical] Integer Division Precision Loss in normalizeDebtTokensToUnderlying Leads to Permanent Collateral Locking
* \#57662 \[SC-Critical] portion of users alAsset amount that staked in transmuter can be lost forever when \`amount > cumulativeEarmarked\`
* \#58626 \[SC-Critical] Repayment Fee Overpayment in Liquidation Repay-Only Path
* \#58163 \[SC-Critical] Total loss of user Funds in claim redemption
* \#58757 \[SC-Critical] Forgotten cover in \_earmark() causes systematic over-earmarking and temporary freezing of user collateral
* \#58125 \[SC-Critical] Repayment Fee Overpayment from Pooled Collateral
* \#58443 \[SC-Critical] Incorrect Consumption of Yield Cover in redeem, Leading to Reuse of Accrued Yield
* \#58772 \[SC-Critical] \`\_resolveRepaymentFee\` overpays liquidators when collateral is gone, letting attackers drain MYT
* \#58323 \[SC-Critical] The Alchemist::burn function experiences precision loss, resulting in the avoidance of protocol fees
* \#57582 \[SC-Critical] Calling \_earmark one block apart skips the block's earmark value
* \#58689 \[SC-Critical] Incorrect deduction logic in \`AlchemistV3::redeem()\` may lead to insufficient contract collateral
* \#56702 \[SC-Critical] \`claimRedemption\` would not return all alAsset that is not get converted to MYT in some case
* \#58270 \[SC-Critical] incorrect handling of debt cover in redeem can affect early liquidation and incorrectly sync accounts
* \#57916 \[SC-Critical] Repay removes earmark, meant to be reducing debt while collateral is still reduced
* \#56491 \[SC-Critical] User Collateral Loss Triggered by setMinimumCollateralization Update
* \#57617 \[SC-Critical] Protocol-paid repayment fee transfer allows draining of protocol MYT (yield)
* \#57973 \[SC-Critical] repay doesnt set lastTransmuterTokenBalance leading to the same balance covering earmark twice
* \#58301 \[SC-Critical] Accounting Issue in Liquidation Logic After Force Repay we charge repayment fee even if collateral balanc cannot account for it
* \#58036 \[SC-Critical] Incorrect Fee Deduction May Drain Collateral Pool When Account Balance Is Insufficient
* \#57852 \[SC-Critical] Old borrowers steal from new borrowers after redemptions are claimed
* \#57066 \[SC-Critical] A malicious actor can keep calling \`poke\` at every block to prevent collateral earmarking exposing transmuter users to delayed redemptions and loss of funds
* \#58564 \[SC-Critical] Earmarked funds fail to accumulate when \_earmark is called in consecutive blocks
* \#58544 \[SC-Critical] it is possible to underflow on \`\_sync\` making positions bricked forever
* \#56365 \[SC-Critical] Liquidation Fee Overdraft Drains Pooled Collateral
* \#56622 \[SC-Critical] Repayment Fee Overpays Liquidators Using Pooled Collateral After \_forceRepay
* \#56740 \[SC-Critical] Unbounded Liquidation Fee Allows Theft of Shared Collateral
* \#57330 \[SC-Critical] \_resolveRepaymentFee returns initial fee when fee is greater collateral balance
* \#58338 \[SC-Critical] AlchemistV3 Repayment Fee Can Exceed Remaining Collateral Leading to Position Insolvency
* \#56798 \[SC-Critical] Flash-Vote Exploit Drains All Funds via AlchemistAllocator
* \#58131 \[SC-Critical] Rounding Errors in Debt-to-Collateral Conversions Allow Attackers to Drain Protocol Assets
* \#58101 \[SC-Critical] Repayment-only liquidation overpays fee from pooled collateral
* \#57101 \[SC-Critical] Same-block earmark early-exit leaves stale transmuter balance, causing under-earmarking
* \#58127 \[SC-Critical] Users can invoke the poke() function whenever the lastEarmarkDebtBlock is exactly one block behind the current block.number which lead to affecting users earmarked debt
* \#58399 \[SC-Critical] Precision Loss in badDebtRatio Calculation Causes Overpayment and DOS
* \#57138 \[SC-Critical] Protocol subsidizes repayment fees during liquidation
* \#57441 \[SC-Critical] Repay-Only Fee Drain in AlchemistV3
* \#56555 \[SC-Critical] User can avoid Bad Debt ratio scaling when claiming redeem, leading to protocol insolvency
* \#57587 \[SC-Critical] \_earmark() reduction of transmuterDifference does not always account for the full transmuter balance diff which can cause permanent earmark to accrue in Alchemist
* \#58234 \[SC-Critical] There is a problem related ot Repayment Fee Overpayment can lead to Protocol Insolvency
* \#58464 \[SC-Critical] Repayment fee paid from protocol funds when user collateral is depleted
* \#58138 \[SC-Critical] Liquidator fees could surpass the user remaining collateral resulting in protocol insolvency

</details>

<details>

<summary>High</summary>

* \#58755 \[SC-High] Users position that are synced at certain times overestimate collateralBalance of the position
* \#56571 \[SC-High] Inflated claim payouts from double-counted MYT after liquidation
* \#57036 \[SC-High] Unconditional Debt Reduction Before Protocol Fee Check in Force Repayment
* \#57041 \[SC-High] Deallocation Accounting Mismatch Between Vault and Adapter
* \#58004 \[SC-High] Protocol Insolvency from \`cumulativeEarmarked\` During \`\_forceRepay()\`
* \#58526 \[SC-High] Missing accounting update in liquidation functions leads to permanent DOS on deposits
* \#58466 \[SC-High] Liquidation Fee Payment Failure Due to Redundant wrong Collateral Check
* \#58471 \[SC-High] Accounting error in \`\_forceRepay\`/\`\_doLiquidation\` overstates TVL, enabling under-scaled redemptions and potential insolvency
* \#58474 \[SC-High] Liquidator will bypass liquidation fees affecting protocol revenue
* \#56827 \[SC-High] Missing Global Earmark Reduction in \_forceRepay
* \#57633 \[SC-High] Block-gated \_earmark() call in redeem() nullifies prefunded Transmuter cover on the first redemption of each block, leading to collateral overpayment and potential protocol insolvency
* \#57288 \[SC-High] Flawed rounding logic in TokeAutoEth deallocate function causes permanent freezing of funds
* \#57172 \[SC-High] Missing \_mytSharesDeposited Decrements in Liquidation Flows Causes Accounting Divergence
* \#58450 \[SC-High] Missing Transmuter Balance Update After Redemption Blocks Future Earmarking and Underfunds Redemptions
* \#57308 \[SC-High] AlchemistV3 does not update \`\_mytSharesDeposited\` when performing liquidation, causing global accounting and liquidation logic mismatch
* \#56402 \[SC-High] \`killSwitch\` leaves vault assets stranded and blocks withdrawals
* \#58113 \[SC-High] StargateEthPoolStrategy.realAssets return false real assets
* \#58394 \[SC-High] MEV opportunity because no slippage protection in TokeAutoEthStrategy
* \#57510 \[SC-High] Stale Locked Collateral Tracking During Price Appreciation Causes Disproportionate Redemption Losses
* \#57345 \[SC-High] Missing cumulativeEarmarked Decrement in \_forceRepay Breaks Earmarking Invariant Leading to Unfair Redemption Burden Distribution
* \#58547 \[SC-High] Mismatched Accounting and Transfer for Capped Fees
* \#58337 \[SC-High] Incorrect Handling of cumulativeEarmarked in \_forceRepay leads to inflated survival accumulator.
* \#57745 \[SC-High] Syn fails to update the rawLocked valuation leading to a loss of fund for users with rawlock > 0 when total lock become 0.
* \#58572 \[SC-High] Liquidation of account \`\`collateral\`\` doesn't subtract \`\`\_mytSharesDeposited\`\` which creates bad debt in the system and causes \`\`insolvency\`\`.
* \#58409 \[SC-High] ## \[HIGH] Arithmetic Underflow in \`MYTStrategy.sol\`'s \`deallocate()\` Check Prevents Yield Withdrawal
* \#57954 \[SC-High] Lackf of tracking of excess cover in \`\_earmark\` function leads to permanent loss of cover value and stuck user positions.
* \#57460 \[SC-High] Protocol fails to subtract fee from total locked when burning and repaying
* \#58425 \[SC-High] Missing slippage protection when depositing to TokeAuto strategies
* \#57559 \[SC-High] Missing \`\_mytSharesDeposited\` Decrement in Liquidation Paths Enables Theft of Unclaimed Yield and Protocol Insolvency
* \#58387 \[SC-High] Liquidator Fee in the \_doliquidation Function Withheld When Collateral Is Exhausted Leading to Seized Fee Trapped in Protocol
* \#57447 \[SC-High] Untracked MYT outflows inflate TVL causing liquidation suppression
* \#57148 \[SC-High] \`\_mytSharesDeposited\` variable is not correctly updated during liquidations, leading to wrong assumptions and incorrect bad debt calculation in the Transmuter.
* \#58763 \[SC-High] Accounting is broken when redeem() is bypassed due to Transmuter balance
* \#58398 \[SC-High] No Slippage Protection on Large Allocation Deposits
* \#57751 \[SC-High] There is a problem related to forced liquidation branch and this creates issue thatk cna drains protocol backing
* \#56727 \[SC-High] Underlying increase in forced repayments leads to insolvency
* \#56673 \[SC-High] Zero-cost fee farming via forced earmarked repayment
* \#58736 \[SC-High] Missing TVL Accounting in \_forceRepay and \_doLiquidation Leads to Protocol Insolvency
* \#58274 \[SC-High] Liquidation fee logic in \`\_doLiquidation\` strands liquidator rewards when balance is exhausted, freezing funds
* \#56714 \[SC-High] Accounting Invariant Violation in \_forceRepay Leads to Protocol Insolvency
* \#57730 \[SC-High] Liquidation Does Not Decrease mytSharesDeposited
* \#56363 \[SC-High] \`\_mytSharesDeposited\` not correctly updated in all cases, leading to incorrect protocol collateralization and reduced liquidation incentives
* \#56395 \[SC-High] Accounting desync in liquidation outflows leads to artificial deposit cap exhaustion and denial-of-Service on recapitalization
* \#58236 \[SC-High] Accounting mismatch: \`\_forceRepay\`/\`\_doLiquidation\` fail to decrement \`\_mytSharesDeposited\`, locking deposit capacity and overstating collateral
* \#58098 \[SC-High] There is a problem from ledger TVL sesync inliquidations cause a under-liquidation and systemic insolvency risk
* \#56824 \[SC-High] Missing update to \_mytSharesDeposited during liquidation
* \#58354 \[SC-High] \`\_forceRepay\` does not decrement \`\_mytSharesDeposited\`, causing a temporal blocking of new deposits
* \#58771 \[SC-High] Incorrect Tracking of Total Deposited Yield Tokens (\_mytSharesDeposited) in Liquidation and Force Repayment Paths
* \#56552 \[SC-High] Liquidation fee misrouting in AlchemistV3.\_doLiquidation() leads to theft of unclaimed yield (liquidator fee stranded)
* \#57530 \[SC-High] Stale TVL Accounting in Liquidations Leads to Protocol Insolvency
* \#57668 \[SC-High] Missing collateral tracking update during liquidation leads to inflated total value calculation and delayed under-collateralization protection
* \#58192 \[SC-High] TokeAutoEth Strategy Tokens Locked When AutoPool(router) Enforces maxDeposit Cap
* \#57129 \[SC-High] Missing \_mytSharesDeposited Decrement in Liquidation Functions Causes Permanent TVL Inflation
* \#58358 \[SC-High] Mismatched CollateralWeight and rawLocked Causes Incorrect Collateral Removal in Sync
* \#57941 \[SC-High] Incorrect handling of deallocate return val causes any interest gains in a strategy to become unclaimable and permanently locked
* \#58269 \[SC-High] Liquidator Fee Not Paid When Fee Equals Surplus
* \#56672 \[SC-High] Inconsistent MYT share accounting leads to under-liquidation and solvency risk
* \#57907 \[SC-High] Incorrect forced-repayment accounting allows debt forgiveness and frees locked collateral (systemic loss)
* \#57585 \[SC-High] AlchemistV3 does not properly update CDP collateralBalance when redemptions exceed \_totalLocked which enables some CDPs to over-withdraw collateral on account of others
* \#58347 \[SC-High] Accounting Drift Due to Missing \`\_mytSharesDeposited\` Decrements During Liquidation
* \#57930 \[SC-High] Allocation tracking underflow in strategy deallocation Leads to protocol insolvency
* \#56817 \[SC-High] ForceRepay doesn't decrement \_mytSharesDeposited, inflating TVL
* \#58112 \[SC-High] A malicious user can avoid getting penalized upon a Transmuter redemption by depositing and withdrawing collateral in the Alchemist
* \#58628 \[SC-High] Attackers Can Avoid Redemption Losses By Temporarily Burning and Re-Borrowing The Debt
* \#57088 \[SC-High] Unscaled collateral accounting in redeem lets users withdraw more than intended
* \#57726 \[SC-High] AlchemistV3: MYT TVL accounting drift on liquidation/forceRepay blocks deposits via depositCap (Medium — Smart contract unable to operate due to lack of token funds)
* \#58396 \[SC-High] Total locked is not cleared proportionally to the total debt, this forces the collateral weight to become incorrect and new users transmuter redeem repayment will repay more debt fo...
* \#57950 \[SC-High] Unit Mismatch in \_addDebt() Collateralization Check Allows Unbacked Debt Issuance and Protocol Insolvency
* \#57963 \[SC-High] Incorrect \_mytSharesDeposited Accounting in liquidate() Allows Theft of User Funds via Corrupted Bad Debt Ratio
* \#58070 \[SC-High] Forced-repay accounting lets borrowers erase debt without paying equivalent assets (protocol deficit / insolvency)
* \#57632 \[SC-High] Inflated TVL in \_mytSharesDeposited hides protocol insolvency
* \#57970 \[SC-High] \_forceRepay Leaves \`cumulativeEarmarked\` Stale
* \#58435 \[SC-High] Systemic Accounting Bug Leads to Protocol Insolvency
* \#57793 \[SC-High] \`cumulativeEarmarked\` variable is not updated in \`\_forceRepay\` function, breaking core internal logic and leading to user funds being stuck.
* \#57102 \[SC-High] TVL Overstatement from \_mytSharesDeposited Desync Enables Softened Liquidations & No‑Haircut Over‑Redemptions (Transmuter)
* \#58086 \[SC-High] Mis-accounting of MYT outflows inflates TVL, distorts collateralization, and can DoS deposits/liquidations
* \#58442 \[SC-High] Liquidation Breaks Core Accounting Invariant: Missing cumulativeEarmarked Update in \_forceRepay() Causes Permanent State Drift
* \#57995 \[SC-High] Missing Slippage Protection in TokeAutoUSDStrategy Allocation Function Leads to Permanent Value Loss
* \#56678 \[SC-High] Missing Internal MYT Shares Accounting in Liquidation Functions Causes Deposit Blocking and Protocol Insolvency Risk Through Inflated TVL Calculations
* \#57464 \[SC-High] Incorrect accounting in stargate strategy causes protocol insolvency and user liquidations
* \#58369 \[SC-High] Missing \_mytSharesDeposited Decrements in \_forceRepay/\_doLiquidation Leads to Smart Contract Unable to Operate Due to Lack of Token Funds
* \#57152 \[SC-High] Assets Permanently Locked Due to KillSwitch Flag
* \#58792 \[SC-High] the cumulativeEarmark does not decrease in \_forceRepay which lead to transfer more collateral from users even when all earmark debt cleared which breaks the alchemix v3 core logic
* \#58019 \[SC-High] Flawed killSwitch Implementation in MYTStrategy Leads to Permanent Loss of Funds
* \#57860 \[SC-High] Incorrect \`\_mytSharesDeposited\` accounting inflates collateral and underreports bad debt, enabling insolvency
* \#57861 \[SC-High] Missing Slippage Protection in Tokemak Autopool Allocation Functions Leads to Direct Theft of User Funds
* \#58452 \[SC-High] MYTStrategy Allocation underflow in deallocate() when allocation + profits exceed old allocation
* \#57197 \[SC-High] Incorrect \_totalLocked Reduction
* \#57604 \[SC-High] Nominal accounting mismatch in Moonwell strategies leads to permanent locking of all generated yield
* \#57212 \[SC-High] \_totalLocked is not properly decremented in the redeem function causing system insolvency.
* \#58324 \[SC-High] Incorrect Return Value in \_deallocate Function Leads to Permanent Fund Locking in MYTStrategy Implementations
* \#58363 \[SC-High] Accounting Corruption in Liquidations Due to Missing Global Counter Update
* \#57972 \[SC-High] liquidation doesn't update \_mytSharesDeposited
* \#58472 \[SC-High] Liquidator Base Fee Seized but Not Paid Due to Post‑Deduction Balance Check
* \#56815 \[SC-High] Missing \`\_mytSharesDeposited\` Decrements in Internal Outflows cause TVL Inflation & Deposit DoS
* \#56359 \[SC-High] Permanent Deposit Freeze After \_forceRepay() Misaccounts Freed Shares
* \#58615 \[SC-High] \_mytSharesDeposited didn't get updated after \_forceRepay && \_doLiquidation called
* \#58275 \[SC-High] account.rawLocked not clear even when debt is clear
* \#57725 \[SC-High] # \`AlchemistV::liquidate\` is not updating the \`\_mytSharesDeposited\` which makes it inflated and can cause deposits DoS and liquidations malfunction that may cause protocol insolvency.
* \#58794 \[SC-High] Hardcoded 0 amount as the minSharesOut to depositMax(...) function call does not provide slippage protection
* \#57506 \[SC-High] force repay don't update cumulativeEarmarked variable
* \#56936 \[SC-High] Missing \`\_mytSharesDeposited\` decrements on repay/liquidation → TVL drift, false over‑collateralization, and deposit‑cap DoS
* \#56389 \[SC-High] \`\_mytSharesDeposited\` is not updated on liquidation outflows which could lead to solvency illusion and misreported global ratios
* \#58519 \[SC-High] Double Counting of Collateral Due to \`\_mytSharesDeposited\` not being updated during liquidations
* \#57760 \[SC-High] MytStrategy.\_allocate/\_deallocate doesnt account for profit and loss
* \#56776 \[SC-High] TVL Manipulation via Missing \_mytSharesDeposited Decrement in Liquidations
* \#57553 \[SC-High] \_mytSharesDeposited is not updated in liquidations which breaks bad debt ratio/alchemistCR calculations and causes failures in bad debt handling and liquidation handling
* \#58067 \[SC-High] Asymmetric deallocation in TokeAutoEthStrategy leads to permanent WETH funds stuck in strategy
* \#58754 \[SC-High] Missing \`\_mytSharesDeposited\` decrements in \`AlchemistV3\` \`\_forceRepay\`/\`\_doLiquidation\`
* \#58177 \[SC-High] Transmuter::claimRedemption cant update \_mytSharesDeposited leading to permanent underlying value state inside Alchemist
* \#57787 \[SC-High] asset can be transferred to strategies even when the killSwitch enabled without posibility to use this funds for allocation
* \#57189 \[SC-High] AlchemistCurator contract not implement setForceDeallocatePenalty
* \#56845 \[SC-High] The deposit will be reverted because \`\_mytSharesDeposited\` references an outdated value
* \#57918 \[SC-High] Incorrect \`totalLocked\` Collateral Accounting in AlchemistV3
* \#58658 \[SC-High] cumulativeEarmarked not updated
* \#56809 \[SC-High] Vulnerable redemption survival ratio in \_sync allows theft of alTokens
* \#56442 \[SC-High] Inflated \`\_totalLocked\` because vault yield accrual would skew \`\_collateralWeight\` calculation
* \#58035 \[SC-High] killSwitch early-return in strategy causes vault-to-adapter asset leakage, mis-accounting, and deallocation DOS
* \#56956 \[SC-High] Lack of slippage control in Tokemak strategies can make MYT suffer losses on allocation
* \#58010 \[SC-High] Slippage tolerance not enforced in TokeAutoUSDStrategy
* \#56560 \[SC-High] Liquidation base fee transfer is gated by a condition that’s usually false
* \#57883 \[SC-High] \_mytSharesDeposited Updates in Liquidation Functions Leads to Critical TVL Inflation
* \#58129 \[SC-High] Missing \_mytSharesDeposited Update in \_forceRepay() Causes Accounting Inconsistency which can DOS deposit and Liquidation
* \#57544 \[SC-High] \`\_mytSharesDeposited\` is not reduced upon fee transfers to protocol
* \#58116 \[SC-High] TVL Accounting Mismatch Leading to Protocol Insolvency
* \#58260 \[SC-High] Inconsistent collateral accounting where Force-Repay/Liquidation transfer out MYT without adjusting TVL
* \#57740 \[SC-High] EulerETH strategy will have WETH locked in the strategy contract
* \#58215 \[SC-High] Funds Can Become Permanently Stuck in Adapter When Kill Switch is Enabled
* \#58346 \[SC-High] \_forceRepay() fails to decrement cumulativeEarmarked, breaking earmark invariant and skewing redemptions
* \#58196 \[SC-High] AaveV3ARBUSDCStrategy strategy will have its reward stuck in Aave USDC
* \#57369 \[SC-High] Deallocation may revert due to an underflow
* \#56516 \[SC-High] allocate assets in killSwitch mode can lead to assets stuck on contract
* \#57825 \[SC-High] Forced repay cover enables double-counted debt reduction in redeem
* \#56923 \[SC-High] Missing cumulativeEarmarked Update in \_forceRepay Causes Incorrect Debt Accounting in AlchemistV3
* \#57678 \[SC-High] Liquidation fee is deducted from user but not paid to liquidator
* \#58150 \[SC-High] Missing Slippage Protection in \`TokeAutoUSDStrategy::\_allocate\` Leads to Direct Theft of User Funds via MEV Sandwich Attacks
* \#58266 \[SC-High] Partial liquidation strands base fee due to post-seizure balance check
* \#58782 \[SC-High] Rewards earned by EulerARBUSDCStrategy will not be withdrawable from Euler pool on Arbitrum
* \#57692 \[SC-High] AlchemistV3 Liquidation Fee Loss Vulnerability
* \#58524 \[SC-High] When liquidating, there are cases where the Fee is not paid to the liquidator.
* \#56757 \[SC-High] Incorrect leftover-collateral check blocks liquidator fee payment leading broken incentives & delayed deleveraging
* \#58742 \[SC-High] Liquidators will not earn fees in some cases
* \#58759 \[SC-High] Yield Stuck in Adapter Contracts Forever
* \#58781 \[SC-High] \_totalLocked Accounting Mismatch Leading to Token Balance Deficit in AlchemistV3
* \#58780 \[SC-High] WETH Yield will be locked on AaveWETH pool on Arbitrum
* \#57680 \[SC-High] PeapodsETHStrategy unable to withdraw yield from price share increase
* \#57476 \[SC-High] \_forceRepay() fails to decrement global cumulativeEarmarked
* \#58522 \[SC-High] Earmark consumes excess cover, inflating cumulativeEarmarked
* \#56975 \[SC-High] Liquidation Fee Trapping in AlchemistV3
* \#58383 \[SC-High] Due to \`cumulativeEarmarked\` not being updated in \`Alchemix::\_forceRepay\` user funds are locked longer due to slower debt decay and Calculation of System Collaterization Rate is Inc...
* \#58769 \[SC-High] \`\_forceRepay\` fails to decrement global \`cumulativeEarmarked\`, causing redemption accounting desynchronization and potential protocol-wide redemption halt
* \#58799 \[SC-High] \`\_forceRepay\` does not reduce \`cumulativeEarmarked\` which leads to wrong accounting: users debts are incorrectly higher which can cause wrongful liquidations
* \#57704 \[SC-High] Missing Global State Update in \_forceRepay Leads to Permanent Freezing of Unclaimed Yield
* \#57023 \[SC-High] Global earmark not reduced in \_forceRepay lets redeem() over-burn global debt (cross-account leakage, protocol insolvency)
* \#58635 \[SC-High] \`\`cumulativeEarmarked\`\` is not subtracted in \`\`\_forceRepay()\`\`.
* \#57532 \[SC-High] Assets are not accounted for when the contract is in killSwitch mode
* \#57849 \[SC-High] Funds gets stuck even when killswitch is enabled
* \#58723 \[SC-High] \`cumulativeEarmarked\` is not updated at \`\_forceRepay\`
* \#58534 \[SC-High] Zero Slippage Protection in Toke strategies Allocation
* \#58702 \[SC-High] No slippage provided in Auto strategy implementation will open room for MEV attacks
* \#56902 \[SC-High] Strategy Adapter \`AaveV3OPUSDCStrategy\` would not work well with aToken rebasing mechanism
* \#58287 \[SC-High] \_mytSharesDeposited is not updated on some token transfer
* \#58768 \[SC-High] \`\_mytSharesDeposited is not updated during liquidations, breaking core accounting
* \#56791 \[SC-High] Missing \`\_mytSharesDeposited\` Decrements in Token Transfers
* \#58207 \[SC-High] AlchemistV3 \_mytSharesDeposited Not Reduced When Repaid Collateral Sent to Transmuter
* \#57378 \[SC-High] Impossible to withdraw yield from strategies
* \#58530 \[SC-High] Protocol insolvency via stale \`\_totalLocked\`: zeroed \`\_totalLocked\` prevents \`\_collateralWeight\` update in \`redeem()\` leading to missed collateral haircut
* \#58061 \[SC-High] Incorrect collateral and fee Check in \_doLiquidation Allows Liquidator to loose fee.
* \#57196 \[SC-High] Artificially inflated \`\_mytSharesDeposited\` in \`AlchemixV3.sol\` deflates bad debt ratio in \`Transmuter.sol\`
* \#58502 \[SC-High] Deposit cap denial of service due to stale \_mytSharesDeposited during liquidation
* \#58491 \[SC-High] \_mytSharesDeposited Not Reduced on Liquidation, leading to Deposit Cap Bypass and potential insovency
* \#56628 \[SC-High] \`\_liquidate\` does not update \`\_mytSharesDeposited\` that is reduced by fees
* \#58395 \[SC-High] Repayment fee exit leaves \`\_mytSharesDeposited\` inflated, hiding protocol insolvency
* \#57533 \[SC-High] Inaccurate TVL Calculation Prevents Liquidations, Leading to Protocol Insolvency Risk
* \#58606 \[SC-High] Missing collateral accounting in liquidation leads to inflated bad debt calculations
* \#56545 \[SC-High] Force Repayment Leaves Stale Global Earmarks, Freezing Transmuter Redemptions
* \#56719 \[SC-High] The function \_forcerepay reduces debt before clamp, creating unbacked loan forgiveness and protocol insolvency
* \#57977 \[SC-High] Inconsistent rawLocked State of a user after subdebt Leads to Irrecoverable User Collateral Loss

</details>

<details>

<summary>Medium</summary>

* \#56751 \[SC-Medium] StargateEthPoolStrategy deallocate function redeem less weth than expected
* \#56692 \[SC-Medium] \`ZeroXSwapVerifier\` verification will always revert due to wrong hardcoded execution function selectors
* \#57096 \[SC-Medium] The implementation of TokeAutoEth::\_allocate is incorrect
* \#56839 \[SC-Medium] Moonwell Strategies Fail to Check Compound Error Codes Causing Silent Allocation Failures
* \#57311 \[SC-Medium] Moonwell allocation and deallocation can fail silently, causing incorrect state updates and loss of yield
* \#57331 \[SC-Medium] Conditional ETH Wrapping Logic Causes Withdrawal DoS in MoonwellWETH and StargateETH Strategies
* \#58741 \[SC-Medium] Action function signatures to 0x Settler are wrong
* \#58231 \[SC-Medium] Attacker can stop protocol from allocating assets to the AutoETH vaults
* \#57167 \[SC-Medium] Missing \`claim\` Function in Euler and Morpho strategies Leads to Loss of Yield Rewards
* \#58273 \[SC-Medium] Incorrect Hardcoded 0x Settler Function Selectors
* \#56859 \[SC-Medium] LP/underlying mismatch in \`StargateEthPoolStrategy::\_deallocate\` causes withdrawal DoS
* \#58336 \[SC-Medium] Additive Update to Survival Accumulator Causing Overflow
* \#58707 \[SC-Medium] Moonwell strategy \_allocate() does not revert when mint fails which can result in a sudden drop in Myt share price and consequently sever under-collateralization
* \#56873 \[SC-Medium] Incorrect ETH Wrapping Condition in MoonwellWETHStrategy.\_deallocate() Leads to Temporary Freezing of Funds
* \#56522 \[SC-Medium] \`TokeAutoUSDStrategy::\_allocate()\` and \`TokeAutoEthStrategy::\_allocate()\` may suffer a denial-of-service (DoS) due to token amount mismatch in \`AutopilotRouter::depositMax()\`
* \#58456 \[SC-Medium] Account Can Enter Unliquidatable State with Residual Debt
* \#58645 \[SC-Medium] Incorrect WETH Wrapping Amount in \`MoonwellWETHStrategy.\_deallocate()\` Wraps \`ethRedeemed\` Instead of \`amount\`
* \#57957 \[SC-Medium] Loss of EulerETH vault yields for Euler WETH Strategy
* \#57788 \[SC-Medium] Missing \`\_claimRewards()\` Implementation in AaveV3ARBUSDCStrategy Leads to Permanent Loss of Aave Incentive Rewards
* \#58203 \[SC-Medium] Moonwell Strategies Silent Failure Due to Unchecked mint() and redeemUnderlying() Return Values
* \#56855 \[SC-Medium] Liquidations Fail With Arithmetic Underflow When Forced Repayment Exhausts Collateral
* \#58093 \[SC-Medium] MORPHO reward in \`MorphoYearnOGWETH\` will be lost or stuck
* \#58006 \[SC-Medium] \`MoonwellUSDCStrategy.\_allocate\` ignores Compound-style mint failures and corrupts vault accounting
* \#58773 \[SC-Medium] In Stargate Incorrect Allocation Cap Accounting Leading to Unnecessary DoS
* \#58449 \[SC-Medium] TokeAutoEth Strategy Balance-Approval Mismatch DOS
* \#57867 \[SC-Medium] ZeroXSwapVerifier erroneously rejects Uniswap v3 swaps due to an an incorrect selector
* \#58605 \[SC-Medium] Missing \_claimRewards in AaveV3ARBUSDCStrategy leads to permanent freezing of accrued Aave incentives
* \#57752 \[SC-Medium] Aave and Euler incentives for MYT will be lost due to unimplemented \`\_claimRewards\` function
* \#58081 \[SC-Medium] Missing check in function \`AlchemistV3::setMinimumCollateralization\` could lead to set \`minimumCollateralization > globalMinimumCollateralization\`.
* \#58291 \[SC-Medium] Unlike setters collateralization functions, AlchemistV3::initialize doesnt enforce collateralization invariants allowing to break them
* \#58168 \[SC-Medium] Safe Position Liquidation Vulnerability in AlchemistV3 When minimumCollateralization Equals collateralizationLowerBound
* \#58616 \[SC-Medium] Liquidation can revert due to 0 amount fee withdraw
* \#56800 \[SC-Medium] Minimum collateral change lets liquidators seize compliant accounts
* \#57335 \[SC-Medium] Zero min-out ERC-4626 deposits cause under-mint and permanent allocation loss
* \#58728 \[SC-Medium] When the strategy is at a loss, the assets cannot be withdrawn
* \#57511 \[SC-Medium] Protocol could atleast be taking a part of the protocol fee
* \#58611 \[SC-Medium] Double-counting of earmarked debt repayments as cover leads to user funds being stuck and protocol insolvency.
* \#58515 \[SC-Medium] A liquidated position can end the liquidation process still below \`collateralizationLowerBound\`, allowing for double liquidation of positions.
* \#57272 \[SC-Medium] Silent Failures on Moonwell Deposit are not catched by strategy
* \#58787 \[SC-Medium] When allocation amount is greater than the maxDeposit of TokeAutoETh.sol, the remaining is stuck in TokeAutoEth.sol
* \#58087 \[SC-Medium] MoonwellUSDCStrategy ignores redeemUnderlying error codes → temporary freezing of funds (withdrawals revert)
* \#58730 \[SC-Medium] An attacker can prevent any TokenAuto strategy allocation by making a donation to the vault of as little as 1 wei of underlying token
* \#58181 \[SC-Medium] A griefer can cause a permanent DoS in TokeAutoETH/TokeAutoUSDCStrategy::allocate.
* \#56775 \[SC-Medium] Permanent freezing of funds from precision/dust + strict deallocation check
* \#58209 \[SC-Medium] Lack of Slippage Protection in Transmuter.claimRedemption and AlchemistV3.withdraw Leads to User Yield Losses
* \#56878 \[SC-Medium] The permissionedCalls check can be bypass
* \#57017 \[SC-Medium] \`AaveV3ARBWETHStrategy\`cant claim AAVE incentive
* \#57770 \[SC-Medium] Admin Can Bypass \`permissionedCalls\` Protection Using Multicall
* \#58751 \[SC-Medium] \`setMinimumCollateralization\` allows for increasing the current \`minimumCollateralization\` , instantly exposing users to risk of liquidation
* \#58185 \[SC-Medium] Incorrect \_survivalAccumulator accounting logic after \_earmarkWeight reaches 128 breaks core system invariants and can lead to protocol insolvency
* \#56706 \[SC-Medium] StargateEthPoolStrategy Incomplete ETH Wrapping Causes Withdrawal DoS
* \#57526 \[SC-Medium] \`StargateEthPoolStrategy\` rounding mismatch freezes \`VaultV2\` allocations
* \#57565 \[SC-Medium] The amount of dust will be permanently locked in \`StargateEthPoolStrategy\`
* \#58022 \[SC-Medium] Accounting Mismatch and Fund Stuck Due to Dust ETH on StargateEthPoolStrategy
* \#58427 \[SC-Medium] \`StargateEthPoolStrategy::\_allocate()\` and \`\_deallocate()\` Inconsistent Dust Handling Causes ETH to be Permanently Locked in Strategy Contract
* \#58105 \[SC-Medium] ZeroXSwapVerifier decodes execute payload with wrong ABI (bytes vs bytes\[]) → temporary freezing of funds
* \#57183 \[SC-Medium] Missing Incentive Rewards Claiming in Multiple Strategy Contracts
* \#57812 \[SC-Medium] No function to claim Aave Incentives
* \#58239 \[SC-Medium] Missing Aave incentives rewards claiming mechanism leads to permanent loss of protocol royalties
* \#58130 \[SC-Medium] Asymmetric Validation in Collateralization Setters Allows Protocol Misconfiguration Breaking All Borrowing
* \#58080 \[SC-Medium] Aave V3 strategies fail to claim OP/ARB liquidity mining rewards, causing permanent loss of yield
* \#56927 \[SC-Medium] setMinimumCollateralization function also needs a another check
* \#56982 \[SC-Medium] Incorrect function selectors used in ZeroXSwapVerifier
* \#58722 \[SC-Medium] TokenAuto strategy allocation uses maxDeposit which may allocate less than requested, leaving any excess funds permanently locked
* \#58115 \[SC-Medium] Incorrect WETH deposit amount prevents deposited ETH through \`receive\` function to cover strategy loss.
* \#58639 \[SC-Medium] \`\`Off by One\`\` issue in the \`\`\_forceRepay()\`\` function causes protocol to lose funds in the form of \`\`protocol fee\`\`.
* \#58403 \[SC-Medium] Missing Checks for Transaction Return Values in Moonwell Strategies
* \#57227 \[SC-Medium] Unchecked Return Codes in MoonwellUSDCStrategy Leading to Stuck Funds
* \#56960 \[SC-Medium] Missing Slippage Protection During Redemption Execution, lead to loss of token for user.
* \#58492 \[SC-Medium] Unbounded Deposit Exposure in TokeAutoEthStrategy::\_allocate()
* \#58313 \[SC-Medium] Incorrect allocation accounting and dust handling in \`StargateEthPoolStrategy\` causes systematic loss, cap mis-accounting, and deallocation reverts
* \#57646 \[SC-Medium] ABI Signature Mismatch in ZeroXSwapVerifier Causes Complete Failure to Verify Legitimate 0x Settler Transactions
* \#57483 \[SC-Medium] Fees could be skipped when there is not enough collateral
* \#57771 \[SC-Medium] Fee not collected in \_forceRepay when should
* \#58718 \[SC-Medium] In \_forceRepay Protocol Fee Collection Leads to Theft of Unclaimed Yield
* \#56846 \[SC-Medium] Liquidation will return because of insufficient funds
* \#56737 \[SC-Medium] The return value of mint is not checked
* \#58334 \[SC-Medium] Incorrect Function Selectors
* \#58033 \[SC-Medium] Unimplemented \`\_claimRewards()\` Function Results in Permanent Freezing of Aave Incentive Rewards
* \#57545 \[SC-Medium] Stargate ETH Strategy Rounding Bug

</details>

<details>

<summary>Low</summary>

* \#57964 \[SC-Low] Improper validation of absoluteCap and relativeCap enables excessive fund allocation in AlchemistAllocator.
* \#56893 \[SC-Low] Pending admin cannot accept ownership in AlchemistCurator
* \#58642 \[SC-Low] Cap Bypass in \`AlchemistAllocator.deallocate()\` Allows Over-Deallocation Beyond Computed Limits
* \#58051 \[SC-Low] Incorrect Access Control in acceptAdminOwnership()
* \#58666 \[SC-Low] Recipient/owner not enforced in action verifiers enables theft of swap proceeds
* \#56383 \[SC-Low] The \`AlchemistCurator::acceptAdminOwnership\` can't be called by the pending admin and if the function is called without pending admin the admin rigths will be lost
* \#57546 \[SC-Low] \`MoonwellUSDCStrategy\` fail to claim its reward from Moonwell Comptroller
* \#57473 \[SC-Low] Inverted Comparison Operator Allows Operators Admin-Level Allocation Privileges
* \#56947 \[SC-Low] Flawed Access Control in AlchemistCurator Admin Transfer Pattern Leads to Risk of Permanent Loss of Control
* \#57862 \[SC-Low] Incorrect \`balanceBefore\` reading order in \`MorphoYearnOGWETHStrategy::\_deallocate\` function leads to wrong event emission
* \#57983 \[SC-Low] Direct Asset Drain via \`ZeroXSwapVerifier\` Bypass and \`MYTStrategy\` Unlimited \`Permit2\` Approvals
* \#58512 \[SC-Low] MYTStrategy \`isValidSignature\` is implemented wrong and will not work
* \#56517 \[SC-Low] ZeroXSwapVerifier validates struct but executes external actions, enabling direct fund theft
* \#58462 \[SC-Low] Incorrect post-withdraw balance measurement causes false loss reporting and mis-accounting in \`MorphoYearnOGWETHStrategy.\_deallocate\`
* \#58424 \[SC-Low] MorphoYearnOGWETH Strategy - Balance Check Order Bug
* \#58672 \[SC-Low] Incorrect Balance Check Sequence
* \#56633 \[SC-Low] Access Control Flaw in acceptAdminOwnership() Prevents Secure Admin Transfer Leading to Potential Permanent Loss of Curator Control
* \#58190 \[SC-Low] Operator Has No Allocation Restrictions in \[\`AlchemistAllocator\`]\(<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#>...
* \#58310 \[SC-Low] Strategy \`FluidARBUSDCStrategy\` cant claim fluid token reward
* \#56882 \[SC-Low] Missing Cap Enforcement in AlchemistAllocator Allows Operators to Bypass Risk Controls
* \#58120 \[SC-Low] Incorrect balance measurement in MorphoYearnOGWETH strategy leads to incorrect deallocation loss registering
* \#58590 \[SC-Low] Incorrect balance-read ordering in MorphoYearnOGWETHStrategy.\_deallocate
* \#58579 \[SC-Low] Inconsistent Admin Management Implementation in AlchemistCurator.sol
* \#58749 \[SC-Low] Incorrect balance snapshot
* \#58078 \[SC-Low] Access Control Bypass in ZeroXSwapVerifier - Missing Owner Validation
* \#57625 \[SC-Low] Incorrect Cover Accounting in \_earmark Leads to Earmarking Failure and Value Leakage
* \#58400 \[SC-Low] Alchemist allocator does not actually enforce caps
* \#58607 \[SC-Low] Incorrect access control in admin ownership transfer allows only current admin to accept ownership instead of pending admin
* \#57599 \[SC-Low] Protocol wrongly withdraws before checking balance of withdraw
* \#57079 \[SC-Low] \[H-1] MorphoYearnOGWETH Strategy: Incorrect Balance Measurement Order in \_deallocate() Causes DoS on Withdrawals with Any Loss
* \#57514 \[SC-Low] Calldata verification bypass in 0x preflight logic enables arbitrary from/recipient manipulation and direct fund theft
* \#58648 \[SC-Low] Incorrect wethBalanceBefore read causes broken loss detection in deallocation
* \#57534 \[SC-Low] Small debt positions cannot be liquidated due to zero amount checks on token vaults
* \#58345 \[SC-Low] Operators in \`AlchemistAllocator.sol\` can allocate higher than DAO defined limits
* \#57621 \[SC-Low] Improper reward claiming in TokeAutoEthStrategy sends TOKE tokens to wrong address causing permanent freezing of unclaimed yield
* \#58709 \[SC-Low] Naive 0x fill parsing lets attackers spoof token and amount checks
* \#58362 \[SC-Low] Users will lose TokeMak rewards earned in TokeAutoEthStrategy
* \#57127 \[SC-Low] Pending admin should call the function instead of admin
* \#58480 \[SC-Low] Missing recipient and token binding in verifySwapCalldata leads to unauthorized fund transfers
* \#58516 \[SC-Low] Inverted Min/Max Logic in AlchemistAllocator Operator Cap Calculation
* \#58604 \[SC-Low] Verification bypass in \`\_verifyExecuteMetaTxnCalldata\` enables arbitrary 0x actions to pass checks and execute in the \`ZeroXSwapVerifier.sol\` contract
* \#56343 \[SC-Low] MorphoYearnOGWETH \_deallocate function always emits StrategyDeallocationLoss due to flawed balance measurement
* \#58497 \[SC-Low] The amount of WETH redeemed is not calculated properly in MorphoYearnOGWETH
* \#58329 \[SC-Low] Incorrect Balance Measurement in \`MorphoYearnOGWETH.\_deallocate()\` Leads to Temporary Freezing of Funds via Spurious Loss Events
* \#58778 \[SC-Low] ZeroXSwapVerifier implements incorrect data extraction logic enabling verification bypass in future strategy integrations
* \#58348 \[SC-Low] ZeroXSwapVerifier accepts malicious 0x calldata (recipient not bound, minOut ignored, transferFrom misused) -> attacker can route strategy/vault funds to self (Direct theft)
* \#58416 \[SC-Low] Unclaimed Extra Rewards in Tokemak Integration Lead to Permanent Freezing of Yield
* \#58422 \[SC-Low] MorphoYearn OG WETH strategy always emits deallocation-loss event due to zero delta calculation
* \#57746 \[SC-Low] Broken contract ownership logic at AlchemistV3.sol
* \#58143 \[SC-Low] Unused Cap Enforcement Variables (adjusted)
* \#56836 \[SC-Low] Ownership Transfer Failure in [\`AlchemistCurator\`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistCurator.sol) Prevents Future DAO Governance or Recovery
* \#57975 \[SC-Low] Broken admin rotation in acceptAdminOwnership() causes permanent governance lockout
* \#56625 \[SC-Low] Broken ownership transfer logic in AlchemistCurator permanently freezes contract operations
* \#57989 \[SC-Low] Broken isValidSignature leads to fund freezing
* \#58088 \[SC-Low] Inadequate enforcement of global cap enables cumulative over‑allocation
* \#58089 \[SC-Low] Arithmetic underflow revert in \`\_deallocate\`
* \#57837 \[SC-Low] \`MoonwellWETHStrategy\` cant claim reward from Moonwell Comptroller
* \#58002 \[SC-Low] Missing submitRemoveStrategy Function
* \#58149 \[SC-Low] MorphoYearnOGWETH incorrectly reports loss and triggers StrategyDeallocationLoss event
* \#57169 \[SC-Low] ZeroXSwapVerifier Policy Bypass via RFQ fillData Prefix (Token & Amount Spoof)
* \#58636 \[SC-Low] Broken Two-Step Admin Transfer Prevents Legitimate Admin Succession in AlchemistCurator
* \#58189 \[SC-Low] Two-step mechanism to transfer ownership is broken due to incorrect access control
* \#58259 \[SC-Low] Broken operator logic inside AlchemistCurator
* \#58386 \[SC-Low] Rewards claimed during deallocation remain stranded on strategy and unaccounted
* \#57665 \[SC-Low] Incorrect Balance Measurement in \`\_deallocate\` function of \`MorphoYearnOGWETHStrategy\`
* \#57697 \[SC-Low] Missing Recipient & \`from\` Checks in ZeroXSwapVerifier Enable Direct Asset Theft
* \#57866 \[SC-Low] Failure to verify the recipient's address can result in the theft of purchased tokens
* \#58578 \[SC-Low] ZeroXSwapVerifier allows attackers to drain strategy tokens via crafted calldata
* \#58575 \[SC-Low] Operator Limit Bypass
* \#58506 \[SC-Low] Adjusted Cap Limits Are Never Enforced
* \#58797 \[SC-Low] The \`TokeAuto\` strategies implementation does not accurately report the actual assets held by the strategy
* \#58796 \[SC-Low] Incorrect balance snapshot in \_deallocate() causes wethRedeemed always = 0
* \#57749 \[SC-Low] ZeroXSwapVerifier misses critical sender/recipient/minOut validations, allowing malicious 0x calldata to drain funds (Critical — Direct theft)
* \#57090 \[SC-Low] Ownership Transfer Failure in [\`AlchemistCurator\`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistCurator.sol) Prevents Future DAO Governance or Recovery
* \#56583 \[SC-Low] Wrong 2 step transferAdminOwnerShip logic and insufficient checks in AlchemistCurator.sol leads to permanent admin ownership loss
* \#58257 \[SC-Low] In TokeAutoETH deallocate can be DOSed if the vault incuring losses
* \#56830 \[SC-Low] Broken admin Ownership transfer Logic, acceptAdminOwnership() requires current admin instead of pending Admin, Blocking Role Claim.
* \#58513 \[SC-Low] Broken Access Control in AlchemistCurator.acceptAdminOwnership() Prevents Admin Transfer
* \#56911 \[SC-Low] Incorrectly implemented two-step admin ownership transfer mechanism prevents new admin to accept role
* \#57328 \[SC-Low] Once \`tokeLockDuration\` is the opposite of zero in \`TokeAutoEthStrategy\`, accumulated rewards in \`accToke\` can be stuck
* \#57316 \[SC-Low] Allocation Cap Enforcement Missing & DeadCode
* \#56582 \[SC-Low] AlchemistCurator::removeStrategy is unable to remove strategies from vaults due to wrong logic implementation
* \#56832 \[SC-Low] AlchemistCurator contract doesn't allow to remove strategies from the MYT morpho V2 vault.
* \#57806 \[SC-Low] Staking Graph argument bounds are incorrectly defined
* \#58393 \[SC-Low] Wrong order in balance querying instructions in MorphoYearnOGWETHStrategy::\_deallocate function leads to always emit StrategyDeallocationLoss event
* \#58352 \[SC-Low] Assets Become Permanently Stuck in TokeAutoEth Strategy Due to Strict Balance Check
* \#58542 \[SC-Low] \[Low] Logic Error in MorphoYearnOGWETHStrategy.\_deallocate(): \`wethRedeemed\` Always Zero → All Deallocations Emit \`StrategyDeallocationLoss\`
* \#58357 \[SC-Low] Permanent Freezing of TokeAutoEth strategy rewards in MYT Vault
* \#57057 \[SC-Low] Wrong order of balance checks in MorphoYearnOGWETHStrategy
* \#57251 \[SC-Low] Curator Cannot Remove Adapter Due to Timelock Requirement
* \#56887 \[SC-Low] Incorrect balance tracking in MorphoYearnOGWETHStrategy \_deallocate function leads to wrong loss event emission(Resend))
* \#56324 \[SC-Low] Missing \`\`\`from==owner \`\`\`check in transferFrom verifier → direct theft of user funds
* \#58743 \[SC-Low] ZeroXSwapVerifier Recipient Validation Bypass
* \#57114 \[SC-Low] Inherited \`setAdmin\` function allows to bypass two-step admin ownership transfer mechanism
* \#58198 \[SC-Low] Broken Two-Step Admin Transfer Pattern
* \#56418 \[SC-Low] Two step owner transfer is broken and can lead to unforseen damages
* \#57622 \[SC-Low] Lack of claimed reward handling in MYT strategies will keep all external token rewards stuck forever
* \#56465 \[SC-Low] \`getTotalDeposited\` doesn't reflect the correct total deposited
* \#57439 \[SC-Low] Incorrect \`badDebtRatio\` rounding in \`Transmuter::claimRedemption()\` may cause funds to become permanently stuck
* \#58210 \[SC-Low] Incorrect balance measurement in deallocation disables loss detection in MorphoYearnOGWETH.
* \#58423 \[SC-Low] Pending Admin Cannot Accept Ownership Transfer in \`AlchemistCurator\`
* \#58249 \[SC-Low] Broken Two-Step Admin Handover in AlchemistCurator
* \#57394 \[SC-Low] \`acceptAdminOwnership()\` only allows the current admin to finalise transfers
* \#58124 \[SC-Low] Direct Theft of Funds via Malicious actions\[] in execute() Call Due to Incorrect Calldata Verification
* \#58627 \[SC-Low] Incorrect delta calculation in \_deallocate() causes wethRedeemed to always be zero.
* \#56498 \[SC-Low] Reserve Drainage Due to Incorrect Balance Measurement
* \#58322 \[SC-Low] Incorrect Emit Due to Wrong Ordering of wethBalanceBefore Calculation
* \#58360 \[SC-Low] Round-down calculation in \`convertToShares()\` leads to deallocation failure in TokeAutoEth strategy
* \#58408 \[SC-Low] Underflow account.rawLocked on \_subDebt due to rounding inconsistency
* \#58410 \[SC-Low] Tokemak Strategy Deallocation Causes TOKE Token Lockup
* \#58419 \[SC-Low] AlchemistCurator two-step ownership transfer mis-implemented
* \#58469 \[SC-Low] Pending Admin Cannot Accept Ownership
* \#56689 \[SC-Low] Reward token TOKE is stuck in MYT
* \#58555 \[SC-Low] \`AlchemistCurator\` 2-step ownership transfer is implemented incorrectly
* \#58428 \[SC-Low] TOKE reward loss when calling deallocate
* \#58376 \[SC-Low] claimRewards() function permanently locks earned Toke reward token on Morpho VaultV2
* \#58527 \[SC-Low] Complete loss of all reward value on TokeAutoEthStrategy \_claimRewards
* \#56451 \[SC-Low] \`AlchemistAllocator::allocate()\` and \`deallocate()\` do not enforce cap checks as intended
* \#57644 \[SC-Low] Unenforced cap logic in AlchemistAllocator allows not controlled allocations
* \#58325 \[SC-Low] Operator Can Shift vault Funds to Risky Strategies Without Oversight, leading to potential loss of user funds.
* \#58714 \[SC-Low] Pending Admin cannot accept Ownership in \`AlchemistCurator.sol\`
* \#58734 \[SC-Low] Broken strategy realAssets calculation
* \#56983 \[SC-Low] Tokemak rewards sent to MYT vault contract (not strategy) -> rewards stranded
* \#58056 \[SC-Low] The Auto ETH and USDC staking rewards will stuck in vault
* \#57067 \[SC-Low] Overstated Per-Account Locked Collateral Due to Global Clamp in \_subDebt
* \#56332 \[SC-Low] Pending Admin Cannot Accept Ownership
* \#57479 \[SC-Low] Logical bug in \`AlchemistCurator::acceptAdminOwnership\`, asking to current admin to accept ownership.
* \#58007 \[SC-Low] \`pendingAdmin\` cannot call \`acceptAdminOwnership()\` to accept admin role
* \#58333 \[SC-Low] Incorrect onlyAdmin Modifier in acceptAdminOwnership
* \#57024 \[SC-Low] \`wethBalanceBefore\` is computed after withdrawal in \`\_deallocate\` function in MorphoYearnOGWETHStrategy contract, leading to systematic StrategyDeallocationLoss event emission.
* \#56909 \[SC-Low] Incorrect balance snapshot in strategy deallocation causes false loss events and masks real shortfalls
* \#56529 \[SC-Low] Incorrect token balance calculation in MorphoYearnOGWETHStrategy.sol::\_deallocate() leads to wrong event emitted every time
* \#57926 \[SC-Low] The conditional 'StrategyDeallocationLoss' event in \`MorphoYearnOGWETHStrategy::\_deallocate\` gets logged all the time due a misplacement in variable declaration
* \#56961 \[SC-Low] Incorrect balance snapshot check in \`\_deallocate()\` logs false deallocation loss in MorphoYearnOGWETH strategy
* \#56962 \[SC-Low] Balance Check Logic Error in \_deallocate() Function Leads to Broken Loss Detection and False Event Emissions
* \#57777 \[SC-Low] ZeroX swap verifier bypass enables direct theft of user funds
* \#58079 \[SC-Low] Missing from validation in ZeroXSwapVerifier.verifySwapCalldata() enables direct theft of approved funds
* \#58289 \[SC-Low] Missing addresses Verification in ZeroXSwapVerifier
* \#57516 \[SC-Low] Arbitrary External Call in ZeroXSwapVerifier Leads to Theft of Unclaimed Yield
* \#58418 \[SC-Low] \`verifySwapCalldata\` cant verify the output token of the swap
* \#58705 \[SC-Low] Mismatch between emitted protocol fee and actual fee paid in \_forceRepay due to strict inequality check
* \#58040 \[SC-Low] removeStrategy() is Non-Functional
* \#57346 \[SC-Low] AlchemistAllocator Compares Incompatible Units (Asset Wei vs WAD Percentage)
* \#57982 \[SC-Low] Permanently stuck rewards in the Vault
* \#58473 \[SC-Low] Wrong redeemed amount calculation in MorphoYearnOGWETH strategy
* \#58110 \[SC-Low] MorphoYearnOGWETHStrategy will always report strategy loss
* \#57637 \[SC-Low] acceptAdminOwnership doesn't allow expected user approval
* \#56709 \[SC-Low] ZeroXSwapVerifier Missing Source Validation
* \#58133 \[SC-Low] TOKE Rewards Permanently Locked in Strategy adapter
* \#58488 \[SC-Low] TokeAutoUSDStrategy claims rewards to itself automatically when deallocate is called but since reward token is Tokemak the rewards remain permanently locked
* \#58520 \[SC-Low] Pending admin cannot accept ownership
* \#57123 \[SC-Low] Incorrect 2 step ownership in AlchemistCurator
* \#56602 \[SC-Low] Function takes incorrect modifier
* \#58077 \[SC-Low] Reward tokens are incorrectly claimed to strategy contract during deallocation leads to permanent token loss
* \#58244 \[SC-Low] Incorrect balance check order in \`MorphoYearnOGWETH\` strategy leads to false deallocation loss events

</details>

<details>

<summary>Insight</summary>

* \#56494 \[SC-Insight] Gas Optimization: Redundant External Calls in Strategy \_deallocate Functions
* \#58667 \[SC-Insight] Permit2 is approved the wrong asset which leads to loss of funds or failing swaps
* \#58719 \[SC-Insight] \[INSIGHT] Gas Optimization: Save gas by using the cached fee amount in burn() and repay() in \`Alchemist.sol\`
* \#56336 \[SC-Insight] \`StargateEthPoolStrategy::\_deallocate\` would emit false deallocating loss event in some cases
* \#58739 \[SC-Insight] Decimals mismatch causes 1e12 under-reporting in strategy returns, letting allocations silently exceed per-strategy and global caps
* \#56346 \[SC-Insight] Redundant calculation of feeAmount in repay function
* \#57448 \[SC-Insight] Unnecessary computation of lockedCollateral in \`\_addDebt()\` and \`\_subDebt()\`
* \#58094 \[SC-Insight] AutopoolETH vault slippage during LP token liquidation leads to temporary fund freezing
* \#57522 \[SC-Insight] useCurrent flag ignored in preview functions in Moonwell strategies
* \#58326 \[SC-Insight] The value of the burned Peapods share token may exceed expectations
* \#56368 \[SC-Insight] \`AlchemistTokenVault::deposit()\` should use \`safeTransferFrom()\` instead of \`transferFrom()\`; \`AlchemistTokenVault::withdraw()\` should use \`safeTransfer()\` instead of \`transfer()\`
* \#56621 \[SC-Insight] Broken withdrawal logic in AaveV3ARBUSDCStrategy permanently locks user funds
* \#56806 \[SC-Insight] Broken withdrawal logic in AaveV3ARBWETHStrategy permanently locks user funds
* \#57969 \[SC-Insight] Lack of incentive to liquidate small positions can cause the system to accumulate bad debt
* \#56348 \[SC-Insight] Incorrect APY calculation in MYTStrategy::\_approxAPY() causes underreported yields
* \#58762 \[SC-Insight] Manipulation of \`feeInUnderlying\` through front-running during liquidations on Ethereum
* \#56658 \[SC-Insight] Transmuter's tokenURI does not revert for nonexistent tokenIds
* \#57816 \[SC-Insight] Critical Incentive Failure in calculateLiquidation Leads to Protocol Insolvency Risk During Global Bad Debt
* \#56528 \[SC-Insight] Unbounded \`slippageBPS\` Can Freeze Withdrawals
* \#56326 \[SC-Insight] Variable could be immutable
* \#56350 \[SC-Insight] Implementation contract AlchemistV3 not locked (\_disableInitializers() missing)
* \#56462 \[SC-Insight] Unused Mapping Causes Unnecessary Storage Gas Consumption
* \#58076 \[SC-Insight] Fix unit mismatch in \_doLiquidation: collateralInUnderlying -> collateralInDebt
* \#58735 \[SC-Insight] \`\`calculateLiquidation\`\` reverts due to divide by \`\`ZERO\`\` if \`\`targetCollateralization = FIXED\_POINT\_SCALAR\`\`
* \#57563 \[SC-Insight] Reward tokens being permanently frozen in TokeAutoUSDStrategy
* \#56328 \[SC-Insight] Redundant require statement in EulerUSDCStrategy \_deallocate function leads to unnecessary gas consumption
* \#56801 \[SC-Insight] Function burn could be gas optimized
* \#56406 \[SC-Insight] \`getEstimatedYield\` never updates after snapshots
* \#56730 \[SC-Insight] Transmuter \`tokenURI()\` is not EIP-721 compliance
* \#58552 \[SC-Insight] Single transfer instead of multiple saves gas
* \#57791 \[SC-Insight] Receipt Token Misconfiguration in Aave Strategies
* \#56572 \[SC-Insight] Aave V3 lending pool is immutable in Aave strategies
* \#56949 \[SC-Insight] Uncapped collateral transfer in redemption leads to accounting discrepancy enabling theft of user funds
* \#58703 \[SC-Insight] Cached interest rate calculation in PeapodsETH strategy leads to inaccurate APR/APY estimates
* \#58146 \[SC-Insight] Whitelist can be disabled repeatedly, contradicting intended program behavior.
* \#56895 \[SC-Insight] Function approveMint is vulnerable to race conditions
* \#57291 \[SC-Insight] Hardcoded Slippage in MYT Strategy
* \#56561 \[SC-Insight] Fee amount is recomputed multiple times when the initial value has already been cached
* \#57923 \[SC-Insight] Redundant Synthetic Transfers in claimRedemption When amountNottransmuted is Zero
* \#56427 \[SC-Insight] src/utils/PermissionedProxy.sol::setPermissionedCall incomplete event emission because it doesnt include value argument for signature
* \#56347 \[SC-Insight] burn contains redundant calculations
* \#56518 \[SC-Insight] \`claimWithdrawalQueue\` discards claimed amount
* \#58356 \[SC-Insight] The Alchemist TokeAuto Strategies doesn't use recommended best practice by TokeAuto.
* \#57606 \[SC-Insight] Attacker can DoS deposits by hitting the deposit cap
* \#57028 \[SC-Insight] Wrong amount variable in Repay event
* \#57832 \[SC-Insight] Cap Logic Error in AlchemistAllocator
* \#57208 \[SC-Insight] It is possible to prevent lowering the deposit cap by front-running

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#58755 \[SC-High] Users position that are synced at certain times overestimate collateralBalance of the position
* \#57964 \[SC-Low] Improper validation of absoluteCap and relativeCap enables excessive fund allocation in AlchemistAllocator.
* \#56494 \[SC-Insight] Gas Optimization: Redundant External Calls in Strategy \_deallocate Functions
* \#56893 \[SC-Low] Pending admin cannot accept ownership in AlchemistCurator
* \#58642 \[SC-Low] Cap Bypass in \`AlchemistAllocator.deallocate()\` Allows Over-Deallocation Beyond Computed Limits
* \#58051 \[SC-Low] Incorrect Access Control in acceptAdminOwnership()
* \#58666 \[SC-Low] Recipient/owner not enforced in action verifiers enables theft of swap proceeds
* \#58667 \[SC-Insight] Permit2 is approved the wrong asset which leads to loss of funds or failing swaps
* \#56383 \[SC-Low] The \`AlchemistCurator::acceptAdminOwnership\` can't be called by the pending admin and if the function is called without pending admin the admin rigths will be lost
* \#57774 \[SC-Critical] Redemption Earmark Mechanism Can Be Permanently Blocked via Single-Block Earmark Calls
* \#58793 \[SC-Critical] Repayment Fee Overpayment from Global Collateral Pool
* \#57093 \[SC-Critical] Potential Locked Funds Due to Partial Redeem Shortfall and miss calculation, lead to user loss their myt token forever.
* \#57546 \[SC-Low] \`MoonwellUSDCStrategy\` fail to claim its reward from Moonwell Comptroller
* \#57473 \[SC-Low] Inverted Comparison Operator Allows Operators Admin-Level Allocation Privileges
* \#56571 \[SC-High] Inflated claim payouts from double-counted MYT after liquidation
* \#56947 \[SC-Low] Flawed Access Control in AlchemistCurator Admin Transfer Pattern Leads to Risk of Permanent Loss of Control
* \#57862 \[SC-Low] Incorrect \`balanceBefore\` reading order in \`MorphoYearnOGWETHStrategy::\_deallocate\` function leads to wrong event emission
* \#57983 \[SC-Low] Direct Asset Drain via \`ZeroXSwapVerifier\` Bypass and \`MYTStrategy\` Unlimited \`Permit2\` Approvals
* \#56751 \[SC-Medium] StargateEthPoolStrategy deallocate function redeem less weth than expected
* \#57036 \[SC-High] Unconditional Debt Reduction Before Protocol Fee Check in Force Repayment
* \#57041 \[SC-High] Deallocation Accounting Mismatch Between Vault and Adapter
* \#58512 \[SC-Low] MYTStrategy \`isValidSignature\` is implemented wrong and will not work
* \#58004 \[SC-High] Protocol Insolvency from \`cumulativeEarmarked\` During \`\_forceRepay()\`
* \#58276 \[SC-Critical] Uncapped \`feeInYield\` in \`\_resolveRepaymentFee\` allows for collateral theft from other depositors
* \#56517 \[SC-Low] ZeroXSwapVerifier validates struct but executes external actions, enabling direct fund theft
* \#58462 \[SC-Low] Incorrect post-withdraw balance measurement causes false loss reporting and mis-accounting in \`MorphoYearnOGWETHStrategy.\_deallocate\`
* \#58526 \[SC-High] Missing accounting update in liquidation functions leads to permanent DOS on deposits
* \#58466 \[SC-High] Liquidation Fee Payment Failure Due to Redundant wrong Collateral Check
* \#58518 \[SC-Critical] Liquidation will steal Repayment Fee from Innocent Users Funds
* \#56692 \[SC-Medium] \`ZeroXSwapVerifier\` verification will always revert due to wrong hardcoded execution function selectors
* \#58471 \[SC-High] Accounting error in \`\_forceRepay\`/\`\_doLiquidation\` overstates TVL, enabling under-scaled redemptions and potential insolvency
* \#58474 \[SC-High] Liquidator will bypass liquidation fees affecting protocol revenue
* \#57096 \[SC-Medium] The implementation of TokeAutoEth::\_allocate is incorrect
* \#58424 \[SC-Low] MorphoYearnOGWETH Strategy - Balance Check Order Bug
* \#56839 \[SC-Medium] Moonwell Strategies Fail to Check Compound Error Codes Causing Silent Allocation Failures
* \#56827 \[SC-High] Missing Global Earmark Reduction in \_forceRepay
* \#57122 \[SC-Critical] Mismatch Between Capped Fee and Returned Fee in \`\_resolveRepaymentFee\`
* \#58672 \[SC-Low] Incorrect Balance Check Sequence
* \#57633 \[SC-High] Block-gated \_earmark() call in redeem() nullifies prefunded Transmuter cover on the first redemption of each block, leading to collateral overpayment and potential protocol insolvency
* \#57288 \[SC-High] Flawed rounding logic in TokeAutoEth deallocate function causes permanent freezing of funds
* \#57172 \[SC-High] Missing \_mytSharesDeposited Decrements in Liquidation Flows Causes Accounting Divergence
* \#58450 \[SC-High] Missing Transmuter Balance Update After Redemption Blocks Future Earmarking and Underfunds Redemptions
* \#57308 \[SC-High] AlchemistV3 does not update \`\_mytSharesDeposited\` when performing liquidation, causing global accounting and liquidation logic mismatch
* \#57311 \[SC-Medium] Moonwell allocation and deallocation can fail silently, causing incorrect state updates and loss of yield
* \#56402 \[SC-High] \`killSwitch\` leaves vault assets stranded and blocks withdrawals
* \#58531 \[SC-Critical] QueryGraph function Zero-Return Bug causing Tracking Earmarking Failure Over Progressive Block Intervals
* \#57331 \[SC-Medium] Conditional ETH Wrapping Logic Causes Withdrawal DoS in MoonwellWETH and StargateETH Strategies
* \#58113 \[SC-High] StargateEthPoolStrategy.realAssets return false real assets
* \#58288 \[SC-Critical] Incorrect Fee Payment Logic Leads to Underpayment
* \#58306 \[SC-Critical] Repayment Fee Not Adjusted for Insufficient Collateral
* \#58394 \[SC-High] MEV opportunity because no slippage protection in TokeAutoEthStrategy
* \#56633 \[SC-Low] Access Control Flaw in acceptAdminOwnership() Prevents Secure Admin Transfer Leading to Potential Permanent Loss of Curator Control
* \#58190 \[SC-Low] Operator Has No Allocation Restrictions in \[\`AlchemistAllocator\`]\(<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#>...
* \#58310 \[SC-Low] Strategy \`FluidARBUSDCStrategy\` cant claim fluid token reward
* \#57510 \[SC-High] Stale Locked Collateral Tracking During Price Appreciation Causes Disproportionate Redemption Losses
* \#57345 \[SC-High] Missing cumulativeEarmarked Decrement in \_forceRepay Breaks Earmarking Invariant Leading to Unfair Redemption Burden Distribution
* \#58547 \[SC-High] Mismatched Accounting and Transfer for Capped Fees
* \#58337 \[SC-High] Incorrect Handling of cumulativeEarmarked in \_forceRepay leads to inflated survival accumulator.
* \#57745 \[SC-High] Syn fails to update the rawLocked valuation leading to a loss of fund for users with rawlock > 0 when total lock become 0.
* \#58572 \[SC-High] Liquidation of account \`\`collateral\`\` doesn't subtract \`\`\_mytSharesDeposited\`\` which creates bad debt in the system and causes \`\`insolvency\`\`.
* \#57360 \[SC-Critical] Unreconciled repayment fee transfer enables MYT overpayment and TVL inflation
* \#56882 \[SC-Low] Missing Cap Enforcement in AlchemistAllocator Allows Operators to Bypass Risk Controls
* \#58409 \[SC-High] ## \[HIGH] Arithmetic Underflow in \`MYTStrategy.sol\`'s \`deallocate()\` Check Prevents Yield Withdrawal
* \#56435 \[SC-Critical] AlchemistV3: repayment‑only liquidation pays liquidator from pool (fee leak) → theft of unclaimed yield
* \#57954 \[SC-High] Lackf of tracking of excess cover in \`\_earmark\` function leads to permanent loss of cover value and stuck user positions.
* \#57460 \[SC-High] Protocol fails to subtract fee from total locked when burning and repaying
* \#58688 \[SC-Critical] \`AlchemistV3::\_liquidate\` can steal other users’ collateral
* \#58320 \[SC-Critical] Incorrect Fee Return Value in \_resolveRepaymentFee Enables Fund Theft Under Extreme Conditions
* \#58425 \[SC-High] Missing slippage protection when depositing to TokeAuto strategies
* \#58120 \[SC-Low] Incorrect balance measurement in MorphoYearnOGWETH strategy leads to incorrect deallocation loss registering
* \#58741 \[SC-Medium] Action function signatures to 0x Settler are wrong
* \#58231 \[SC-Medium] Attacker can stop protocol from allocating assets to the AutoETH vaults
* \#58683 \[SC-Critical] There is an issue in earmarked debt eeduction in the repay() can causes a permanent fund freeze
* \#58447 \[SC-Critical] Unfair Collateral Loss Through Socialized Redemption Costs
* \#57559 \[SC-High] Missing \`\_mytSharesDeposited\` Decrement in Liquidation Paths Enables Theft of Unclaimed Yield and Protocol Insolvency
* \#58387 \[SC-High] Liquidator Fee in the \_doliquidation Function Withheld When Collateral Is Exhausted Leading to Seized Fee Trapped in Protocol
* \#57447 \[SC-High] Untracked MYT outflows inflate TVL causing liquidation suppression
* \#58590 \[SC-Low] Incorrect balance-read ordering in MorphoYearnOGWETHStrategy.\_deallocate
* \#58579 \[SC-Low] Inconsistent Admin Management Implementation in AlchemistCurator.sol
* \#58749 \[SC-Low] Incorrect balance snapshot
* \#57167 \[SC-Medium] Missing \`claim\` Function in Euler and Morpho strategies Leads to Loss of Yield Rewards
* \#57148 \[SC-High] \`\_mytSharesDeposited\` variable is not correctly updated during liquidations, leading to wrong assumptions and incorrect bad debt calculation in the Transmuter.
* \#58763 \[SC-High] Accounting is broken when redeem() is bypassed due to Transmuter balance
* \#58273 \[SC-Medium] Incorrect Hardcoded 0x Settler Function Selectors
* \#58573 \[SC-Critical] AlchemistV3 Repayment Fee Cross-Account Theft Vulnerability
* \#58398 \[SC-High] No Slippage Protection on Large Allocation Deposits
* \#57751 \[SC-High] There is a problem related to forced liquidation branch and this creates issue thatk cna drains protocol backing
* \#58413 \[SC-Critical] Attacker/user can prevent Earmark from updating the earnmarkweight causing the transmuter action to repay det gradually to fail for all users
* \#56727 \[SC-High] Underlying increase in forced repayments leads to insolvency
* \#58078 \[SC-Low] Access Control Bypass in ZeroXSwapVerifier - Missing Owner Validation
* \#57625 \[SC-Low] Incorrect Cover Accounting in \_earmark Leads to Earmarking Failure and Value Leakage
* \#56673 \[SC-High] Zero-cost fee farming via forced earmarked repayment
* \#58400 \[SC-Low] Alchemist allocator does not actually enforce caps
* \#58607 \[SC-Low] Incorrect access control in admin ownership transfer allows only current admin to accept ownership instead of pending admin
* \#56732 \[SC-Critical] Incorrect boundary condition in queryGraph leads to systematic under-earmarking and transmuter redemption fund loss
* \#56385 \[SC-Critical] Repayment fee can be paid from the pool even when the account has no collateral left
* \#57599 \[SC-Low] Protocol wrongly withdraws before checking balance of withdraw
* \#58736 \[SC-High] Missing TVL Accounting in \_forceRepay and \_doLiquidation Leads to Protocol Insolvency
* \#58274 \[SC-High] Liquidation fee logic in \`\_doLiquidation\` strands liquidator rewards when balance is exhausted, freezing funds
* \#56714 \[SC-High] Accounting Invariant Violation in \_forceRepay Leads to Protocol Insolvency
* \#58719 \[SC-Insight] \[INSIGHT] Gas Optimization: Save gas by using the cached fee amount in burn() and repay() in \`Alchemist.sol\`
* \#57730 \[SC-High] Liquidation Does Not Decrease mytSharesDeposited
* \#56859 \[SC-Medium] LP/underlying mismatch in \`StargateEthPoolStrategy::\_deallocate\` causes withdrawal DoS
* \#57079 \[SC-Low] \[H-1] MorphoYearnOGWETH Strategy: Incorrect Balance Measurement Order in \_deallocate() Causes DoS on Withdrawals with Any Loss
* \#57590 \[SC-Critical] Double-counted Transmuter cover in \`redeem()\` allows overstated redemptions and potential over-withdraw/over-borrow
* \#57514 \[SC-Low] Calldata verification bypass in 0x preflight logic enables arbitrary from/recipient manipulation and direct fund theft
* \#56363 \[SC-High] \`\_mytSharesDeposited\` not correctly updated in all cases, leading to incorrect protocol collateralization and reduced liquidation incentives
* \#56395 \[SC-High] Accounting desync in liquidation outflows leads to artificial deposit cap exhaustion and denial-of-Service on recapitalization
* \#58236 \[SC-High] Accounting mismatch: \`\_forceRepay\`/\`\_doLiquidation\` fail to decrement \`\_mytSharesDeposited\`, locking deposit capacity and overstating collateral
* \#58724 \[SC-Critical] Partial Redemption Burns Full Position — Accounting Desynchronization and Potential Underpayment in Transmuter.claimRedemption()
* \#58098 \[SC-High] There is a problem from ledger TVL sesync inliquidations cause a under-liquidation and systemic insolvency risk
* \#56824 \[SC-High] Missing update to \_mytSharesDeposited during liquidation
* \#58648 \[SC-Low] Incorrect wethBalanceBefore read causes broken loss detection in deallocation
* \#58336 \[SC-Medium] Additive Update to Survival Accumulator Causing Overflow
* \#58354 \[SC-High] \`\_forceRepay\` does not decrement \`\_mytSharesDeposited\`, causing a temporal blocking of new deposits
* \#58771 \[SC-High] Incorrect Tracking of Total Deposited Yield Tokens (\_mytSharesDeposited) in Liquidation and Force Repayment Paths
* \#56552 \[SC-High] Liquidation fee misrouting in AlchemistV3.\_doLiquidation() leads to theft of unclaimed yield (liquidator fee stranded)
* \#57530 \[SC-High] Stale TVL Accounting in Liquidations Leads to Protocol Insolvency
* \#56336 \[SC-Insight] \`StargateEthPoolStrategy::\_deallocate\` would emit false deallocating loss event in some cases
* \#58707 \[SC-Medium] Moonwell strategy \_allocate() does not revert when mint fails which can result in a sudden drop in Myt share price and consequently sever under-collateralization
* \#57534 \[SC-Low] Small debt positions cannot be liquidated due to zero amount checks on token vaults
* \#58739 \[SC-Insight] Decimals mismatch causes 1e12 under-reporting in strategy returns, letting allocations silently exceed per-strategy and global caps
* \#58345 \[SC-Low] Operators in \`AlchemistAllocator.sol\` can allocate higher than DAO defined limits
* \#56794 \[SC-Critical] Liquidators can be overpaid due to accounting error
* \#57668 \[SC-High] Missing collateral tracking update during liquidation leads to inflated total value calculation and delayed under-collateralization protection
* \#57621 \[SC-Low] Improper reward claiming in TokeAutoEthStrategy sends TOKE tokens to wrong address causing permanent freezing of unclaimed yield
* \#58709 \[SC-Low] Naive 0x fill parsing lets attackers spoof token and amount checks
* \#56873 \[SC-Medium] Incorrect ETH Wrapping Condition in MoonwellWETHStrategy.\_deallocate() Leads to Temporary Freezing of Funds
* \#56346 \[SC-Insight] Redundant calculation of feeAmount in repay function
* \#58362 \[SC-Low] Users will lose TokeMak rewards earned in TokeAutoEthStrategy
* \#58192 \[SC-High] TokeAutoEth Strategy Tokens Locked When AutoPool(router) Enforces maxDeposit Cap
* \#57127 \[SC-Low] Pending admin should call the function instead of admin
* \#57129 \[SC-High] Missing \_mytSharesDeposited Decrement in Liquidation Functions Causes Permanent TVL Inflation
* \#58507 \[SC-Critical] Repayment fee after forceRepay could result in socialized loss during global undercollateralization
* \#56965 \[SC-Critical] AlchemistV3 handling of added Transmuter coverage includes an error that enables an attacker to cause protocol insolvency
* \#58480 \[SC-Low] Missing recipient and token binding in verifySwapCalldata leads to unauthorized fund transfers
* \#58516 \[SC-Low] Inverted Min/Max Logic in AlchemistAllocator Operator Cap Calculation
* \#58358 \[SC-High] Mismatched CollateralWeight and rawLocked Causes Incorrect Collateral Removal in Sync
* \#56522 \[SC-Medium] \`TokeAutoUSDStrategy::\_allocate()\` and \`TokeAutoEthStrategy::\_allocate()\` may suffer a denial-of-service (DoS) due to token amount mismatch in \`AutopilotRouter::depositMax()\`
* \#58604 \[SC-Low] Verification bypass in \`\_verifyExecuteMetaTxnCalldata\` enables arbitrary 0x actions to pass checks and execute in the \`ZeroXSwapVerifier.sol\` contract
* \#57941 \[SC-High] Incorrect handling of deallocate return val causes any interest gains in a strategy to become unclaimable and permanently locked
* \#56519 \[SC-Critical] Unchecked repayment fee transfer in \`\_liquidate\` pays liquidators from other users’ collateral
* \#58280 \[SC-Critical] Repayment's fee is charged from other users causing the contract to fail when the myt total balance of a user cannot cover the fee
* \#57053 \[SC-Critical] Integer Division Precision Loss in normalizeDebtTokensToUnderlying Leads to Permanent Collateral Locking
* \#57448 \[SC-Insight] Unnecessary computation of lockedCollateral in \`\_addDebt()\` and \`\_subDebt()\`
* \#56343 \[SC-Low] MorphoYearnOGWETH \_deallocate function always emits StrategyDeallocationLoss due to flawed balance measurement
* \#58497 \[SC-Low] The amount of WETH redeemed is not calculated properly in MorphoYearnOGWETH
* \#58269 \[SC-High] Liquidator Fee Not Paid When Fee Equals Surplus
* \#58329 \[SC-Low] Incorrect Balance Measurement in \`MorphoYearnOGWETH.\_deallocate()\` Leads to Temporary Freezing of Funds via Spurious Loss Events
* \#57662 \[SC-Critical] portion of users alAsset amount that staked in transmuter can be lost forever when \`amount > cumulativeEarmarked\`
* \#58626 \[SC-Critical] Repayment Fee Overpayment in Liquidation Repay-Only Path
* \#58094 \[SC-Insight] AutopoolETH vault slippage during LP token liquidation leads to temporary fund freezing
* \#56672 \[SC-High] Inconsistent MYT share accounting leads to under-liquidation and solvency risk
* \#57907 \[SC-High] Incorrect forced-repayment accounting allows debt forgiveness and frees locked collateral (systemic loss)
* \#57585 \[SC-High] AlchemistV3 does not properly update CDP collateralBalance when redemptions exceed \_totalLocked which enables some CDPs to over-withdraw collateral on account of others
* \#58778 \[SC-Low] ZeroXSwapVerifier implements incorrect data extraction logic enabling verification bypass in future strategy integrations
* \#58347 \[SC-High] Accounting Drift Due to Missing \`\_mytSharesDeposited\` Decrements During Liquidation
* \#58348 \[SC-Low] ZeroXSwapVerifier accepts malicious 0x calldata (recipient not bound, minOut ignored, transferFrom misused) -> attacker can route strategy/vault funds to self (Direct theft)
* \#58416 \[SC-Low] Unclaimed Extra Rewards in Tokemak Integration Lead to Permanent Freezing of Yield
* \#58422 \[SC-Low] MorphoYearn OG WETH strategy always emits deallocation-loss event due to zero delta calculation
* \#58456 \[SC-Medium] Account Can Enter Unliquidatable State with Residual Debt
* \#57930 \[SC-High] Allocation tracking underflow in strategy deallocation Leads to protocol insolvency
* \#56817 \[SC-High] ForceRepay doesn't decrement \_mytSharesDeposited, inflating TVL
* \#58112 \[SC-High] A malicious user can avoid getting penalized upon a Transmuter redemption by depositing and withdrawing collateral in the Alchemist
* \#58628 \[SC-High] Attackers Can Avoid Redemption Losses By Temporarily Burning and Re-Borrowing The Debt
* \#57522 \[SC-Insight] useCurrent flag ignored in preview functions in Moonwell strategies
* \#58326 \[SC-Insight] The value of the burned Peapods share token may exceed expectations
* \#57088 \[SC-High] Unscaled collateral accounting in redeem lets users withdraw more than intended
* \#57726 \[SC-High] AlchemistV3: MYT TVL accounting drift on liquidation/forceRepay blocks deposits via depositCap (Medium — Smart contract unable to operate due to lack of token funds)
* \#58396 \[SC-High] Total locked is not cleared proportionally to the total debt, this forces the collateral weight to become incorrect and new users transmuter redeem repayment will repay more debt fo...
* \#58163 \[SC-Critical] Total loss of user Funds in claim redemption
* \#57950 \[SC-High] Unit Mismatch in \_addDebt() Collateralization Check Allows Unbacked Debt Issuance and Protocol Insolvency
* \#56368 \[SC-Insight] \`AlchemistTokenVault::deposit()\` should use \`safeTransferFrom()\` instead of \`transferFrom()\`; \`AlchemistTokenVault::withdraw()\` should use \`safeTransfer()\` instead of \`transfer()\`
* \#57746 \[SC-Low] Broken contract ownership logic at AlchemistV3.sol
* \#58143 \[SC-Low] Unused Cap Enforcement Variables (adjusted)
* \#58645 \[SC-Medium] Incorrect WETH Wrapping Amount in \`MoonwellWETHStrategy.\_deallocate()\` Wraps \`ethRedeemed\` Instead of \`amount\`
* \#58757 \[SC-Critical] Forgotten cover in \_earmark() causes systematic over-earmarking and temporary freezing of user collateral
* \#57957 \[SC-Medium] Loss of EulerETH vault yields for Euler WETH Strategy
* \#56836 \[SC-Low] Ownership Transfer Failure in [\`AlchemistCurator\`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistCurator.sol) Prevents Future DAO Governance or Recovery
* \#57963 \[SC-High] Incorrect \_mytSharesDeposited Accounting in liquidate() Allows Theft of User Funds via Corrupted Bad Debt Ratio
* \#58070 \[SC-High] Forced-repay accounting lets borrowers erase debt without paying equivalent assets (protocol deficit / insolvency)
* \#57632 \[SC-High] Inflated TVL in \_mytSharesDeposited hides protocol insolvency
* \#56621 \[SC-Insight] Broken withdrawal logic in AaveV3ARBUSDCStrategy permanently locks user funds
* \#58125 \[SC-Critical] Repayment Fee Overpayment from Pooled Collateral
* \#56806 \[SC-Insight] Broken withdrawal logic in AaveV3ARBWETHStrategy permanently locks user funds
* \#57970 \[SC-High] \_forceRepay Leaves \`cumulativeEarmarked\` Stale
* \#57969 \[SC-Insight] Lack of incentive to liquidate small positions can cause the system to accumulate bad debt
* \#56348 \[SC-Insight] Incorrect APY calculation in MYTStrategy::\_approxAPY() causes underreported yields
* \#57788 \[SC-Medium] Missing \`\_claimRewards()\` Implementation in AaveV3ARBUSDCStrategy Leads to Permanent Loss of Aave Incentive Rewards
* \#58435 \[SC-High] Systemic Accounting Bug Leads to Protocol Insolvency
* \#57975 \[SC-Low] Broken admin rotation in acceptAdminOwnership() causes permanent governance lockout
* \#58762 \[SC-Insight] Manipulation of \`feeInUnderlying\` through front-running during liquidations on Ethereum
* \#57793 \[SC-High] \`cumulativeEarmarked\` variable is not updated in \`\_forceRepay\` function, breaking core internal logic and leading to user funds being stuck.
* \#57102 \[SC-High] TVL Overstatement from \_mytSharesDeposited Desync Enables Softened Liquidations & No‑Haircut Over‑Redemptions (Transmuter)
* \#56625 \[SC-Low] Broken ownership transfer logic in AlchemistCurator permanently freezes contract operations
* \#58203 \[SC-Medium] Moonwell Strategies Silent Failure Due to Unchecked mint() and redeemUnderlying() Return Values
* \#56658 \[SC-Insight] Transmuter's tokenURI does not revert for nonexistent tokenIds
* \#57989 \[SC-Low] Broken isValidSignature leads to fund freezing
* \#58086 \[SC-High] Mis-accounting of MYT outflows inflates TVL, distorts collateralization, and can DoS deposits/liquidations
* \#58088 \[SC-Low] Inadequate enforcement of global cap enables cumulative over‑allocation
* \#57816 \[SC-Insight] Critical Incentive Failure in calculateLiquidation Leads to Protocol Insolvency Risk During Global Bad Debt
* \#56855 \[SC-Medium] Liquidations Fail With Arithmetic Underflow When Forced Repayment Exhausts Collateral
* \#58089 \[SC-Low] Arithmetic underflow revert in \`\_deallocate\`
* \#56528 \[SC-Insight] Unbounded \`slippageBPS\` Can Freeze Withdrawals
* \#58443 \[SC-Critical] Incorrect Consumption of Yield Cover in redeem, Leading to Reuse of Accrued Yield
* \#58442 \[SC-High] Liquidation Breaks Core Accounting Invariant: Missing cumulativeEarmarked Update in \_forceRepay() Causes Permanent State Drift
* \#57995 \[SC-High] Missing Slippage Protection in TokeAutoUSDStrategy Allocation Function Leads to Permanent Value Loss
* \#58093 \[SC-Medium] MORPHO reward in \`MorphoYearnOGWETH\` will be lost or stuck
* \#56678 \[SC-High] Missing Internal MYT Shares Accounting in Liquidation Functions Causes Deposit Blocking and Protocol Insolvency Risk Through Inflated TVL Calculations
* \#57464 \[SC-High] Incorrect accounting in stargate strategy causes protocol insolvency and user liquidations
* \#58369 \[SC-High] Missing \_mytSharesDeposited Decrements in \_forceRepay/\_doLiquidation Leads to Smart Contract Unable to Operate Due to Lack of Token Funds
* \#57837 \[SC-Low] \`MoonwellWETHStrategy\` cant claim reward from Moonwell Comptroller
* \#58002 \[SC-Low] Missing submitRemoveStrategy Function
* \#57152 \[SC-High] Assets Permanently Locked Due to KillSwitch Flag
* \#56326 \[SC-Insight] Variable could be immutable
* \#58006 \[SC-Medium] \`MoonwellUSDCStrategy.\_allocate\` ignores Compound-style mint failures and corrupts vault accounting
* \#58772 \[SC-Critical] \`\_resolveRepaymentFee\` overpays liquidators when collateral is gone, letting attackers drain MYT
* \#58792 \[SC-High] the cumulativeEarmark does not decrease in \_forceRepay which lead to transfer more collateral from users even when all earmark debt cleared which breaks the alchemix v3 core logic
* \#58773 \[SC-Medium] In Stargate Incorrect Allocation Cap Accounting Leading to Unnecessary DoS
* \#58149 \[SC-Low] MorphoYearnOGWETH incorrectly reports loss and triggers StrategyDeallocationLoss event
* \#58449 \[SC-Medium] TokeAutoEth Strategy Balance-Approval Mismatch DOS
* \#58019 \[SC-High] Flawed killSwitch Implementation in MYTStrategy Leads to Permanent Loss of Funds
* \#57860 \[SC-High] Incorrect \`\_mytSharesDeposited\` accounting inflates collateral and underreports bad debt, enabling insolvency
* \#57861 \[SC-High] Missing Slippage Protection in Tokemak Autopool Allocation Functions Leads to Direct Theft of User Funds
* \#57169 \[SC-Low] ZeroXSwapVerifier Policy Bypass via RFQ fillData Prefix (Token & Amount Spoof)
* \#58452 \[SC-High] MYTStrategy Allocation underflow in deallocate() when allocation + profits exceed old allocation
* \#57197 \[SC-High] Incorrect \_totalLocked Reduction
* \#57604 \[SC-High] Nominal accounting mismatch in Moonwell strategies leads to permanent locking of all generated yield
* \#57867 \[SC-Medium] ZeroXSwapVerifier erroneously rejects Uniswap v3 swaps due to an an incorrect selector
* \#58636 \[SC-Low] Broken Two-Step Admin Transfer Prevents Legitimate Admin Succession in AlchemistCurator
* \#57212 \[SC-High] \_totalLocked is not properly decremented in the redeem function causing system insolvency.
* \#58189 \[SC-Low] Two-step mechanism to transfer ownership is broken due to incorrect access control
* \#56350 \[SC-Insight] Implementation contract AlchemistV3 not locked (\_disableInitializers() missing)
* \#56462 \[SC-Insight] Unused Mapping Causes Unnecessary Storage Gas Consumption
* \#58259 \[SC-Low] Broken operator logic inside AlchemistCurator
* \#58323 \[SC-Critical] The Alchemist::burn function experiences precision loss, resulting in the avoidance of protocol fees
* \#58324 \[SC-High] Incorrect Return Value in \_deallocate Function Leads to Permanent Fund Locking in MYTStrategy Implementations
* \#58363 \[SC-High] Accounting Corruption in Liquidations Due to Missing Global Counter Update
* \#57582 \[SC-Critical] Calling \_earmark one block apart skips the block's earmark value
* \#58076 \[SC-Insight] Fix unit mismatch in \_doLiquidation: collateralInUnderlying -> collateralInDebt
* \#57972 \[SC-High] liquidation doesn't update \_mytSharesDeposited
* \#58735 \[SC-Insight] \`\`calculateLiquidation\`\` reverts due to divide by \`\`ZERO\`\` if \`\`targetCollateralization = FIXED\_POINT\_SCALAR\`\`
* \#58472 \[SC-High] Liquidator Base Fee Seized but Not Paid Due to Post‑Deduction Balance Check
* \#57563 \[SC-Insight] Reward tokens being permanently frozen in TokeAutoUSDStrategy
* \#58386 \[SC-Low] Rewards claimed during deallocation remain stranded on strategy and unaccounted
* \#58689 \[SC-Critical] Incorrect deduction logic in \`AlchemistV3::redeem()\` may lead to insufficient contract collateral
* \#56815 \[SC-High] Missing \`\_mytSharesDeposited\` Decrements in Internal Outflows cause TVL Inflation & Deposit DoS
* \#56359 \[SC-High] Permanent Deposit Freeze After \_forceRepay() Misaccounts Freed Shares
* \#58605 \[SC-Medium] Missing \_claimRewards in AaveV3ARBUSDCStrategy leads to permanent freezing of accrued Aave incentives
* \#58615 \[SC-High] \_mytSharesDeposited didn't get updated after \_forceRepay && \_doLiquidation called
* \#58275 \[SC-High] account.rawLocked not clear even when debt is clear
* \#57725 \[SC-High] # \`AlchemistV::liquidate\` is not updating the \`\_mytSharesDeposited\` which makes it inflated and can cause deposits DoS and liquidations malfunction that may cause protocol insolvency.
* \#57752 \[SC-Medium] Aave and Euler incentives for MYT will be lost due to unimplemented \`\_claimRewards\` function
* \#56702 \[SC-Critical] \`claimRedemption\` would not return all alAsset that is not get converted to MYT in some case
* \#57665 \[SC-Low] Incorrect Balance Measurement in \`\_deallocate\` function of \`MorphoYearnOGWETHStrategy\`
* \#58270 \[SC-Critical] incorrect handling of debt cover in redeem can affect early liquidation and incorrectly sync accounts
* \#57697 \[SC-Low] Missing Recipient & \`from\` Checks in ZeroXSwapVerifier Enable Direct Asset Theft
* \#57866 \[SC-Low] Failure to verify the recipient's address can result in the theft of purchased tokens
* \#58578 \[SC-Low] ZeroXSwapVerifier allows attackers to drain strategy tokens via crafted calldata
* \#58575 \[SC-Low] Operator Limit Bypass
* \#58506 \[SC-Low] Adjusted Cap Limits Are Never Enforced
* \#58794 \[SC-High] Hardcoded 0 amount as the minSharesOut to depositMax(...) function call does not provide slippage protection
* \#56328 \[SC-Insight] Redundant require statement in EulerUSDCStrategy \_deallocate function leads to unnecessary gas consumption
* \#57506 \[SC-High] force repay don't update cumulativeEarmarked variable
* \#56936 \[SC-High] Missing \`\_mytSharesDeposited\` decrements on repay/liquidation → TVL drift, false over‑collateralization, and deposit‑cap DoS
* \#58797 \[SC-Low] The \`TokeAuto\` strategies implementation does not accurately report the actual assets held by the strategy
* \#58796 \[SC-Low] Incorrect balance snapshot in \_deallocate() causes wethRedeemed always = 0
* \#56801 \[SC-Insight] Function burn could be gas optimized
* \#57749 \[SC-Low] ZeroXSwapVerifier misses critical sender/recipient/minOut validations, allowing malicious 0x calldata to drain funds (Critical — Direct theft)
* \#57090 \[SC-Low] Ownership Transfer Failure in [\`AlchemistCurator\`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistCurator.sol) Prevents Future DAO Governance or Recovery
* \#56583 \[SC-Low] Wrong 2 step transferAdminOwnerShip logic and insufficient checks in AlchemistCurator.sol leads to permanent admin ownership loss
* \#58081 \[SC-Medium] Missing check in function \`AlchemistV3::setMinimumCollateralization\` could lead to set \`minimumCollateralization > globalMinimumCollateralization\`.
* \#58291 \[SC-Medium] Unlike setters collateralization functions, AlchemistV3::initialize doesnt enforce collateralization invariants allowing to break them
* \#57916 \[SC-Critical] Repay removes earmark, meant to be reducing debt while collateral is still reduced
* \#58257 \[SC-Low] In TokeAutoETH deallocate can be DOSed if the vault incuring losses
* \#56406 \[SC-Insight] \`getEstimatedYield\` never updates after snapshots
* \#56491 \[SC-Critical] User Collateral Loss Triggered by setMinimumCollateralization Update
* \#56830 \[SC-Low] Broken admin Ownership transfer Logic, acceptAdminOwnership() requires current admin instead of pending Admin, Blocking Role Claim.
* \#58513 \[SC-Low] Broken Access Control in AlchemistCurator.acceptAdminOwnership() Prevents Admin Transfer
* \#56911 \[SC-Low] Incorrectly implemented two-step admin ownership transfer mechanism prevents new admin to accept role
* \#56730 \[SC-Insight] Transmuter \`tokenURI()\` is not EIP-721 compliance
* \#57328 \[SC-Low] Once \`tokeLockDuration\` is the opposite of zero in \`TokeAutoEthStrategy\`, accumulated rewards in \`accToke\` can be stuck
* \#56389 \[SC-High] \`\_mytSharesDeposited\` is not updated on liquidation outflows which could lead to solvency illusion and misreported global ratios
* \#58519 \[SC-High] Double Counting of Collateral Due to \`\_mytSharesDeposited\` not being updated during liquidations
* \#57316 \[SC-Low] Allocation Cap Enforcement Missing & DeadCode
* \#57760 \[SC-High] MytStrategy.\_allocate/\_deallocate doesnt account for profit and loss
* \#56776 \[SC-High] TVL Manipulation via Missing \_mytSharesDeposited Decrement in Liquidations
* \#57553 \[SC-High] \_mytSharesDeposited is not updated in liquidations which breaks bad debt ratio/alchemistCR calculations and causes failures in bad debt handling and liquidation handling
* \#58067 \[SC-High] Asymmetric deallocation in TokeAutoEthStrategy leads to permanent WETH funds stuck in strategy
* \#58754 \[SC-High] Missing \`\_mytSharesDeposited\` decrements in \`AlchemistV3\` \`\_forceRepay\`/\`\_doLiquidation\`
* \#58168 \[SC-Medium] Safe Position Liquidation Vulnerability in AlchemistV3 When minimumCollateralization Equals collateralizationLowerBound
* \#58177 \[SC-High] Transmuter::claimRedemption cant update \_mytSharesDeposited leading to permanent underlying value state inside Alchemist
* \#58552 \[SC-Insight] Single transfer instead of multiple saves gas
* \#57787 \[SC-High] asset can be transferred to strategies even when the killSwitch enabled without posibility to use this funds for allocation
* \#57791 \[SC-Insight] Receipt Token Misconfiguration in Aave Strategies
* \#57617 \[SC-Critical] Protocol-paid repayment fee transfer allows draining of protocol MYT (yield)
* \#57189 \[SC-High] AlchemistCurator contract not implement setForceDeallocatePenalty
* \#58616 \[SC-Medium] Liquidation can revert due to 0 amount fee withdraw
* \#56582 \[SC-Low] AlchemistCurator::removeStrategy is unable to remove strategies from vaults due to wrong logic implementation
* \#57973 \[SC-Critical] repay doesnt set lastTransmuterTokenBalance leading to the same balance covering earmark twice
* \#58301 \[SC-Critical] Accounting Issue in Liquidation Logic After Force Repay we charge repayment fee even if collateral balanc cannot account for it
* \#56800 \[SC-Medium] Minimum collateral change lets liquidators seize compliant accounts
* \#56832 \[SC-Low] AlchemistCurator contract doesn't allow to remove strategies from the MYT morpho V2 vault.
* \#56845 \[SC-High] The deposit will be reverted because \`\_mytSharesDeposited\` references an outdated value
* \#57806 \[SC-Low] Staking Graph argument bounds are incorrectly defined
* \#57335 \[SC-Medium] Zero min-out ERC-4626 deposits cause under-mint and permanent allocation loss
* \#58728 \[SC-Medium] When the strategy is at a loss, the assets cannot be withdrawn
* \#58393 \[SC-Low] Wrong order in balance querying instructions in MorphoYearnOGWETHStrategy::\_deallocate function leads to always emit StrategyDeallocationLoss event
* \#57918 \[SC-High] Incorrect \`totalLocked\` Collateral Accounting in AlchemistV3
* \#58352 \[SC-Low] Assets Become Permanently Stuck in TokeAutoEth Strategy Due to Strict Balance Check
* \#58036 \[SC-Critical] Incorrect Fee Deduction May Drain Collateral Pool When Account Balance Is Insufficient
* \#58542 \[SC-Low] \[Low] Logic Error in MorphoYearnOGWETHStrategy.\_deallocate(): \`wethRedeemed\` Always Zero → All Deallocations Emit \`StrategyDeallocationLoss\`
* \#58357 \[SC-Low] Permanent Freezing of TokeAutoEth strategy rewards in MYT Vault
* \#57511 \[SC-Medium] Protocol could atleast be taking a part of the protocol fee
* \#58611 \[SC-Medium] Double-counting of earmarked debt repayments as cover leads to user funds being stuck and protocol insolvency.
* \#56572 \[SC-Insight] Aave V3 lending pool is immutable in Aave strategies
* \#56949 \[SC-Insight] Uncapped collateral transfer in redemption leads to accounting discrepancy enabling theft of user funds
* \#58515 \[SC-Medium] A liquidated position can end the liquidation process still below \`collateralizationLowerBound\`, allowing for double liquidation of positions.
* \#57272 \[SC-Medium] Silent Failures on Moonwell Deposit are not catched by strategy
* \#57057 \[SC-Low] Wrong order of balance checks in MorphoYearnOGWETHStrategy
* \#57251 \[SC-Low] Curator Cannot Remove Adapter Due to Timelock Requirement
* \#56887 \[SC-Low] Incorrect balance tracking in MorphoYearnOGWETHStrategy \_deallocate function leads to wrong loss event emission(Resend))
* \#58787 \[SC-Medium] When allocation amount is greater than the maxDeposit of TokeAutoETh.sol, the remaining is stuck in TokeAutoEth.sol
* \#56324 \[SC-Low] Missing \`\`\`from==owner \`\`\`check in transferFrom verifier → direct theft of user funds
* \#58658 \[SC-High] cumulativeEarmarked not updated
* \#58087 \[SC-Medium] MoonwellUSDCStrategy ignores redeemUnderlying error codes → temporary freezing of funds (withdrawals revert)
* \#58743 \[SC-Low] ZeroXSwapVerifier Recipient Validation Bypass
* \#58730 \[SC-Medium] An attacker can prevent any TokenAuto strategy allocation by making a donation to the vault of as little as 1 wei of underlying token
* \#58181 \[SC-Medium] A griefer can cause a permanent DoS in TokeAutoETH/TokeAutoUSDCStrategy::allocate.
* \#57852 \[SC-Critical] Old borrowers steal from new borrowers after redemptions are claimed
* \#58703 \[SC-Insight] Cached interest rate calculation in PeapodsETH strategy leads to inaccurate APR/APY estimates
* \#56809 \[SC-High] Vulnerable redemption survival ratio in \_sync allows theft of alTokens
* \#57066 \[SC-Critical] A malicious actor can keep calling \`poke\` at every block to prevent collateral earmarking exposing transmuter users to delayed redemptions and loss of funds
* \#58564 \[SC-Critical] Earmarked funds fail to accumulate when \_earmark is called in consecutive blocks
* \#58544 \[SC-Critical] it is possible to underflow on \`\_sync\` making positions bricked forever
* \#58146 \[SC-Insight] Whitelist can be disabled repeatedly, contradicting intended program behavior.
* \#56365 \[SC-Critical] Liquidation Fee Overdraft Drains Pooled Collateral
* \#56775 \[SC-Medium] Permanent freezing of funds from precision/dust + strict deallocation check
* \#56622 \[SC-Critical] Repayment Fee Overpays Liquidators Using Pooled Collateral After \_forceRepay
* \#56740 \[SC-Critical] Unbounded Liquidation Fee Allows Theft of Shared Collateral
* \#57330 \[SC-Critical] \_resolveRepaymentFee returns initial fee when fee is greater collateral balance
* \#56895 \[SC-Insight] Function approveMint is vulnerable to race conditions
* \#58209 \[SC-Medium] Lack of Slippage Protection in Transmuter.claimRedemption and AlchemistV3.withdraw Leads to User Yield Losses
* \#57114 \[SC-Low] Inherited \`setAdmin\` function allows to bypass two-step admin ownership transfer mechanism
* \#56442 \[SC-High] Inflated \`\_totalLocked\` because vault yield accrual would skew \`\_collateralWeight\` calculation
* \#58035 \[SC-High] killSwitch early-return in strategy causes vault-to-adapter asset leakage, mis-accounting, and deallocation DOS
* \#56878 \[SC-Medium] The permissionedCalls check can be bypass
* \#58198 \[SC-Low] Broken Two-Step Admin Transfer Pattern
* \#56418 \[SC-Low] Two step owner transfer is broken and can lead to unforseen damages
* \#57622 \[SC-Low] Lack of claimed reward handling in MYT strategies will keep all external token rewards stuck forever
* \#57017 \[SC-Medium] \`AaveV3ARBWETHStrategy\`cant claim AAVE incentive
* \#56465 \[SC-Low] \`getTotalDeposited\` doesn't reflect the correct total deposited
* \#57770 \[SC-Medium] Admin Can Bypass \`permissionedCalls\` Protection Using Multicall
* \#56956 \[SC-High] Lack of slippage control in Tokemak strategies can make MYT suffer losses on allocation
* \#58010 \[SC-High] Slippage tolerance not enforced in TokeAutoUSDStrategy
* \#56560 \[SC-High] Liquidation base fee transfer is gated by a condition that’s usually false
* \#57883 \[SC-High] \_mytSharesDeposited Updates in Liquidation Functions Leads to Critical TVL Inflation
* \#58129 \[SC-High] Missing \_mytSharesDeposited Update in \_forceRepay() Causes Accounting Inconsistency which can DOS deposit and Liquidation
* \#57544 \[SC-High] \`\_mytSharesDeposited\` is not reduced upon fee transfers to protocol
* \#58116 \[SC-High] TVL Accounting Mismatch Leading to Protocol Insolvency
* \#58260 \[SC-High] Inconsistent collateral accounting where Force-Repay/Liquidation transfer out MYT without adjusting TVL
* \#57439 \[SC-Low] Incorrect \`badDebtRatio\` rounding in \`Transmuter::claimRedemption()\` may cause funds to become permanently stuck
* \#57291 \[SC-Insight] Hardcoded Slippage in MYT Strategy
* \#58751 \[SC-Medium] \`setMinimumCollateralization\` allows for increasing the current \`minimumCollateralization\` , instantly exposing users to risk of liquidation
* \#58185 \[SC-Medium] Incorrect \_survivalAccumulator accounting logic after \_earmarkWeight reaches 128 breaks core system invariants and can lead to protocol insolvency
* \#58338 \[SC-Critical] AlchemistV3 Repayment Fee Can Exceed Remaining Collateral Leading to Position Insolvency
* \#56706 \[SC-Medium] StargateEthPoolStrategy Incomplete ETH Wrapping Causes Withdrawal DoS
* \#57526 \[SC-Medium] \`StargateEthPoolStrategy\` rounding mismatch freezes \`VaultV2\` allocations
* \#57565 \[SC-Medium] The amount of dust will be permanently locked in \`StargateEthPoolStrategy\`
* \#58022 \[SC-Medium] Accounting Mismatch and Fund Stuck Due to Dust ETH on StargateEthPoolStrategy
* \#58427 \[SC-Medium] \`StargateEthPoolStrategy::\_allocate()\` and \`\_deallocate()\` Inconsistent Dust Handling Causes ETH to be Permanently Locked in Strategy Contract
* \#56561 \[SC-Insight] Fee amount is recomputed multiple times when the initial value has already been cached
* \#58210 \[SC-Low] Incorrect balance measurement in deallocation disables loss detection in MorphoYearnOGWETH.
* \#57740 \[SC-High] EulerETH strategy will have WETH locked in the strategy contract
* \#56798 \[SC-Critical] Flash-Vote Exploit Drains All Funds via AlchemistAllocator
* \#58215 \[SC-High] Funds Can Become Permanently Stuck in Adapter When Kill Switch is Enabled
* \#58423 \[SC-Low] Pending Admin Cannot Accept Ownership Transfer in \`AlchemistCurator\`
* \#58346 \[SC-High] \_forceRepay() fails to decrement cumulativeEarmarked, breaking earmark invariant and skewing redemptions
* \#58196 \[SC-High] AaveV3ARBUSDCStrategy strategy will have its reward stuck in Aave USDC
* \#58105 \[SC-Medium] ZeroXSwapVerifier decodes execute payload with wrong ABI (bytes vs bytes\[]) → temporary freezing of funds
* \#58249 \[SC-Low] Broken Two-Step Admin Handover in AlchemistCurator
* \#57369 \[SC-High] Deallocation may revert due to an underflow
* \#57923 \[SC-Insight] Redundant Synthetic Transfers in claimRedemption When amountNottransmuted is Zero
* \#57183 \[SC-Medium] Missing Incentive Rewards Claiming in Multiple Strategy Contracts
* \#57812 \[SC-Medium] No function to claim Aave Incentives
* \#58239 \[SC-Medium] Missing Aave incentives rewards claiming mechanism leads to permanent loss of protocol royalties
* \#56427 \[SC-Insight] src/utils/PermissionedProxy.sol::setPermissionedCall incomplete event emission because it doesnt include value argument for signature
* \#56347 \[SC-Insight] burn contains redundant calculations
* \#56516 \[SC-High] allocate assets in killSwitch mode can lead to assets stuck on contract
* \#57394 \[SC-Low] \`acceptAdminOwnership()\` only allows the current admin to finalise transfers
* \#58124 \[SC-Low] Direct Theft of Funds via Malicious actions\[] in execute() Call Due to Incorrect Calldata Verification
* \#58130 \[SC-Medium] Asymmetric Validation in Collateralization Setters Allows Protocol Misconfiguration Breaking All Borrowing
* \#58080 \[SC-Medium] Aave V3 strategies fail to claim OP/ARB liquidity mining rewards, causing permanent loss of yield
* \#58131 \[SC-Critical] Rounding Errors in Debt-to-Collateral Conversions Allow Attackers to Drain Protocol Assets
* \#58627 \[SC-Low] Incorrect delta calculation in \_deallocate() causes wethRedeemed to always be zero.
* \#57825 \[SC-High] Forced repay cover enables double-counted debt reduction in redeem
* \#56923 \[SC-High] Missing cumulativeEarmarked Update in \_forceRepay Causes Incorrect Debt Accounting in AlchemistV3
* \#56927 \[SC-Medium] setMinimumCollateralization function also needs a another check
* \#57678 \[SC-High] Liquidation fee is deducted from user but not paid to liquidator
* \#56498 \[SC-Low] Reserve Drainage Due to Incorrect Balance Measurement
* \#58150 \[SC-High] Missing Slippage Protection in \`TokeAutoUSDStrategy::\_allocate\` Leads to Direct Theft of User Funds via MEV Sandwich Attacks
* \#56518 \[SC-Insight] \`claimWithdrawalQueue\` discards claimed amount
* \#56982 \[SC-Medium] Incorrect function selectors used in ZeroXSwapVerifier
* \#58101 \[SC-Critical] Repayment-only liquidation overpays fee from pooled collateral
* \#58266 \[SC-High] Partial liquidation strands base fee due to post-seizure balance check
* \#58322 \[SC-Low] Incorrect Emit Due to Wrong Ordering of wethBalanceBefore Calculation
* \#58360 \[SC-Low] Round-down calculation in \`convertToShares()\` leads to deallocation failure in TokeAutoEth strategy
* \#58356 \[SC-Insight] The Alchemist TokeAuto Strategies doesn't use recommended best practice by TokeAuto.
* \#58408 \[SC-Low] Underflow account.rawLocked on \_subDebt due to rounding inconsistency
* \#58410 \[SC-Low] Tokemak Strategy Deallocation Causes TOKE Token Lockup
* \#58419 \[SC-Low] AlchemistCurator two-step ownership transfer mis-implemented
* \#58469 \[SC-Low] Pending Admin Cannot Accept Ownership
* \#58782 \[SC-High] Rewards earned by EulerARBUSDCStrategy will not be withdrawable from Euler pool on Arbitrum
* \#56689 \[SC-Low] Reward token TOKE is stuck in MYT
* \#58555 \[SC-Low] \`AlchemistCurator\` 2-step ownership transfer is implemented incorrectly
* \#58428 \[SC-Low] TOKE reward loss when calling deallocate
* \#58376 \[SC-Low] claimRewards() function permanently locks earned Toke reward token on Morpho VaultV2
* \#58527 \[SC-Low] Complete loss of all reward value on TokeAutoEthStrategy \_claimRewards
* \#57692 \[SC-High] AlchemistV3 Liquidation Fee Loss Vulnerability
* \#58524 \[SC-High] When liquidating, there are cases where the Fee is not paid to the liquidator.
* \#56757 \[SC-High] Incorrect leftover-collateral check blocks liquidator fee payment leading broken incentives & delayed deleveraging
* \#56451 \[SC-Low] \`AlchemistAllocator::allocate()\` and \`deallocate()\` do not enforce cap checks as intended
* \#57644 \[SC-Low] Unenforced cap logic in AlchemistAllocator allows not controlled allocations
* \#58325 \[SC-Low] Operator Can Shift vault Funds to Risky Strategies Without Oversight, leading to potential loss of user funds.
* \#57101 \[SC-Critical] Same-block earmark early-exit leaves stale transmuter balance, causing under-earmarking
* \#58714 \[SC-Low] Pending Admin cannot accept Ownership in \`AlchemistCurator.sol\`
* \#58734 \[SC-Low] Broken strategy realAssets calculation
* \#58742 \[SC-High] Liquidators will not earn fees in some cases
* \#58759 \[SC-High] Yield Stuck in Adapter Contracts Forever
* \#58781 \[SC-High] \_totalLocked Accounting Mismatch Leading to Token Balance Deficit in AlchemistV3
* \#58780 \[SC-High] WETH Yield will be locked on AaveWETH pool on Arbitrum
* \#56983 \[SC-Low] Tokemak rewards sent to MYT vault contract (not strategy) -> rewards stranded
* \#58056 \[SC-Low] The Auto ETH and USDC staking rewards will stuck in vault
* \#57067 \[SC-Low] Overstated Per-Account Locked Collateral Due to Global Clamp in \_subDebt
* \#56332 \[SC-Low] Pending Admin Cannot Accept Ownership
* \#57479 \[SC-Low] Logical bug in \`AlchemistCurator::acceptAdminOwnership\`, asking to current admin to accept ownership.
* \#58007 \[SC-Low] \`pendingAdmin\` cannot call \`acceptAdminOwnership()\` to accept admin role
* \#58333 \[SC-Low] Incorrect onlyAdmin Modifier in acceptAdminOwnership
* \#57680 \[SC-High] PeapodsETHStrategy unable to withdraw yield from price share increase
* \#57476 \[SC-High] \_forceRepay() fails to decrement global cumulativeEarmarked
* \#58722 \[SC-Medium] TokenAuto strategy allocation uses maxDeposit which may allocate less than requested, leaving any excess funds permanently locked
* \#58522 \[SC-High] Earmark consumes excess cover, inflating cumulativeEarmarked
* \#56975 \[SC-High] Liquidation Fee Trapping in AlchemistV3
* \#57024 \[SC-Low] \`wethBalanceBefore\` is computed after withdrawal in \`\_deallocate\` function in MorphoYearnOGWETHStrategy contract, leading to systematic StrategyDeallocationLoss event emission.
* \#58115 \[SC-Medium] Incorrect WETH deposit amount prevents deposited ETH through \`receive\` function to cover strategy loss.
* \#58639 \[SC-Medium] \`\`Off by One\`\` issue in the \`\`\_forceRepay()\`\` function causes protocol to lose funds in the form of \`\`protocol fee\`\`.
* \#58403 \[SC-Medium] Missing Checks for Transaction Return Values in Moonwell Strategies
* \#57227 \[SC-Medium] Unchecked Return Codes in MoonwellUSDCStrategy Leading to Stuck Funds
* \#56909 \[SC-Low] Incorrect balance snapshot in strategy deallocation causes false loss events and masks real shortfalls
* \#56529 \[SC-Low] Incorrect token balance calculation in MorphoYearnOGWETHStrategy.sol::\_deallocate() leads to wrong event emitted every time
* \#57926 \[SC-Low] The conditional 'StrategyDeallocationLoss' event in \`MorphoYearnOGWETHStrategy::\_deallocate\` gets logged all the time due a misplacement in variable declaration
* \#56961 \[SC-Low] Incorrect balance snapshot check in \`\_deallocate()\` logs false deallocation loss in MorphoYearnOGWETH strategy
* \#56962 \[SC-Low] Balance Check Logic Error in \_deallocate() Function Leads to Broken Loss Detection and False Event Emissions
* \#58383 \[SC-High] Due to \`cumulativeEarmarked\` not being updated in \`Alchemix::\_forceRepay\` user funds are locked longer due to slower debt decay and Calculation of System Collaterization Rate is Inc...
* \#58769 \[SC-High] \`\_forceRepay\` fails to decrement global \`cumulativeEarmarked\`, causing redemption accounting desynchronization and potential protocol-wide redemption halt
* \#58799 \[SC-High] \`\_forceRepay\` does not reduce \`cumulativeEarmarked\` which leads to wrong accounting: users debts are incorrectly higher which can cause wrongful liquidations
* \#57704 \[SC-High] Missing Global State Update in \_forceRepay Leads to Permanent Freezing of Unclaimed Yield
* \#57023 \[SC-High] Global earmark not reduced in \_forceRepay lets redeem() over-burn global debt (cross-account leakage, protocol insolvency)
* \#58635 \[SC-High] \`\`cumulativeEarmarked\`\` is not subtracted in \`\`\_forceRepay()\`\`.
* \#57777 \[SC-Low] ZeroX swap verifier bypass enables direct theft of user funds
* \#56960 \[SC-Medium] Missing Slippage Protection During Redemption Execution, lead to loss of token for user.
* \#58079 \[SC-Low] Missing from validation in ZeroXSwapVerifier.verifySwapCalldata() enables direct theft of approved funds
* \#58289 \[SC-Low] Missing addresses Verification in ZeroXSwapVerifier
* \#57516 \[SC-Low] Arbitrary External Call in ZeroXSwapVerifier Leads to Theft of Unclaimed Yield
* \#57606 \[SC-Insight] Attacker can DoS deposits by hitting the deposit cap
* \#58492 \[SC-Medium] Unbounded Deposit Exposure in TokeAutoEthStrategy::\_allocate()
* \#57532 \[SC-High] Assets are not accounted for when the contract is in killSwitch mode
* \#57849 \[SC-High] Funds gets stuck even when killswitch is enabled
* \#58127 \[SC-Critical] Users can invoke the poke() function whenever the lastEarmarkDebtBlock is exactly one block behind the current block.number which lead to affecting users earmarked debt
* \#58723 \[SC-High] \`cumulativeEarmarked\` is not updated at \`\_forceRepay\`
* \#58418 \[SC-Low] \`verifySwapCalldata\` cant verify the output token of the swap
* \#57028 \[SC-Insight] Wrong amount variable in Repay event
* \#58705 \[SC-Low] Mismatch between emitted protocol fee and actual fee paid in \_forceRepay due to strict inequality check
* \#57832 \[SC-Insight] Cap Logic Error in AlchemistAllocator
* \#58534 \[SC-High] Zero Slippage Protection in Toke strategies Allocation
* \#58702 \[SC-High] No slippage provided in Auto strategy implementation will open room for MEV attacks
* \#56902 \[SC-High] Strategy Adapter \`AaveV3OPUSDCStrategy\` would not work well with aToken rebasing mechanism
* \#58287 \[SC-High] \_mytSharesDeposited is not updated on some token transfer
* \#58399 \[SC-Critical] Precision Loss in badDebtRatio Calculation Causes Overpayment and DOS
* \#58768 \[SC-High] \`\_mytSharesDeposited is not updated during liquidations, breaking core accounting
* \#56791 \[SC-High] Missing \`\_mytSharesDeposited\` Decrements in Token Transfers
* \#58207 \[SC-High] AlchemistV3 \_mytSharesDeposited Not Reduced When Repaid Collateral Sent to Transmuter
* \#57138 \[SC-Critical] Protocol subsidizes repayment fees during liquidation
* \#58040 \[SC-Low] removeStrategy() is Non-Functional
* \#57346 \[SC-Low] AlchemistAllocator Compares Incompatible Units (Asset Wei vs WAD Percentage)
* \#57441 \[SC-Critical] Repay-Only Fee Drain in AlchemistV3
* \#57982 \[SC-Low] Permanently stuck rewards in the Vault
* \#58473 \[SC-Low] Wrong redeemed amount calculation in MorphoYearnOGWETH strategy
* \#57378 \[SC-High] Impossible to withdraw yield from strategies
* \#58530 \[SC-High] Protocol insolvency via stale \`\_totalLocked\`: zeroed \`\_totalLocked\` prevents \`\_collateralWeight\` update in \`redeem()\` leading to missed collateral haircut
* \#56555 \[SC-Critical] User can avoid Bad Debt ratio scaling when claiming redeem, leading to protocol insolvency
* \#58110 \[SC-Low] MorphoYearnOGWETHStrategy will always report strategy loss
* \#58313 \[SC-Medium] Incorrect allocation accounting and dust handling in \`StargateEthPoolStrategy\` causes systematic loss, cap mis-accounting, and deallocation reverts
* \#58061 \[SC-High] Incorrect collateral and fee Check in \_doLiquidation Allows Liquidator to loose fee.
* \#57637 \[SC-Low] acceptAdminOwnership doesn't allow expected user approval
* \#56709 \[SC-Low] ZeroXSwapVerifier Missing Source Validation
* \#57208 \[SC-Insight] It is possible to prevent lowering the deposit cap by front-running
* \#57646 \[SC-Medium] ABI Signature Mismatch in ZeroXSwapVerifier Causes Complete Failure to Verify Legitimate 0x Settler Transactions
* \#58133 \[SC-Low] TOKE Rewards Permanently Locked in Strategy adapter
* \#57587 \[SC-Critical] \_earmark() reduction of transmuterDifference does not always account for the full transmuter balance diff which can cause permanent earmark to accrue in Alchemist
* \#57483 \[SC-Medium] Fees could be skipped when there is not enough collateral
* \#57771 \[SC-Medium] Fee not collected in \_forceRepay when should
* \#58718 \[SC-Medium] In \_forceRepay Protocol Fee Collection Leads to Theft of Unclaimed Yield
* \#58488 \[SC-Low] TokeAutoUSDStrategy claims rewards to itself automatically when deallocate is called but since reward token is Tokemak the rewards remain permanently locked
* \#57196 \[SC-High] Artificially inflated \`\_mytSharesDeposited\` in \`AlchemixV3.sol\` deflates bad debt ratio in \`Transmuter.sol\`
* \#58502 \[SC-High] Deposit cap denial of service due to stale \_mytSharesDeposited during liquidation
* \#58234 \[SC-Critical] There is a problem related ot Repayment Fee Overpayment can lead to Protocol Insolvency
* \#58491 \[SC-High] \_mytSharesDeposited Not Reduced on Liquidation, leading to Deposit Cap Bypass and potential insovency
* \#56628 \[SC-High] \`\_liquidate\` does not update \`\_mytSharesDeposited\` that is reduced by fees
* \#58395 \[SC-High] Repayment fee exit leaves \`\_mytSharesDeposited\` inflated, hiding protocol insolvency
* \#57533 \[SC-High] Inaccurate TVL Calculation Prevents Liquidations, Leading to Protocol Insolvency Risk
* \#58520 \[SC-Low] Pending admin cannot accept ownership
* \#58606 \[SC-High] Missing collateral accounting in liquidation leads to inflated bad debt calculations
* \#56545 \[SC-High] Force Repayment Leaves Stale Global Earmarks, Freezing Transmuter Redemptions
* \#56846 \[SC-Medium] Liquidation will return because of insufficient funds
* \#56719 \[SC-High] The function \_forcerepay reduces debt before clamp, creating unbacked loan forgiveness and protocol insolvency
* \#57123 \[SC-Low] Incorrect 2 step ownership in AlchemistCurator
* \#58464 \[SC-Critical] Repayment fee paid from protocol funds when user collateral is depleted
* \#58138 \[SC-Critical] Liquidator fees could surpass the user remaining collateral resulting in protocol insolvency
* \#56602 \[SC-Low] Function takes incorrect modifier
* \#58077 \[SC-Low] Reward tokens are incorrectly claimed to strategy contract during deallocation leads to permanent token loss
* \#57977 \[SC-High] Inconsistent rawLocked State of a user after subdebt Leads to Irrecoverable User Collateral Loss
* \#56737 \[SC-Medium] The return value of mint is not checked
* \#58334 \[SC-Medium] Incorrect Function Selectors
* \#58244 \[SC-Low] Incorrect balance check order in \`MorphoYearnOGWETH\` strategy leads to false deallocation loss events
* \#58033 \[SC-Medium] Unimplemented \`\_claimRewards()\` Function Results in Permanent Freezing of Aave Incentive Rewards
* \#57545 \[SC-Medium] Stargate ETH Strategy Rounding Bug

</details>
