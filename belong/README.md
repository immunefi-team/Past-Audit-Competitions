# Belong

## Reports by Severity

<details>

<summary>Critical</summary>

* \#57039 \[SC-Critical] Processing Fee Logic Flaw in payToVenue() Causes Permanent Loss of Platform Revenue
* \#57298 \[SC-Critical] State-sync omission in \`Staking\` transfers forces transferred sLONG holders into penalized emergency exits
* \#57635 \[SC-Critical] ERC4626 share transfers desynchronize time-lock ledger, blocking standard withdrawals for recipients
* \#56941 \[SC-Critical] Staking vault vulnerable to first-depositor donation attack
* \#57628 \[SC-Critical] Improper Transfer can lead to funds been frozen
* \#57809 \[SC-Critical] inflation of shares in staking contract
* \#57813 \[SC-Critical] Transfer recipients will pay unwarranted emergency withdrawal penalties for share positions they legitimately own
* \#57399 \[SC-Critical] ERC4626 staking lockbook breaks share fungibility - partial transfers can DoS withdrawals
* \#56850 \[SC-Critical] Donation attack posible on Staking.sol because its totalAsset() uses asset.balanceOf()
* \#56907 \[SC-Critical] Attacker can steal first depositor's asset with inflation attack
* \#57398 \[SC-Critical] Incorrect Platform Subsidy Processing in LONG Payments Causing Venue Payout Failures
* \#57482 \[SC-Critical] Front-running a donation can inflate the share causing users to lose funds
* \#57716 \[SC-Critical] ERC4626 Inflation bug in Staking contract
* \#56896 \[SC-Critical] staking contract is vulnerable to inflation attack making malicious 1st staker Grief the following stakers
* \#56863 \[SC-Critical] First Depositor Advantage
* \#57008 \[SC-Critical] emergencyWithdraw Function Malfunction Due to Missing Validation in \_removeAnySharesFor
* \#57932 \[SC-Critical] Attacker can bypass stake lock
* \#57401 \[SC-Critical] ERC4626 Inflation Attack Vulnerability
* \#57685 \[SC-Critical] Vulnerabilities in the design of the token's staking mechanism resulted in financial harm to users involved in transfer-related operations.
* \#57736 \[SC-Critical] First depositor attack is possible
* \#56872 \[SC-Critical] Freezing of funds
* \#57942 \[SC-Critical] Transferred \`sLONG\` Shares Are Permanently Unredeemable Due to Missing Stake Entry Creation
* \#57924 \[SC-Critical] The staking contract is suceptible to the classic first depositor exploit

</details>

<details>

<summary>High</summary>

* \#57296 \[SC-High] Retroactive referral tier underpayment in RoyaltiesReceiverV2 due to dynamic shares applied to historical funds
* \#57290 \[SC-High] MEV Sandwich Attack Vulnerability - No User-Controlled Slippage Protection in Token Swaps
* \#57885 \[SC-High] Dynamic Share Drift in RoyaltiesReceiverV2
* \#57888 \[SC-High] Referral tier upgrades freeze legacy royalties
* \#57898 \[SC-High] Unprotected Swap Function Allows Sandwich Attacks
* \#57061 \[SC-High] Retroactive Share Recalculation Causes Royalty Distribution Failure
* \#57221 \[SC-High] Incorrect Processing Fee Calculation Causes Venue Payouts to Be Misallocated
* \#57877 \[SC-High] AccessToken creators can bypass fees so that platform address will receive 0 fees.
* \#57829 \[SC-High] Incorrect Fee Implementation in payToVenue LONG Payment Path Causes Protocol Fees to be Permanently Locked in Escrow
* \#57671 \[SC-High] RoyaltiesReceiverV2.shares() referralShare uses dynamic values, which may result in failure to release funds properly.
* \#57676 \[SC-High] Cross-token accounting in Receiver allows permanent freezing of ERC20 royalty payouts
* \#57452 \[SC-High] On-chain Quoter reliance and spot-price-based swaps enable pool manipulation and value extraction from protocol-controlled conversions (USDC ↔ LONG).
* \#57435 \[SC-High] Missing Slippage Protection Enables Direct Theft via MEV Sandwich Attacks
* \#57586 \[SC-High] Calculating slippage for swap onchain does not prevent slippage loss
* \#56841 \[SC-High] Sudden addition of rewards will be frontrun with deposits just to steal part of reward
* \#57515 \[SC-High] Cross‑token accounting is broken
* \#57733 \[SC-High] swapExact's slippge is not works as expected
* \#56881 \[SC-High] Temporary Claim Freezing
* \#57237 \[SC-High] Cross-token math contaminates payouts in \`Receiver\`
* \#57786 \[SC-High] Malicious Users can frontrun Staking::distributeRewards to claim majority of rewards
* \#57076 \[SC-High] Incorrect slippage would result in swap manipulations

</details>

<details>

<summary>Medium</summary>

* \#57423 \[SC-Medium] Unbounded Gas Consumption in Emergency Redemption Enables Low-Cost DoS Against Staking Vault Users
* \#57060 \[SC-Medium] Unconditional subsidy withdrawal in \`payToVenue()\` leads to DoS when venue’s LONG pool is depleted
* \#57285 \[SC-Medium] Incomplete Signature in Factory.produce() Enables Full AccessToken Hijacking and Direct Fund Theft
* \#57310 \[SC-Medium] Unaccounted Processing Fees in LONG Payment Path
* \#57327 \[SC-Medium] Title: Front-running Leads to Denial of Service and Unauthorized Referral Farming in Creation Functions.
* \#57594 \[SC-Medium] Signature-collision from abi.encodePacked (adjacent strings) enables unauthorized NFT actions (mint/URI abuse)
* \#57703 \[SC-Medium] DoS with Revert via Unbounded Loop
* \#57284 \[SC-Medium] Updating minimum staking period griefs previously unlocked users
* \#57929 \[SC-Medium] \`\_produce\` function doesn't check if creator is the caller allowing frontrunning attacks
* \#57723 \[SC-Medium] Signature Replay, Front‑Run, and Timing Control Issues
* \#57724 \[SC-Medium] Universal signature for \`produce()\` allows front-running and collection hijack
* \#57738 \[SC-Medium] Name-squatting / front-run on produce() allows attacker to preempt legitimate creator and capture future mint revenue
* \#57775 \[SC-Medium] PayToVenue will revert due to NotEnoughLONGs funds in the Escrow contract
* \#57015 \[SC-Medium] Unbounded Array Loop
* \#57796 \[SC-Medium] Signature hashing collision in SignatureVerifier lets attacker deploy forged AccessToken/CreditToken metadata (Critical — Unintended alteration of what the NFT represents)
* \#57800 \[SC-Medium] Signature Replay Vulnerability in BelongCheckIn::distributePromoterPayments
* \#56860 \[SC-Medium] Hash collision in signature verification
* \#57519 \[SC-Medium] Unbounded Stake Array Allows permanent withdraw lock via Dust Deposits on Behalf of Victims
* \#56810 \[SC-Medium] AccessToken: Cross‑contract signature replay allows unauthorized minting on other collections
* \#57203 \[SC-Medium] \[REVISED] Malicious AccessToken creator can steal gas via \`mintStaticPrice\` or \`mintDynamicPrice\`
* \#57940 \[SC-Medium] Deterministic address collision in Cairo deployment causes DoS and unintended receiver sharing
* \#57426 \[SC-Medium] Dynamic Price Signature Replay Allows Unlimited Minting at Historical Prices
* \#57712 \[SC-Medium] Receiver Deployment DoS via Salt Reuse
* \#57314 \[SC-Medium] Signature Replay and Hash Collision via \`abi.encodePacked\` in SignatureVerifier.sol
* \#57271 \[SC-Medium] Incorrect penalty calculation on emergency withdrawals/redemption's
* \#57485 \[SC-Medium] emergencyWithdraw cost more penalty than expected
* \#57669 \[SC-Medium] Stakers will bypass minStakePeriod time locks and extract rewards without commitment through emergency withdrawal mechanism
* \#57236 \[SC-Medium] AccessToken Collection Front-Running Attack - Permanent Ownership Hijack
* \#56867 \[SC-Medium] Signature Collision caused counterfeit AccessToken collections with arbitrary name/symbol/URI
* \#57283 \[SC-Medium] Unauthorised Promoter Payouts due to Signature Replay Attack.
* \#57717 \[SC-Medium] Attacker can spam tiny stakes to a victim and make their withdrawal run out of gas (griefing DoS)
* \#57358 \[SC-Medium] Unlimited Stake Entries Allow Account Griefing via Tiny Deposits
* \#57891 \[SC-Medium] Signature replay lets attackers hijack NFT collection deployment
* \#57766 \[SC-Medium] attacker can permanently lock any user’s funds
* \#57848 \[SC-Medium] Permanent Freezing of funds due to no minimum stake limit
* \#57939 \[SC-Medium] Signature Collision via abi.encodePacked
* \#57279 \[SC-Medium] Signature replayability — repeated use of signed access tokens allows duplicate mints (High)
* \#57580 \[SC-Medium] Signature Replay Enables Frontrunning of produce/produceCreditToken
* \#57854 \[SC-Medium] Front-Running Attack Allows Collection Ownership Theft
* \#57373 \[SC-Medium] Signature Replay Vulnerability Due to Missing Nonce and Deadline Checks
* \#57864 \[SC-Medium] abi.encodePacked Hash Collision Vulnerability in Dynamic Type Encoding Permits Malicious Signature Bypass, Enabling Unauthorized and Repeatable Transaction Execution
* \#56869 \[SC-Medium] Hijacking deployment of \`AccessToken\` and stealing ownership to prevent further deployments
* \#56814 \[SC-Medium] Users Can Create unauthorized AccessToken collections by exploiting \`abi.encodePacked\` collision
* \#57875 \[SC-Medium] Signature bypass lets creators alter key AccessToken parameters before deployment
* \#57245 \[SC-Medium] Needless iterations in for loops should be removed for better optimization and code maintenance
* \#57677 \[SC-Medium] Signature Replay in venueDeposit Enables Affiliate Referral Code Hijacking Leading to Unauthorized Commission Theft
* \#57089 \[SC-Medium] Unauthorized collection hijack via unsigned creator
* \#57917 \[SC-Medium] Penallty can be bypassed in \`Staking.sol::\_emergencyWithdraw\`
* \#57911 \[SC-Medium] Signature are malleable in SignatureVerifier.sol.
* \#57388 \[SC-Medium] Cross contract signature replay because verifying contract is not included in the digest
* \#57895 \[SC-Medium] Lack of msg.sender Validation in Collection Creation Signature Enables Front-Running Attack Leading to Creator Impersonation
* \#57362 \[SC-Medium] Attacker can DoS user withdraw in staking contract
* \#57194 \[SC-Medium] Signature replay across collections (missing contract binding)
* \#57938 \[SC-Medium] \`\_produce\` function doesn't check if creator is the caller allowing frontrunning attacks
* \#57702 \[SC-Medium] The long payment path is sensitive to the long inventory in Escrow, and insufficient inventory can easily lead to business unavailability (DoS of long payments).
* \#57790 \[SC-Medium] Withdrawal Denial-of-Service via Dust Stake Spam
* \#57458 \[SC-Medium] DoS/Griefing in batch ETH payout: malicious payee \`receive()\` can revert and block \`releaseAll\` for all payees in RoyaltiesReceiverV2
* \#57445 \[SC-Medium] Signature Replay with Mutable Parameters
* \#57850 \[SC-Medium] By transferring his staking shares to another non staking address, allowing him to bypass \`minStakePeriod\`
* \#57427 \[SC-Medium] Mint signatures are not bound to a collection which makes cross-collection replay possible under a shared signer
* \#57927 \[SC-Medium] Front‑Run Takeover in Factory.produce
* \#57905 \[SC-Medium] Signature Malleability and Replay Attack Vulnerabilities in Signature Verification
* \#57615 \[SC-Medium] Permanent Freezing of User Assets in \`Staking.sol\`
* \#57727 \[SC-Medium] Venues with autostake LONG PaymentType can be griefed and cause permanent freeze of LONG token
* \#57691 \[SC-Medium] Malicious Referrer can permanently block ETH payment flow
* \#57610 \[SC-Medium] Venues can steal from customers by replaying payments via \`BelongCheckIn::payToVenue\`
* \#56826 \[SC-Medium] attacker can bloat a victim’s stakes (array) and cause withdrawals/emergency flows to run out of gas
* \#57634 \[SC-Medium] Unauthorized minting of NFTs due to Signature Replay
* \#57437 \[SC-Medium] Front-running in Factory#produce()

</details>

<details>

<summary>Low</summary>

* \#57374 \[SC-Low] Staking Tier Misclassification
* \#57454 \[SC-Low] Referral percentages schedule stuck on first configuration
* \#57799 \[SC-Low] Retroactive Lock Period Changes Affect Existing Stakes
* \#57884 \[SC-Low] Staking Tier Manipulation via ERC4626 Shares (sLONG)
* \#57872 \[SC-Low] Processing fee computed on full LONG amount instead of subsidy in payToVenue, underpaying venues and enabling LONG-payment DoS under misconfiguration
* \#57596 \[SC-Low] Reentrancy in \`distributePromoterPayments()\` allows total theft of promoter and venue funds
* \#57201 \[SC-Low] Missing Collection Expiry Enforcement
* \#57505 \[SC-Low] Missing collection expiration enforcement allows unauthorized minting
* \#57583 \[SC-Low] Promoter Bounty Bait-and-Switch via updateVenueRules
* \#57595 \[SC-Low] Single-Tier Swap Path Can Stall Core Flows
* \#57425 \[SC-Low] Referral percentage updates are ignored due to append-only storage in NFTFactory
* \#57307 \[SC-Low] Cairo factory referral percentages never update
* \#57558 \[SC-Low] Front-running Issue in \`emergencyCancelPayment\`
* \#57718 \[SC-Low] Staking Tier Error: Using ERC4626 shares rather than assets to determine staking tiers leads to long-term distortion in fees and commissions.
* \#57650 \[SC-Low] Wrapped Native Token Routing Can Fail Without Full Validation
* \#57255 \[SC-Low] Allowed minting of NFTs after collection expiry date
* \#57453 \[SC-Low] Attackers can drain user allowance provided to the BelongCheckIn.sol

</details>

<details>

<summary>Insight</summary>

* \#57432 \[SC-Insight] RoyaltiesReceiverV2 Fails to Distribute Full Balance When Royalties Percentages Do Not Sum to 10000
* \#57663 \[SC-Insight] \[GAS] Storage Optimization: \`ERC1155Info\` Struct in \`Structures.sol\` Can Save One Slot Through Field Reordering
* \#57892 \[SC-Insight] Long tokens will be stuck in the escrow if customers exclusively use usdc payments in payToVenue
* \#57735 \[SC-Insight] Whitelist bypass in static mint pricing: trusting signed \`params.whitelisted\` instead of on-chain \`isWhitelisted\` leads to underpricing and access control violation.
* \#57701 \[SC-Insight] AccessToken.collectionExpire is never checked, allowing tokens to be minted even after the collection expires.
* \#57776 \[SC-Insight] Staking.sol is not EIP4626 Compliant Breaking Integrations
* \#57838 \[SC-Insight] Missing Produce Name Sanitization Allows Breaking SNIP-12 Standard Compliance
* \#57910 \[SC-Insight] Missing Validation on Referral Percentage Sum
* \#57804 \[SC-Insight] Unbounded percentages cause underflow and dos in mint payment flow
* \#57902 \[SC-Insight] ERC1155Base re-mint overwrites token URI, allowing post-issuance NFT alteration / griefing
* \#57874 \[SC-Insight] Global metadata wipe on \`burn\`: one promoter’s payout clears the shared ERC1155 token URI for all promoters of the same venue
* \#57467 \[SC-Insight] Unlimited \`referrals\[hashedCode].referralUsers\` increases gas cost with each new referral, making it very expensive.
* \#57810 \[SC-Insight] \[GAS/OPTIMIZATION] Use calldata for External Struct Parameters in \`checkAccessTokenInfo\` or \`checkCustomerInfo\` or \` checkPromoterPaymentDistribution\`
* \#57882 \[SC-Insight] Venue Tokens cannot be withdrawn when there are no promoters involved in customers transactions
* \#57268 \[SC-Insight] ERC1155Base: Missing Collection URI Fallback Causes Significant Gas Waste on Every Token Mint
* \#57656 \[SC-Insight] Incorrect supply-cap check - uses \`token\_id\` instead of \`total\_supply\` - in \`\_base\_mint\`
* \#57913 \[SC-Insight] Missing Validation in \`setParameters\` Allows Invalid Fee Configuration Causing Reverts in \`payToVenue\`
* \#57348 \[SC-Insight] Incorrectly returned values and emitted data on \`Staking\` emergency functionality
* \#57134 \[SC-Insight] AccessToken.sol is not ERC721 compliant
* \#57803 \[SC-Insight] \[GAS] Optimize \`PaymentsInfo\` Struct Layout to Save Storage Slots and Reduce Gas Costs
* \#57931 \[SC-Insight] Consumes more gas than intended in \`getStandardizedPrice\` function in Helper library
* \#57921 \[SC-Insight] Whitelisted role cannot be revoked in \`nft.cairo\`

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#57423 \[SC-Medium] Unbounded Gas Consumption in Emergency Redemption Enables Low-Cost DoS Against Staking Vault Users
* \#57039 \[SC-Critical] Processing Fee Logic Flaw in payToVenue() Causes Permanent Loss of Platform Revenue
* \#57060 \[SC-Medium] Unconditional subsidy withdrawal in \`payToVenue()\` leads to DoS when venue’s LONG pool is depleted
* \#57285 \[SC-Medium] Incomplete Signature in Factory.produce() Enables Full AccessToken Hijacking and Direct Fund Theft
* \#57296 \[SC-High] Retroactive referral tier underpayment in RoyaltiesReceiverV2 due to dynamic shares applied to historical funds
* \#57298 \[SC-Critical] State-sync omission in \`Staking\` transfers forces transferred sLONG holders into penalized emergency exits
* \#57310 \[SC-Medium] Unaccounted Processing Fees in LONG Payment Path
* \#57432 \[SC-Insight] RoyaltiesReceiverV2 Fails to Distribute Full Balance When Royalties Percentages Do Not Sum to 10000
* \#57327 \[SC-Medium] Title: Front-running Leads to Denial of Service and Unauthorized Referral Farming in Creation Functions.
* \#57290 \[SC-High] MEV Sandwich Attack Vulnerability - No User-Controlled Slippage Protection in Token Swaps
* \#57374 \[SC-Low] Staking Tier Misclassification
* \#57454 \[SC-Low] Referral percentages schedule stuck on first configuration
* \#57635 \[SC-Critical] ERC4626 share transfers desynchronize time-lock ledger, blocking standard withdrawals for recipients
* \#57594 \[SC-Medium] Signature-collision from abi.encodePacked (adjacent strings) enables unauthorized NFT actions (mint/URI abuse)
* \#57885 \[SC-High] Dynamic Share Drift in RoyaltiesReceiverV2
* \#57888 \[SC-High] Referral tier upgrades freeze legacy royalties
* \#57898 \[SC-High] Unprotected Swap Function Allows Sandwich Attacks
* \#57663 \[SC-Insight] \[GAS] Storage Optimization: \`ERC1155Info\` Struct in \`Structures.sol\` Can Save One Slot Through Field Reordering
* \#57892 \[SC-Insight] Long tokens will be stuck in the escrow if customers exclusively use usdc payments in payToVenue
* \#57703 \[SC-Medium] DoS with Revert via Unbounded Loop
* \#57284 \[SC-Medium] Updating minimum staking period griefs previously unlocked users
* \#57929 \[SC-Medium] \`\_produce\` function doesn't check if creator is the caller allowing frontrunning attacks
* \#57723 \[SC-Medium] Signature Replay, Front‑Run, and Timing Control Issues
* \#57724 \[SC-Medium] Universal signature for \`produce()\` allows front-running and collection hijack
* \#57735 \[SC-Insight] Whitelist bypass in static mint pricing: trusting signed \`params.whitelisted\` instead of on-chain \`isWhitelisted\` leads to underpricing and access control violation.
* \#57738 \[SC-Medium] Name-squatting / front-run on produce() allows attacker to preempt legitimate creator and capture future mint revenue
* \#56941 \[SC-Critical] Staking vault vulnerable to first-depositor donation attack
* \#57061 \[SC-High] Retroactive Share Recalculation Causes Royalty Distribution Failure
* \#57628 \[SC-Critical] Improper Transfer can lead to funds been frozen
* \#57775 \[SC-Medium] PayToVenue will revert due to NotEnoughLONGs funds in the Escrow contract
* \#57015 \[SC-Medium] Unbounded Array Loop
* \#57701 \[SC-Insight] AccessToken.collectionExpire is never checked, allowing tokens to be minted even after the collection expires.
* \#57796 \[SC-Medium] Signature hashing collision in SignatureVerifier lets attacker deploy forged AccessToken/CreditToken metadata (Critical — Unintended alteration of what the NFT represents)
* \#57799 \[SC-Low] Retroactive Lock Period Changes Affect Existing Stakes
* \#57800 \[SC-Medium] Signature Replay Vulnerability in BelongCheckIn::distributePromoterPayments
* \#57809 \[SC-Critical] inflation of shares in staking contract
* \#57813 \[SC-Critical] Transfer recipients will pay unwarranted emergency withdrawal penalties for share positions they legitimately own
* \#56860 \[SC-Medium] Hash collision in signature verification
* \#57519 \[SC-Medium] Unbounded Stake Array Allows permanent withdraw lock via Dust Deposits on Behalf of Victims
* \#57221 \[SC-High] Incorrect Processing Fee Calculation Causes Venue Payouts to Be Misallocated
* \#57399 \[SC-Critical] ERC4626 staking lockbook breaks share fungibility - partial transfers can DoS withdrawals
* \#56850 \[SC-Critical] Donation attack posible on Staking.sol because its totalAsset() uses asset.balanceOf()
* \#56907 \[SC-Critical] Attacker can steal first depositor's asset with inflation attack
* \#56810 \[SC-Medium] AccessToken: Cross‑contract signature replay allows unauthorized minting on other collections
* \#57877 \[SC-High] AccessToken creators can bypass fees so that platform address will receive 0 fees.
* \#57398 \[SC-Critical] Incorrect Platform Subsidy Processing in LONG Payments Causing Venue Payout Failures
* \#57829 \[SC-High] Incorrect Fee Implementation in payToVenue LONG Payment Path Causes Protocol Fees to be Permanently Locked in Escrow
* \#57203 \[SC-Medium] \[REVISED] Malicious AccessToken creator can steal gas via \`mintStaticPrice\` or \`mintDynamicPrice\`
* \#57940 \[SC-Medium] Deterministic address collision in Cairo deployment causes DoS and unintended receiver sharing
* \#57426 \[SC-Medium] Dynamic Price Signature Replay Allows Unlimited Minting at Historical Prices
* \#57482 \[SC-Critical] Front-running a donation can inflate the share causing users to lose funds
* \#57712 \[SC-Medium] Receiver Deployment DoS via Salt Reuse
* \#57314 \[SC-Medium] Signature Replay and Hash Collision via \`abi.encodePacked\` in SignatureVerifier.sol
* \#57716 \[SC-Critical] ERC4626 Inflation bug in Staking contract
* \#57271 \[SC-Medium] Incorrect penalty calculation on emergency withdrawals/redemption's
* \#57485 \[SC-Medium] emergencyWithdraw cost more penalty than expected
* \#56896 \[SC-Critical] staking contract is vulnerable to inflation attack making malicious 1st staker Grief the following stakers
* \#57669 \[SC-Medium] Stakers will bypass minStakePeriod time locks and extract rewards without commitment through emergency withdrawal mechanism
* \#57884 \[SC-Low] Staking Tier Manipulation via ERC4626 Shares (sLONG)
* \#57236 \[SC-Medium] AccessToken Collection Front-Running Attack - Permanent Ownership Hijack
* \#56867 \[SC-Medium] Signature Collision caused counterfeit AccessToken collections with arbitrary name/symbol/URI
* \#57283 \[SC-Medium] Unauthorised Promoter Payouts due to Signature Replay Attack.
* \#57717 \[SC-Medium] Attacker can spam tiny stakes to a victim and make their withdrawal run out of gas (griefing DoS)
* \#57872 \[SC-Low] Processing fee computed on full LONG amount instead of subsidy in payToVenue, underpaying venues and enabling LONG-payment DoS under misconfiguration
* \#57358 \[SC-Medium] Unlimited Stake Entries Allow Account Griefing via Tiny Deposits
* \#57891 \[SC-Medium] Signature replay lets attackers hijack NFT collection deployment
* \#57766 \[SC-Medium] attacker can permanently lock any user’s funds
* \#57848 \[SC-Medium] Permanent Freezing of funds due to no minimum stake limit
* \#57596 \[SC-Low] Reentrancy in \`distributePromoterPayments()\` allows total theft of promoter and venue funds
* \#57201 \[SC-Low] Missing Collection Expiry Enforcement
* \#57671 \[SC-High] RoyaltiesReceiverV2.shares() referralShare uses dynamic values, which may result in failure to release funds properly.
* \#57939 \[SC-Medium] Signature Collision via abi.encodePacked
* \#57279 \[SC-Medium] Signature replayability — repeated use of signed access tokens allows duplicate mints (High)
* \#57505 \[SC-Low] Missing collection expiration enforcement allows unauthorized minting
* \#57580 \[SC-Medium] Signature Replay Enables Frontrunning of produce/produceCreditToken
* \#57676 \[SC-High] Cross-token accounting in Receiver allows permanent freezing of ERC20 royalty payouts
* \#57452 \[SC-High] On-chain Quoter reliance and spot-price-based swaps enable pool manipulation and value extraction from protocol-controlled conversions (USDC ↔ LONG).
* \#57776 \[SC-Insight] Staking.sol is not EIP4626 Compliant Breaking Integrations
* \#57838 \[SC-Insight] Missing Produce Name Sanitization Allows Breaking SNIP-12 Standard Compliance
* \#57910 \[SC-Insight] Missing Validation on Referral Percentage Sum
* \#57854 \[SC-Medium] Front-Running Attack Allows Collection Ownership Theft
* \#57804 \[SC-Insight] Unbounded percentages cause underflow and dos in mint payment flow
* \#57583 \[SC-Low] Promoter Bounty Bait-and-Switch via updateVenueRules
* \#56863 \[SC-Critical] First Depositor Advantage
* \#57373 \[SC-Medium] Signature Replay Vulnerability Due to Missing Nonce and Deadline Checks
* \#57902 \[SC-Insight] ERC1155Base re-mint overwrites token URI, allowing post-issuance NFT alteration / griefing
* \#57864 \[SC-Medium] abi.encodePacked Hash Collision Vulnerability in Dynamic Type Encoding Permits Malicious Signature Bypass, Enabling Unauthorized and Repeatable Transaction Execution
* \#56869 \[SC-Medium] Hijacking deployment of \`AccessToken\` and stealing ownership to prevent further deployments
* \#56814 \[SC-Medium] Users Can Create unauthorized AccessToken collections by exploiting \`abi.encodePacked\` collision
* \#57874 \[SC-Insight] Global metadata wipe on \`burn\`: one promoter’s payout clears the shared ERC1155 token URI for all promoters of the same venue
* \#57875 \[SC-Medium] Signature bypass lets creators alter key AccessToken parameters before deployment
* \#57245 \[SC-Medium] Needless iterations in for loops should be removed for better optimization and code maintenance
* \#57677 \[SC-Medium] Signature Replay in venueDeposit Enables Affiliate Referral Code Hijacking Leading to Unauthorized Commission Theft
* \#57008 \[SC-Critical] emergencyWithdraw Function Malfunction Due to Missing Validation in \_removeAnySharesFor
* \#57595 \[SC-Low] Single-Tier Swap Path Can Stall Core Flows
* \#57425 \[SC-Low] Referral percentage updates are ignored due to append-only storage in NFTFactory
* \#57307 \[SC-Low] Cairo factory referral percentages never update
* \#57467 \[SC-Insight] Unlimited \`referrals\[hashedCode].referralUsers\` increases gas cost with each new referral, making it very expensive.
* \#57558 \[SC-Low] Front-running Issue in \`emergencyCancelPayment\`
* \#57089 \[SC-Medium] Unauthorized collection hijack via unsigned creator
* \#57917 \[SC-Medium] Penallty can be bypassed in \`Staking.sol::\_emergencyWithdraw\`
* \#57718 \[SC-Low] Staking Tier Error: Using ERC4626 shares rather than assets to determine staking tiers leads to long-term distortion in fees and commissions.
* \#57810 \[SC-Insight] \[GAS/OPTIMIZATION] Use calldata for External Struct Parameters in \`checkAccessTokenInfo\` or \`checkCustomerInfo\` or \` checkPromoterPaymentDistribution\`
* \#57435 \[SC-High] Missing Slippage Protection Enables Direct Theft via MEV Sandwich Attacks
* \#57932 \[SC-Critical] Attacker can bypass stake lock
* \#57911 \[SC-Medium] Signature are malleable in SignatureVerifier.sol.
* \#57401 \[SC-Critical] ERC4626 Inflation Attack Vulnerability
* \#57882 \[SC-Insight] Venue Tokens cannot be withdrawn when there are no promoters involved in customers transactions
* \#57388 \[SC-Medium] Cross contract signature replay because verifying contract is not included in the digest
* \#57650 \[SC-Low] Wrapped Native Token Routing Can Fail Without Full Validation
* \#57255 \[SC-Low] Allowed minting of NFTs after collection expiry date
* \#57895 \[SC-Medium] Lack of msg.sender Validation in Collection Creation Signature Enables Front-Running Attack Leading to Creator Impersonation
* \#57362 \[SC-Medium] Attacker can DoS user withdraw in staking contract
* \#57268 \[SC-Insight] ERC1155Base: Missing Collection URI Fallback Causes Significant Gas Waste on Every Token Mint
* \#57586 \[SC-High] Calculating slippage for swap onchain does not prevent slippage loss
* \#56841 \[SC-High] Sudden addition of rewards will be frontrun with deposits just to steal part of reward
* \#57515 \[SC-High] Cross‑token accounting is broken
* \#57656 \[SC-Insight] Incorrect supply-cap check - uses \`token\_id\` instead of \`total\_supply\` - in \`\_base\_mint\`
* \#57913 \[SC-Insight] Missing Validation in \`setParameters\` Allows Invalid Fee Configuration Causing Reverts in \`payToVenue\`
* \#57685 \[SC-Critical] Vulnerabilities in the design of the token's staking mechanism resulted in financial harm to users involved in transfer-related operations.
* \#57194 \[SC-Medium] Signature replay across collections (missing contract binding)
* \#57348 \[SC-Insight] Incorrectly returned values and emitted data on \`Staking\` emergency functionality
* \#57938 \[SC-Medium] \`\_produce\` function doesn't check if creator is the caller allowing frontrunning attacks
* \#57733 \[SC-High] swapExact's slippge is not works as expected
* \#57736 \[SC-Critical] First depositor attack is possible
* \#57702 \[SC-Medium] The long payment path is sensitive to the long inventory in Escrow, and insufficient inventory can easily lead to business unavailability (DoS of long payments).
* \#56881 \[SC-High] Temporary Claim Freezing
* \#57790 \[SC-Medium] Withdrawal Denial-of-Service via Dust Stake Spam
* \#56872 \[SC-Critical] Freezing of funds
* \#57237 \[SC-High] Cross-token math contaminates payouts in \`Receiver\`
* \#57458 \[SC-Medium] DoS/Griefing in batch ETH payout: malicious payee \`receive()\` can revert and block \`releaseAll\` for all payees in RoyaltiesReceiverV2
* \#57445 \[SC-Medium] Signature Replay with Mutable Parameters
* \#57850 \[SC-Medium] By transferring his staking shares to another non staking address, allowing him to bypass \`minStakePeriod\`
* \#57134 \[SC-Insight] AccessToken.sol is not ERC721 compliant
* \#57427 \[SC-Medium] Mint signatures are not bound to a collection which makes cross-collection replay possible under a shared signer
* \#57927 \[SC-Medium] Front‑Run Takeover in Factory.produce
* \#57905 \[SC-Medium] Signature Malleability and Replay Attack Vulnerabilities in Signature Verification
* \#57786 \[SC-High] Malicious Users can frontrun Staking::distributeRewards to claim majority of rewards
* \#57076 \[SC-High] Incorrect slippage would result in swap manipulations
* \#57615 \[SC-Medium] Permanent Freezing of User Assets in \`Staking.sol\`
* \#57727 \[SC-Medium] Venues with autostake LONG PaymentType can be griefed and cause permanent freeze of LONG token
* \#57691 \[SC-Medium] Malicious Referrer can permanently block ETH payment flow
* \#57610 \[SC-Medium] Venues can steal from customers by replaying payments via \`BelongCheckIn::payToVenue\`
* \#56826 \[SC-Medium] attacker can bloat a victim’s stakes (array) and cause withdrawals/emergency flows to run out of gas
* \#57942 \[SC-Critical] Transferred \`sLONG\` Shares Are Permanently Unredeemable Due to Missing Stake Entry Creation
* \#57634 \[SC-Medium] Unauthorized minting of NFTs due to Signature Replay
* \#57803 \[SC-Insight] \[GAS] Optimize \`PaymentsInfo\` Struct Layout to Save Storage Slots and Reduce Gas Costs
* \#57453 \[SC-Low] Attackers can drain user allowance provided to the BelongCheckIn.sol
* \#57924 \[SC-Critical] The staking contract is suceptible to the classic first depositor exploit
* \#57437 \[SC-Medium] Front-running in Factory#produce()
* \#57931 \[SC-Insight] Consumes more gas than intended in \`getStandardizedPrice\` function in Helper library
* \#57921 \[SC-Insight] Whitelisted role cannot be revoked in \`nft.cairo\`

</details>
