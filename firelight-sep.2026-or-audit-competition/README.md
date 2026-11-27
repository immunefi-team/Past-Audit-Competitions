# Firelight Sep.2026 | Audit Competition

## Reports by Severity

<details>

<summary>High</summary>

* \#88572 \[SC-High] A same-block boundary withdrawal can mutate committed period-start backing and leave settled cover insolvent
* \#90497 \[SC-High] Stale withdrawal claim flag permanently freezes a new same-period withdrawal
* \#89303 \[SC-High] rescueWithdrawFromBlocklisted marks a still-open withdrawal bucket as claimed, leading to permanent freezing of funds
* \#90159 \[SC-High] Redundant state update in rescueWithdrawFromBlocklisted permanently freezes funds for unblocklisted users withdrawing again
* \#90426 \[SC-High] rescueWithdrawFromBlocklisted permanently freezes later same-period withdrawals for a reinstated account
* \#89719 \[SC-High] `rescueWithdrawFromBlocklisted` permanently poisons the `(period, account)` claim flag, unrecoverably freezing any later withdrawal into the same still-open bucket
* \#89530 \[SC-High] A boundary-block withdrawal rewrites committed collateral and makes settled cover unpayable
* \#89315 \[SC-High] Boundary-timestamp withdrawal corrupts the period payout cap, allowing a shareholder to evade slashing and permanently underpay an approved claim
* \#89319 \[SC-High] Exact-period-boundary withdrawal is included in the period-start exposure lookup and permanently underpays the payout receiver
* \#89555 \[SC-High] a staker who times a withdrawal to the exact period-start second halves the incident payout and keeps the difference
* \#88411 \[SC-High] Unresolved Previous-Period Liabilities Are Reused to Underwrite New Cover, Causing a Valid Claim to Receive Zero Payout
* \#88400 \[SC-High] Stale Period-Start Capacity Checkpoint Lets Commitments Ignore Previous-Period Payouts and Settle Fully Unbacked Cover
* \#89499 \[SC-High] Same-timestamp redemption creates competing withdrawal and cover liabilities against the same collateral
* \#88534 \[SC-High] Checkpoint accounting desync in the payout function leads to protocol insolvency: a withdrawal at the exact period start collapses the exposure cap
* \#90401 \[SC-High] Exact-boundary withdrawal overwrites opening exposure and underpays valid claims by 50% at launch leverage
* \#90411 \[SC-High] Same-timestamp checkpoint corruption in withdraw() causes theft through underpaid settled cover
* \#90415 \[SC-High] rescueWithdrawFromBlocklisted marks the period claimed so a later self-withdraw cannot be claimed
* \#89549 \[SC-High] rescue leaves a sticky claimed flag that permanently freezes a later same-period withdrawal
* \#89865 \[SC-High] Period-boundary withdrawal rewrites the period-start exposure checkpoint in FirelightVault, letting a staker evade incident slashing and permanently underpaying the incident payout
* \#90183 \[SC-High] Rescue tombstone permanently freezes newly funded same-bucket withdrawals after unblock
* \#89198 \[SC-High] Stale collateral snapshot in \_computeAvailableCapacity lets the protocol sell cover against funds it has already paid out, driving CAR to 0.17 against the enforced 1.2 floor
* \#88479 \[SC-High] Rescue flags a still-open withdrawal bucket, permanently freezing the victim's later withdrawals for that period
* \#88489 \[SC-High] Stale isWithdrawClaimed flag left by a rescue lets a later same-period re-withdrawal burn a user's shares into a slot no code path can honor — permanent freezing of funds
* \#90222 \[SC-High] Any staker can halve the protocol's maximum incident payout by landing a withdrawal on the exact period-start timestamp
* \#88725 \[SC-High] Rescued withdrawal bucket reuse permanently freezes newly withdrawn funds
* \#89976 \[SC-High] Period-boundary checkpoint overwrite in `FirelightVault` lets an unprivileged staker suppress approved incident payouts and deflate underwriting capacity
* \#88925 \[SC-High] # \[C-2] `CoverOrderAllocator` contract values collateral at a period-start snapshot that a mid-period payout never updates, so cover is sold, paid for and receipted against a vault ...
* \#89588 \[SC-High] Same-block checkpoint overwrite in `FirelightVault.payout` lets any depositor escape assessed loss exposure, leaving the protocol unable to honour cover it already sold
* \#89051 \[SC-High] Rescue marker collision in FirelightVault permanently strands a cleared account's later withdrawal
* \#88525 \[SC-High] Missing upper bound in addPeriodConfiguration leads to permanent freezing of withdrawals
* \#89122 \[SC-High] Same-block period-start withdrawal can lower the payout exposure cap after cover capacity is committed
* \#90067 \[SC-High] `rescueWithdrawFromBlocklisted` latches `isWithdrawClaimed` on a withdrawal bucket that is still accepting credits, permanently freezing every later withdrawal credited into it
* \#90066 \[SC-High] Function `rescueWithdrawFromBlocklisted` permanently freezes a rescued account's future withdrawals
* \#88741 \[SC-High] # \[C-1] `FirelightVault::rescueWithdrawFromBlocklisted()` permanently poisons a still-open withdrawal bucket, freezing a staker's funds with no recovery path
* \#89220 \[SC-High] Stale `isWithdrawClaimed` flag written by `rescueWithdrawFromBlocklisted` leads to permanent freezing of funds: capital deposited after the compliance action has ended is destroyed ...
* \#89224 \[SC-High] Inclusive boundary redemption lets a shareholder retain collateral and underpay a covered claim
* \#89272 \[SC-High] Stale rescue marker permanently freezes a later withdrawal
* \#89281 \[SC-High] Anyone can permanently destroy any staker's entire balance for 0.000001 fXRP
* \#88555 \[SC-High] A depositor withdrawing in the exact period-boundary block underpays an approved incident claim by their own withdrawal, while the vault still holds the money and counts it as payable
* \#88214 \[SC-High] Period `P+1` cover capacity is computed from collateral a pending period-`P` claim will consume
* \#88231 \[SC-High] Rescuing a pending withdrawal permanently poisons that bucket
* \#88237 \[SC-High] Previous-period payouts do not reduce current-period cover capacity, allowing undercollateralized cover to be sold
* \#88244 \[SC-High] Stale vault snapshot in \_computeAvailableCapacity lets the allocator underwrite cover against collateral an incident payout already spent, breaching the enforced solvency floor (pro...
* \#88254 \[SC-High] `rescueWithdrawFromBlocklisted` permanently poisons the `(period, account)` claim flag, causing unrecoverable freezing of any later withdrawal the account makes into the same still-...
* \#88570 \[SC-High] Period-boundary checkpoint overwrite creates unbacked cover and zeroes valid payouts
* \#88576 \[SC-High] `commitAllocation()` can underwrite cover against a period-start snapshot that a same-block withdrawal rewrites, letting LP collateral escape slashing
* \#88270 \[SC-High] rescueWithdrawFromBlocklisted permanently marks (period, from) as claimed, bricking future withdrawals for that period and stranding the funds in pendingWithdrawAssets
* \#88279 \[SC-High] Stale period-start collateral accounting lets prior-period payouts leave newly settled cover fully unbacked
* \#88281 \[SC-High] CoverOrderAllocator reuses collateral still backing previous-period claims, leaving new cover insolvent
* \#88585 \[SC-High] Same-block period-start withdrawal retroactively changes committed collateral and causes protocol insolvency
* \#88286 \[SC-High] `_requestWithdraw()` can reuse a withdrawal bucket already closed by rescue, permanently freezing fresh user funds
* \#88287 \[SC-High] Incorrect period-boundary checkpoint accounting in `FirelightVault.withdraw()` lets stakers evade incident slashing and causes protocol insolvency
* \#89401 \[SC-High] Withdrawal requested at the exact period start second lowers the payout cap but stays payable, so valid claims are underpaid
* \#88775 \[SC-High] Inclusive period-start checkpoint lets a boundary withdrawal escape slashing and underfund a fully collateralized cover claim
* \#88595 \[SC-High] Rescuing a not-yet-claimable withdrawal request permanently poisons the (period, account) pair, freezing any later request on that period
* \#89085 \[SC-High] `payout()`'s exposure-cap snapshot can be corrupted by an ordinary user's withdrawal request landing at a period boundary, causing incident claims to be permanently underpaid
* \#90080 \[SC-High] Period-start redemption appends a deflated exposure checkpoint and underpays valid cover claims
* \#88604 \[SC-High] Stale collateral basis in `_computeAvailableCapacity` lets a period underwrite cover below the protocol's own minimum backing
* \#88308 \[SC-High] `totalAssetsAt` resolves a period's opening assets inclusively, so a withdrawal in the boundary second redefines the ceiling on every claim against that period
* \#89090 \[SC-High] A period-C payout executed during C+1 is invisible to CoverOrderAllocator's period-start capacity snapshot, letting a full book of C+1 cover be sold and settled against a vault the ...
* \#88615 \[SC-High] Stale period-start capacity snapshot lets the allocator over-sell cover after a mid-period payout, causing protocol insolvency
* \#88618 \[SC-High] Assessment approval can execute a replacement loss schedule the Risk Consortium never reviewed
* \#88783 \[SC-High] Backrunning the period-start allocation commitment erases the capacity it was validated against, leaving sold cover unbacked and the vault leg of a valid claim paying zero
* \#89373 \[SC-High] Delayed previous period claims can leave current cover without collateral
* \#88321 \[SC-High] Stale period-start collateral snapshot lets the allocator sell cover against assets already paid out as claims
* \#90104 \[SC-High] Rescuing a blocklisted account's withdrawal permanently bricks that account's next withdrawal
* \#88968 \[SC-High] Cover capacity in CoverOrderAllocator is sized from a period-start snapshot that payouts cannot update, so cover is sold against collateral that has already been paid out
* \#88343 \[SC-High] A payout after the matcher's capacity read lets stale collateral back new cover and creates an unpaid claim
* \#89424 \[SC-High] A withdrawal at the exact period boundary permanently collapses the period's exposure snapshot, voiding paid-for cover and denying capacity
* \#90127 \[SC-High] The period-start collateral snapshot is still writable inside the boundary block, so cover is committed against one basis while the approved claim is capped by a lower one
* \#88353 \[SC-High] A vault exit mined in the first second of a period writes that period's own exposure snapshot, permanently short-paying an approved cover claim
* \#88420 \[SC-High] `rescueWithdrawFromBlocklisted` sets the same flag `claimWithdraw` reads as "already claimed" and never clears it, permanently freezing any later withdrawal the account makes into t...
* \#90131 \[SC-High] Boundary-time checkpoint rewrite lets settled-cover collateral escape payouts, causing insolvency
* \#89428 \[SC-High] Inclusive period-start snapshot in FirelightVault lets any liquidity provider cap incident payouts to near-zero and destroy 80% of a period's underwriting capacity
* \#89055 \[SC-High] Missing period check in `rescueWithdrawFromBlocklisted` marks a still-open period as claimed, causing permanent freezing of funds for any withdrawal later requested into that period
* \#88636 \[SC-High] Period-Boundary Withdrawal Overwrites totalAssets Exposure Checkpoint → Payout Cap Collapses to Zero and the Withdrawer Escapes the Slash
* \#88619 \[SC-High] Stale `isWithdrawClaimed` flag set by `rescueWithdrawFromBlocklisted` permanently freezes a later legitimate withdrawal by the same account, burning the user's shares with no recove...
* \#88638 \[SC-High] `_computeAvailableCapacity` values the first-loss buffer live and the vault at the period start, so the vault half of a payout stays in the capital-adequacy calculation for the rest...
* \#89071 \[SC-High] Withdrawal rescue sets isWithdrawClaimed on an open period bucket, making any later withdrawal into that bucket permanently unclaimable
* \#88565 \[SC-High] Capacity uses a period-start snapshot a payout cannot lower, so cover is sold against collateral already paid out
* \#88653 \[SC-High] Flag collision between rescueWithdrawFromBlocklisted and claimWithdraw leads to permanent freezing of user withdrawals
* \#88684 \[SC-High] Checkpoint overwrites lead to loss of legitimate payouts
* \#89014 \[SC-High] Exact-boundary checkpoint overwrite underpays approved cover claims
* \#88704 \[SC-High] Boundary-block redemption rewrites the collateral snapshot after capacity is committed, reducing a valid claim by exactly the redeemed amount
* \#88705 \[SC-High] Rescued account's later same-period withdrawal is accepted and burned behind a terminal claim flag, permanently freezing user funds
* \#88473 \[SC-High] mutable period start checkpoint causes protocol insolvency by underpaying approved cover claims
* \#88772 \[SC-High] Stale claimed flag after compliance rescue permanently freezes a later withdrawal credited to the same period
* \#89946 \[SC-High] Period-start withdrawal corrupts collateral accounting, allowing settled cover to pay zero while the staker reclaims slashable assets
* \#89908 \[SC-High] Mutable period-start exposure snapshot in `FirelightVault.payout()` leads to protocol insolvency (approved incident claims underpaid)
* \#89958 \[SC-High] Rescuing an unmatured withdrawal lets fresh post-unblock shares be burned into a permanently closed period
* \#88753 \[SC-High] Boundary Snapshot Lets Withdrawals Escape Claims
* \#89854 \[SC-High] Same-block withdrawal rewrites committed period-start assets and makes valid cover payouts zero
* \#89856 \[SC-High] Rescue path permanently freezes a user's funds by marking a future period as already claimed
* \#89777 \[SC-High] Valid target-period withdrawal becomes unclaimable after rescue and unblock
* \#89803 \[SC-High] A withdrawal at the exact period start rewrites the historical asset snapshot, so approved incident claims are underpaid or paid nothing while the vault still holds the money
* \#88345 \[SC-High] Inclusive period-boundary checkpoint lets a staker underfund valid claims
* \#89793 \[SC-High] `rescueWithdrawFromBlocklisted` marks a not-yet-elapsed period as claimed, permanently locking an unrelated future withdrawal by the same account
* \#88688 \[SC-High] CoverOrderAllocator validates capacity against a stale period-start snapshot, letting cover settle after its collateral backing is paid out and causing protocol insolvency
* \#89811 \[SC-High] A completed compliance rescue permanently poisons the source's withdrawal slot and freezes fresh principal after unblocking
* \#88395 \[SC-High] Claim-state poisoning in blocklist rescue permanently freezes a later withdrawal
* \#88338 \[SC-High] Stale vault capacity lets Firelight sell cover against collateral owed to prior claims
* \#90086 \[SC-High] Inclusive period-boundary checkpoint substitution in FirelightVault.withdraw() lets a withdrawing shareholder retain funds owed to an approved claimant
* \#90488 \[SC-High] Exact-Boundary Withdrawal Retroactively Shrinks the Period Snapshot After
* \#90484 \[SC-High] Mutable period-start exposure snapshot in FirelightVault lets a redeeming LP retain slashable assets and underpays a valid cover claim
* \#89714 \[SC-High] A rescued account can permanently freeze fresh principal in an already-claimed withdrawal bucket
* \#90493 \[SC-High] Incorrect claim-flag update in the `rescueWithdrawFromBlocklisted` function leads to permanent freezing of restored users' funds
* \#89590 \[SC-High] Missing terminal-key guard in FirelightVault permanently freezes withdrawals after same-period rescue and restoration
* \#90429 \[SC-High] A period-start withdrawal can remove collateral already committed to cover, causing a zero vault payout on a valid claim
* \#88651 \[SC-High] Inclusive period-start checkpoint lets an exact-boundary withdrawal erase incident payout exposure and later reclaim the pending funds
* \#90357 \[SC-High] `rescueWithdrawFromBlocklisted` latches a withdrawal bucket that is still accepting credits, freezing the account's next withdrawal permanently
* \#89929 \[SC-High] Inclusive period-start lookup lets exact-boundary withdrawals reduce incident payouts and potentially cause protocol insolvency
* \#89579 \[SC-High] rescueWithdrawFromBlocklisted flags a still-open period as claimed without paying anything out, permanently bricking the rescued user's next legitimate withdrawal into that same period
* \#90013 \[SC-High] Permanent freezing of user funds due to state corruption in `rescueWithdrawFromBlocklisted` when resolving compliance false positives
* \#90105 \[SC-High] A routine, non-malicious blocklist rescue permanently poisons the rescued account's *next* withdrawal, with no recovery path for any role
* \#90203 \[SC-High] Missing period bound in rescueWithdrawFromBlocklisted flags an unstarted period as claimed, permanently freezing the account's next withdrawal
* \#90087 \[SC-High] Same-timestamp checkpoint overwrite in FirelightVault lets a permissionless boundary withdrawal underpay a valid same-period payout
* \#89234 \[SC-High] rescueWithdrawFromBlocklisted tombstones an open bucket, permanently locking the user's next withdrawal
* \#88992 \[SC-High] Protocol Insolvency and LP Fund Drain via Stale Underwriting Capacity Calculation Following Cross-Period Payouts
* \#88712 \[SC-High] Mutable period-start snapshot lets LPs remove collateral after it has been committed to cover
* \#88410 \[SC-High] Rescue terminalizes a still-open withdrawal bucket and permanently freezes post-unblock principal
* \#88425 \[SC-High] Rescue marks an open withdrawal bucket claimed and permanently freezes later post-unblock principal
* \#88597 \[SC-High] Same-boundary withdrawal mutates committed collateral and causes protocol insolvency
* \#88553 \[SC-High] CoverOrderAllocator uses stale collateral after payouts

</details>

<details>

<summary>Medium</summary>

* \#90282 \[SC-Medium] Once `payout()` drives `totalAssets()` to \~0 with shares still outstanding, share pricing degenerates: any address can mint `(totalSupply+1)` shares per wei of deposit, capturing ...
* \#89511 \[SC-Medium] Mutable period-start checkpoint lets a CoverNFT callback grief payouts for already-settled cover
* \#89659 \[SC-Medium] Exact period-boundary deposits and withdrawals corrupt the opening-assets checkpoint
* \#89556 \[SC-Medium] A boundary withdrawal can zero the capture-period exposure cap and prevent incident payouts
* \#89500 \[SC-Medium] A deposit placed right after a payout drains the vault mints shares against a near-zero `totalAssets()` — the premium income pre-incident LPs underwrote for goes to whoever deposi...
* \#89677 \[SC-Medium] `rescueWithdrawFromBlocklisted` can pre-claim a live withdrawal bucket and make later withdrawals unclaimable
* \#90409 \[SC-Medium] A slash collapses the share price to near zero, and deposits reopen in the same transaction, letting anyone mint the vault's entire future value for dust
* \#90175 \[SC-Medium] Exact-period-start withdrawal redefines opening exposure and lets an LP evade a valid slash
* \#88347 \[SC-Medium] Rescue writes a terminal claimed-flag onto a still-open withdrawal bucket, permanently trapping the account's next withdrawal
* \#90014 \[SC-Medium] Exact-timestamp withdrawals at period boundaries permanently corrupt total asset snapshots, truncating claim payouts and DoS'ing cover underwriting
* \#90025 \[SC-Medium] A dust deposit after a total-loss payout captures approximately 99% of all future vault inflows
* \#88745 \[SC-Medium] # \[H-1] `FirelightVault::deposit()` mints unbounded shares against a zero-asset pool, letting a 1 FXRP deposit capture the entire period's premium from the stakers who underwrote it
* \#89130 \[SC-Medium] Inclusive period-start snapshot in FirelightVault lets a same-second redeem shrink the payout cap and sold-cover capacity, leading to griefing of cover claimants
* \#89847 \[SC-Medium] Unclearable claim flag in `rescueWithdrawFromBlocklisted` causes permanent freezing of staker withdrawals
* \#90093 \[SC-Medium] A boundary withdrawal changes the period-start exposure lookup after capacity is committed, causing an approved claim to be underpaid, up to zero
* \#88620 \[SC-Medium] Cover capacity in `_computeAvailableCapacity` is sized on a stale period-start asset snapshot, so a prior-period payout lets the allocator commit cover against collateral that has...
* \#90102 \[SC-Medium] Post-slash deposit accounting in FirelightVault leads to theft of recovered staker principal
* \#88330 \[SC-Medium] Post-slash zero-assets state inverts the ERC4626 virtual offset: a 1-wei deposit mints \~half of all shares and captures \~50% of all future vault yield
* \#88337 \[SC-Medium] Cover capacity is read from a period-start snapshot, so cover committed after an incident payout is priced against collateral the payout already spent
* \#88631 \[SC-Medium] FirelightVault: mint() mints shares at \~zero price after an incident payout drains the vault, breaking the ERC4626 share-price invariant and enabling capture of the recovery inflow
* \#88366 \[SC-Medium] Safe signatures for one assessment can trigger 900 additional units of unauthorized depletion
* \#88839 \[SC-Medium] Exact-boundary checkpoint overwrite suppresses payout exposure
* \#88416 \[SC-Medium] Post-payout share-price collapse in `FirelightVault.payout` lets an unprivileged 100-wei deposit capture \~99% of the slashed LPs' subsequent premium, leading to theft of unclaimed...
* \#89238 \[SC-Medium] Unblocking after rescue can permanently freeze same-period withdrawals
* \#89222 \[SC-Medium] A withdrawal at the exact period-start second halves the cover payout
* \#89834 \[SC-Medium] An attacker can grief `commitAllocation` when the `matchingCapacity` is calculated at `block.timestamp = periodStart timestamp`
* \#89904 \[SC-Medium] rescueWithdrawFromBlocklisted marks a still-future withdrawal period as claimed, permanently locking any withdrawal the rescued account later makes into that period
* \#90033 \[SC-Medium] Missing share burn in payout() allows a 1-wei deposit to clone the entire LP supply and steal half of all subsequent vault yield
* \#88477 \[SC-Medium] A rescued (period, user) slot remains writable after being marked claimed, so a later withdrawal burns the user's shares into a position no one can ever claim
* \#89114 \[SC-Medium] Permanent denial-of-service of FirelightVault.deposit() via multiplicative share inflation once totalAssets() reaches zero through a legitimate payout()

</details>

<details>

<summary>Low</summary>

* \#89546 \[SC-Low] \[FirelightVault] redeem() lacks the zero-output guard present in deposit(), burning shares for zero assets when totalAssets < totalSupply
* \#90295 \[SC-Low] maxMint() can return an unmintable share amount, causing mint() to revert and violating ERC-4626
* \#89813 \[SC-Low] `maxMint` overstates the executable cap after a direct asset transfer changes the asset-to-share ratio
* \#89812 \[SC-Low] `maxDeposit` advertises positive capacity that no `deposit` call can execute after a direct asset transfer
* \#90313 \[SC-Low] `FirelightVault` can return positive ERC-4626 entry maxima that cannot be executed
* \#90328 \[SC-Low] Period-open totalAssetsAt(P\_start) is overwritten by any vault flow in the boundary second, mis-sizing the payout cap and matching capacity
* \#88551 \[SC-Low] The capacity guard does not bound vault quantity drift
* \#88234 \[SC-Low] `maxMint` and `maxDeposit` advertise amounts that revert (EIP-4626 violation)
* \#88269 \[SC-Low] Missing `assets == 0` guard in FirelightVault.redeem() burns shares for zero assets when share price < 1 (unfixed variant of Macro L-1)
* \#88292 \[SC-Low] maxMint() can overstate remaining mint capacity and return an amount that mint() immediately rejects
* \#88960 \[SC-Low] `redeem()` \`can burn shares while crediting zero assets
* \#88295 \[SC-Low] `maxMint()` overstates the mintable amount and `mint(maxMint(x), x)` can revert — ERC-4626 conformance break near the deposit cap
* \#88596 \[SC-Low] FirelightVault.maxMint() returns a share count that mint() rejects, and maxDeposit() an asset amount that deposit() rejects
* \#90085 \[SC-Low] The withdraw side of the share/asset conversion has no zero-output guard, so `redeem()` can burn a holder's shares while crediting nothing claimable
* \#88313 \[SC-Low] `maxMint` floors through `convertToShares` while `mint` prices with `Ceil`, so the advertised maximum can exceed the deposit limit by the rounding residue
* \#89230 \[SC-Low] Exact-boundary withdrawal can zero an approved payout after cover capacity is committed
* \#88785 \[SC-Low] FirelightVault.redeem() burns the caller's shares and queues nothing when the redeemed asset value floors to zero — the assets == 0 guard present on deposit() and withdraw() is missing
* \#88834 \[SC-Low] `maxMint` overstates the executable mint limit after exchange-rate changes
* \#88984 \[SC-Low] Missing zero-value check in redeem() silently burns shares for zero assets after totalAssets() reaches zero
* \#88676 \[SC-Low] FirelightVault.redeem() burns a staker's shares for zero assets, records no withdrawal credit, and returns success
* \#89486 \[SC-Low] Incorrect rounding in maxMint() causes mint(maxMint()) to revert with DepositLimitExceeded, violating ERC-4626 spec
* \#89825 \[SC-Low] Missing assets==0 guard in FirelightVault.redeem() burns a user's shares for zero withdrawal credit when the share price is below 1
* \#89876 \[SC-Low] Rounding-inconsistent `maxDeposit` and `maxMint` values can advertise vault entry that necessarily reverts
* \#89290 \[SC-Low] `maxDeposit()` can advertise a positive amount that `deposit()` deterministically rejects after legitimate rewards
* \#90489 \[SC-Low] FirelightVault.maxMint() returns a share count larger than mint() will accept, violating EIP-4626's "MUST NOT be higher than the actual maximum" — the standard mint(maxMint(u), u) in...
* \#88835 \[SC-Low] `maxDeposit` can advertise an amount that `deposit` always rejects
* \#90153 \[SC-Low] CR-01 fix bypass: maxMint() converts the absolute cap instead of remaining capacity, causing mint(maxMint) to revert after rewards
* \#90310 \[SC-Low] `maxMint()` can return an amount that reverts when passed to `mint()`
* \#88973 \[SC-Low] FirelightVault.maxMint() returns a share amount that mint() reverts on with DepositLimitExceeded, because the two functions round in opposite directions across depositLimit

</details>

<details>

<summary>Insight</summary>

* \#88963 \[SC-Insight] `CapacityConfig` field order wastes a full storage slot
* \#88930 \[SC-Insight] Multiple NatSpec inaccuracies across `ICoverOrderAllocator` and `IIncidentManager` produce misleading documentation for integrators

</details>

## Reports by Type

<details>

<summary>Smart Contract</summary>

* \#88572 \[SC-High] A same-block boundary withdrawal can mutate committed period-start backing and leave settled cover insolvent
* \#90282 \[SC-Medium] Once `payout()` drives `totalAssets()` to \~0 with shares still outstanding, share pricing degenerates: any address can mint `(totalSupply+1)` shares per wei of deposit, capturing ...
* \#89511 \[SC-Medium] Mutable period-start checkpoint lets a CoverNFT callback grief payouts for already-settled cover
* \#90497 \[SC-High] Stale withdrawal claim flag permanently freezes a new same-period withdrawal
* \#89303 \[SC-High] rescueWithdrawFromBlocklisted marks a still-open withdrawal bucket as claimed, leading to permanent freezing of funds
* \#90159 \[SC-High] Redundant state update in rescueWithdrawFromBlocklisted permanently freezes funds for unblocklisted users withdrawing again
* \#90426 \[SC-High] rescueWithdrawFromBlocklisted permanently freezes later same-period withdrawals for a reinstated account
* \#89719 \[SC-High] `rescueWithdrawFromBlocklisted` permanently poisons the `(period, account)` claim flag, unrecoverably freezing any later withdrawal into the same still-open bucket
* \#89530 \[SC-High] A boundary-block withdrawal rewrites committed collateral and makes settled cover unpayable
* \#89315 \[SC-High] Boundary-timestamp withdrawal corrupts the period payout cap, allowing a shareholder to evade slashing and permanently underpay an approved claim
* \#89319 \[SC-High] Exact-period-boundary withdrawal is included in the period-start exposure lookup and permanently underpays the payout receiver
* \#89546 \[SC-Low] \[FirelightVault] redeem() lacks the zero-output guard present in deposit(), burning shares for zero assets when totalAssets < totalSupply
* \#89659 \[SC-Medium] Exact period-boundary deposits and withdrawals corrupt the opening-assets checkpoint
* \#89555 \[SC-High] a staker who times a withdrawal to the exact period-start second halves the incident payout and keeps the difference
* \#89556 \[SC-Medium] A boundary withdrawal can zero the capture-period exposure cap and prevent incident payouts
* \#88411 \[SC-High] Unresolved Previous-Period Liabilities Are Reused to Underwrite New Cover, Causing a Valid Claim to Receive Zero Payout
* \#88400 \[SC-High] Stale Period-Start Capacity Checkpoint Lets Commitments Ignore Previous-Period Payouts and Settle Fully Unbacked Cover
* \#89499 \[SC-High] Same-timestamp redemption creates competing withdrawal and cover liabilities against the same collateral
* \#89500 \[SC-Medium] A deposit placed right after a payout drains the vault mints shares against a near-zero `totalAssets()` — the premium income pre-incident LPs underwrote for goes to whoever deposi...
* \#88534 \[SC-High] Checkpoint accounting desync in the payout function leads to protocol insolvency: a withdrawal at the exact period start collapses the exposure cap
* \#90295 \[SC-Low] maxMint() can return an unmintable share amount, causing mint() to revert and violating ERC-4626
* \#89677 \[SC-Medium] `rescueWithdrawFromBlocklisted` can pre-claim a live withdrawal bucket and make later withdrawals unclaimable
* \#90401 \[SC-High] Exact-boundary withdrawal overwrites opening exposure and underpays valid claims by 50% at launch leverage
* \#90409 \[SC-Medium] A slash collapses the share price to near zero, and deposits reopen in the same transaction, letting anyone mint the vault's entire future value for dust
* \#90411 \[SC-High] Same-timestamp checkpoint corruption in withdraw() causes theft through underpaid settled cover
* \#90415 \[SC-High] rescueWithdrawFromBlocklisted marks the period claimed so a later self-withdraw cannot be claimed
* \#89549 \[SC-High] rescue leaves a sticky claimed flag that permanently freezes a later same-period withdrawal
* \#89813 \[SC-Low] `maxMint` overstates the executable cap after a direct asset transfer changes the asset-to-share ratio
* \#89865 \[SC-High] Period-boundary withdrawal rewrites the period-start exposure checkpoint in FirelightVault, letting a staker evade incident slashing and permanently underpaying the incident payout
* \#90175 \[SC-Medium] Exact-period-start withdrawal redefines opening exposure and lets an LP evade a valid slash
* \#90183 \[SC-High] Rescue tombstone permanently freezes newly funded same-bucket withdrawals after unblock
* \#89198 \[SC-High] Stale collateral snapshot in \_computeAvailableCapacity lets the protocol sell cover against funds it has already paid out, driving CAR to 0.17 against the enforced 1.2 floor
* \#88479 \[SC-High] Rescue flags a still-open withdrawal bucket, permanently freezing the victim's later withdrawals for that period
* \#89812 \[SC-Low] `maxDeposit` advertises positive capacity that no `deposit` call can execute after a direct asset transfer
* \#90313 \[SC-Low] `FirelightVault` can return positive ERC-4626 entry maxima that cannot be executed
* \#88489 \[SC-High] Stale isWithdrawClaimed flag left by a rescue lets a later same-period re-withdrawal burn a user's shares into a slot no code path can honor — permanent freezing of funds
* \#88347 \[SC-Medium] Rescue writes a terminal claimed-flag onto a still-open withdrawal bucket, permanently trapping the account's next withdrawal
* \#90222 \[SC-High] Any staker can halve the protocol's maximum incident payout by landing a withdrawal on the exact period-start timestamp
* \#88725 \[SC-High] Rescued withdrawal bucket reuse permanently freezes newly withdrawn funds
* \#89976 \[SC-High] Period-boundary checkpoint overwrite in `FirelightVault` lets an unprivileged staker suppress approved incident payouts and deflate underwriting capacity
* \#88925 \[SC-High] # \[C-2] `CoverOrderAllocator` contract values collateral at a period-start snapshot that a mid-period payout never updates, so cover is sold, paid for and receipted against a vault ...
* \#89588 \[SC-High] Same-block checkpoint overwrite in `FirelightVault.payout` lets any depositor escape assessed loss exposure, leaving the protocol unable to honour cover it already sold
* \#90014 \[SC-Medium] Exact-timestamp withdrawals at period boundaries permanently corrupt total asset snapshots, truncating claim payouts and DoS'ing cover underwriting
* \#90025 \[SC-Medium] A dust deposit after a total-loss payout captures approximately 99% of all future vault inflows
* \#89051 \[SC-High] Rescue marker collision in FirelightVault permanently strands a cleared account's later withdrawal
* \#88525 \[SC-High] Missing upper bound in addPeriodConfiguration leads to permanent freezing of withdrawals
* \#89122 \[SC-High] Same-block period-start withdrawal can lower the payout exposure cap after cover capacity is committed
* \#90067 \[SC-High] `rescueWithdrawFromBlocklisted` latches `isWithdrawClaimed` on a withdrawal bucket that is still accepting credits, permanently freezing every later withdrawal credited into it
* \#90066 \[SC-High] Function `rescueWithdrawFromBlocklisted` permanently freezes a rescued account's future withdrawals
* \#88741 \[SC-High] # \[C-1] `FirelightVault::rescueWithdrawFromBlocklisted()` permanently poisons a still-open withdrawal bucket, freezing a staker's funds with no recovery path
* \#88963 \[SC-Insight] `CapacityConfig` field order wastes a full storage slot
* \#89220 \[SC-High] Stale `isWithdrawClaimed` flag written by `rescueWithdrawFromBlocklisted` leads to permanent freezing of funds: capital deposited after the compliance action has ended is destroyed ...
* \#89224 \[SC-High] Inclusive boundary redemption lets a shareholder retain collateral and underpay a covered claim
* \#90328 \[SC-Low] Period-open totalAssetsAt(P\_start) is overwritten by any vault flow in the boundary second, mis-sizing the payout cap and matching capacity
* \#88745 \[SC-Medium] # \[H-1] `FirelightVault::deposit()` mints unbounded shares against a zero-asset pool, letting a 1 FXRP deposit capture the entire period's premium from the stakers who underwrote it
* \#89272 \[SC-High] Stale rescue marker permanently freezes a later withdrawal
* \#88551 \[SC-Low] The capacity guard does not bound vault quantity drift
* \#89281 \[SC-High] Anyone can permanently destroy any staker's entire balance for 0.000001 fXRP
* \#89130 \[SC-Medium] Inclusive period-start snapshot in FirelightVault lets a same-second redeem shrink the payout cap and sold-cover capacity, leading to griefing of cover claimants
* \#88555 \[SC-High] A depositor withdrawing in the exact period-boundary block underpays an approved incident claim by their own withdrawal, while the vault still holds the money and counts it as payable
* \#88930 \[SC-Insight] Multiple NatSpec inaccuracies across `ICoverOrderAllocator` and `IIncidentManager` produce misleading documentation for integrators
* \#88214 \[SC-High] Period `P+1` cover capacity is computed from collateral a pending period-`P` claim will consume
* \#88231 \[SC-High] Rescuing a pending withdrawal permanently poisons that bucket
* \#88234 \[SC-Low] `maxMint` and `maxDeposit` advertise amounts that revert (EIP-4626 violation)
* \#88237 \[SC-High] Previous-period payouts do not reduce current-period cover capacity, allowing undercollateralized cover to be sold
* \#88244 \[SC-High] Stale vault snapshot in \_computeAvailableCapacity lets the allocator underwrite cover against collateral an incident payout already spent, breaching the enforced solvency floor (pro...
* \#88254 \[SC-High] `rescueWithdrawFromBlocklisted` permanently poisons the `(period, account)` claim flag, causing unrecoverable freezing of any later withdrawal the account makes into the same still-...
* \#88570 \[SC-High] Period-boundary checkpoint overwrite creates unbacked cover and zeroes valid payouts
* \#88576 \[SC-High] `commitAllocation()` can underwrite cover against a period-start snapshot that a same-block withdrawal rewrites, letting LP collateral escape slashing
* \#89847 \[SC-Medium] Unclearable claim flag in `rescueWithdrawFromBlocklisted` causes permanent freezing of staker withdrawals
* \#88269 \[SC-Low] Missing `assets == 0` guard in FirelightVault.redeem() burns shares for zero assets when share price < 1 (unfixed variant of Macro L-1)
* \#88270 \[SC-High] rescueWithdrawFromBlocklisted permanently marks (period, from) as claimed, bricking future withdrawals for that period and stranding the funds in pendingWithdrawAssets
* \#88279 \[SC-High] Stale period-start collateral accounting lets prior-period payouts leave newly settled cover fully unbacked
* \#88281 \[SC-High] CoverOrderAllocator reuses collateral still backing previous-period claims, leaving new cover insolvent
* \#88585 \[SC-High] Same-block period-start withdrawal retroactively changes committed collateral and causes protocol insolvency
* \#88286 \[SC-High] `_requestWithdraw()` can reuse a withdrawal bucket already closed by rescue, permanently freezing fresh user funds
* \#88287 \[SC-High] Incorrect period-boundary checkpoint accounting in `FirelightVault.withdraw()` lets stakers evade incident slashing and causes protocol insolvency
* \#88292 \[SC-Low] maxMint() can overstate remaining mint capacity and return an amount that mint() immediately rejects
* \#88960 \[SC-Low] `redeem()` \`can burn shares while crediting zero assets
* \#88295 \[SC-Low] `maxMint()` overstates the mintable amount and `mint(maxMint(x), x)` can revert — ERC-4626 conformance break near the deposit cap
* \#89401 \[SC-High] Withdrawal requested at the exact period start second lowers the payout cap but stays payable, so valid claims are underpaid
* \#88775 \[SC-High] Inclusive period-start checkpoint lets a boundary withdrawal escape slashing and underfund a fully collateralized cover claim
* \#88595 \[SC-High] Rescuing a not-yet-claimable withdrawal request permanently poisons the (period, account) pair, freezing any later request on that period
* \#88596 \[SC-Low] FirelightVault.maxMint() returns a share count that mint() rejects, and maxDeposit() an asset amount that deposit() rejects
* \#89085 \[SC-High] `payout()`'s exposure-cap snapshot can be corrupted by an ordinary user's withdrawal request landing at a period boundary, causing incident claims to be permanently underpaid
* \#90080 \[SC-High] Period-start redemption appends a deflated exposure checkpoint and underpays valid cover claims
* \#90085 \[SC-Low] The withdraw side of the share/asset conversion has no zero-output guard, so `redeem()` can burn a holder's shares while crediting nothing claimable
* \#88604 \[SC-High] Stale collateral basis in `_computeAvailableCapacity` lets a period underwrite cover below the protocol's own minimum backing
* \#88308 \[SC-High] `totalAssetsAt` resolves a period's opening assets inclusively, so a withdrawal in the boundary second redefines the ceiling on every claim against that period
* \#90093 \[SC-Medium] A boundary withdrawal changes the period-start exposure lookup after capacity is committed, causing an approved claim to be underpaid, up to zero
* \#88313 \[SC-Low] `maxMint` floors through `convertToShares` while `mint` prices with `Ceil`, so the advertised maximum can exceed the deposit limit by the rounding residue
* \#89090 \[SC-High] A period-C payout executed during C+1 is invisible to CoverOrderAllocator's period-start capacity snapshot, letting a full book of C+1 cover be sold and settled against a vault the ...
* \#88615 \[SC-High] Stale period-start capacity snapshot lets the allocator over-sell cover after a mid-period payout, causing protocol insolvency
* \#88618 \[SC-High] Assessment approval can execute a replacement loss schedule the Risk Consortium never reviewed
* \#88783 \[SC-High] Backrunning the period-start allocation commitment erases the capacity it was validated against, leaving sold cover unbacked and the vault leg of a valid claim paying zero
* \#88620 \[SC-Medium] Cover capacity in `_computeAvailableCapacity` is sized on a stale period-start asset snapshot, so a prior-period payout lets the allocator commit cover against collateral that has...
* \#89373 \[SC-High] Delayed previous period claims can leave current cover without collateral
* \#88321 \[SC-High] Stale period-start collateral snapshot lets the allocator sell cover against assets already paid out as claims
* \#90102 \[SC-Medium] Post-slash deposit accounting in FirelightVault leads to theft of recovered staker principal
* \#90104 \[SC-High] Rescuing a blocklisted account's withdrawal permanently bricks that account's next withdrawal
* \#88330 \[SC-Medium] Post-slash zero-assets state inverts the ERC4626 virtual offset: a 1-wei deposit mints \~half of all shares and captures \~50% of all future vault yield
* \#88968 \[SC-High] Cover capacity in CoverOrderAllocator is sized from a period-start snapshot that payouts cannot update, so cover is sold against collateral that has already been paid out
* \#89230 \[SC-Low] Exact-boundary withdrawal can zero an approved payout after cover capacity is committed
* \#88785 \[SC-Low] FirelightVault.redeem() burns the caller's shares and queues nothing when the redeemed asset value floors to zero — the assets == 0 guard present on deposit() and withdraw() is missing
* \#88337 \[SC-Medium] Cover capacity is read from a period-start snapshot, so cover committed after an incident payout is priced against collateral the payout already spent
* \#88343 \[SC-High] A payout after the matcher's capacity read lets stale collateral back new cover and creates an unpaid claim
* \#88631 \[SC-Medium] FirelightVault: mint() mints shares at \~zero price after an incident payout drains the vault, breaking the ERC4626 share-price invariant and enabling capture of the recovery inflow
* \#89424 \[SC-High] A withdrawal at the exact period boundary permanently collapses the period's exposure snapshot, voiding paid-for cover and denying capacity
* \#90127 \[SC-High] The period-start collateral snapshot is still writable inside the boundary block, so cover is committed against one basis while the approved claim is capped by a lower one
* \#88353 \[SC-High] A vault exit mined in the first second of a period writes that period's own exposure snapshot, permanently short-paying an approved cover claim
* \#88420 \[SC-High] `rescueWithdrawFromBlocklisted` sets the same flag `claimWithdraw` reads as "already claimed" and never clears it, permanently freezing any later withdrawal the account makes into t...
* \#90131 \[SC-High] Boundary-time checkpoint rewrite lets settled-cover collateral escape payouts, causing insolvency
* \#89428 \[SC-High] Inclusive period-start snapshot in FirelightVault lets any liquidity provider cap incident payouts to near-zero and destroy 80% of a period's underwriting capacity
* \#89055 \[SC-High] Missing period check in `rescueWithdrawFromBlocklisted` marks a still-open period as claimed, causing permanent freezing of funds for any withdrawal later requested into that period
* \#88366 \[SC-Medium] Safe signatures for one assessment can trigger 900 additional units of unauthorized depletion
* \#88636 \[SC-High] Period-Boundary Withdrawal Overwrites totalAssets Exposure Checkpoint → Payout Cap Collapses to Zero and the Withdrawer Escapes the Slash
* \#88619 \[SC-High] Stale `isWithdrawClaimed` flag set by `rescueWithdrawFromBlocklisted` permanently freezes a later legitimate withdrawal by the same account, burning the user's shares with no recove...
* \#88638 \[SC-High] `_computeAvailableCapacity` values the first-loss buffer live and the vault at the period start, so the vault half of a payout stays in the capital-adequacy calculation for the rest...
* \#89071 \[SC-High] Withdrawal rescue sets isWithdrawClaimed on an open period bucket, making any later withdrawal into that bucket permanently unclaimable
* \#88565 \[SC-High] Capacity uses a period-start snapshot a payout cannot lower, so cover is sold against collateral already paid out
* \#88839 \[SC-Medium] Exact-boundary checkpoint overwrite suppresses payout exposure
* \#88834 \[SC-Low] `maxMint` overstates the executable mint limit after exchange-rate changes
* \#88653 \[SC-High] Flag collision between rescueWithdrawFromBlocklisted and claimWithdraw leads to permanent freezing of user withdrawals
* \#88984 \[SC-Low] Missing zero-value check in redeem() silently burns shares for zero assets after totalAssets() reaches zero
* \#88676 \[SC-Low] FirelightVault.redeem() burns a staker's shares for zero assets, records no withdrawal credit, and returns success
* \#88416 \[SC-Medium] Post-payout share-price collapse in `FirelightVault.payout` lets an unprivileged 100-wei deposit capture \~99% of the slashed LPs' subsequent premium, leading to theft of unclaimed...
* \#88684 \[SC-High] Checkpoint overwrites lead to loss of legitimate payouts
* \#89014 \[SC-High] Exact-boundary checkpoint overwrite underpays approved cover claims
* \#89238 \[SC-Medium] Unblocking after rescue can permanently freeze same-period withdrawals
* \#88704 \[SC-High] Boundary-block redemption rewrites the collateral snapshot after capacity is committed, reducing a valid claim by exactly the redeemed amount
* \#88705 \[SC-High] Rescued account's later same-period withdrawal is accepted and burned behind a terminal claim flag, permanently freezing user funds
* \#88473 \[SC-High] mutable period start checkpoint causes protocol insolvency by underpaying approved cover claims
* \#88772 \[SC-High] Stale claimed flag after compliance rescue permanently freezes a later withdrawal credited to the same period
* \#89486 \[SC-Low] Incorrect rounding in maxMint() causes mint(maxMint()) to revert with DepositLimitExceeded, violating ERC-4626 spec
* \#89946 \[SC-High] Period-start withdrawal corrupts collateral accounting, allowing settled cover to pay zero while the staker reclaims slashable assets
* \#89908 \[SC-High] Mutable period-start exposure snapshot in `FirelightVault.payout()` leads to protocol insolvency (approved incident claims underpaid)
* \#89958 \[SC-High] Rescuing an unmatured withdrawal lets fresh post-unblock shares be burned into a permanently closed period
* \#88753 \[SC-High] Boundary Snapshot Lets Withdrawals Escape Claims
* \#89854 \[SC-High] Same-block withdrawal rewrites committed period-start assets and makes valid cover payouts zero
* \#89856 \[SC-High] Rescue path permanently freezes a user's funds by marking a future period as already claimed
* \#89777 \[SC-High] Valid target-period withdrawal becomes unclaimable after rescue and unblock
* \#89803 \[SC-High] A withdrawal at the exact period start rewrites the historical asset snapshot, so approved incident claims are underpaid or paid nothing while the vault still holds the money
* \#89825 \[SC-Low] Missing assets==0 guard in FirelightVault.redeem() burns a user's shares for zero withdrawal credit when the share price is below 1
* \#89876 \[SC-Low] Rounding-inconsistent `maxDeposit` and `maxMint` values can advertise vault entry that necessarily reverts
* \#89290 \[SC-Low] `maxDeposit()` can advertise a positive amount that `deposit()` deterministically rejects after legitimate rewards
* \#88345 \[SC-High] Inclusive period-boundary checkpoint lets a staker underfund valid claims
* \#89793 \[SC-High] `rescueWithdrawFromBlocklisted` marks a not-yet-elapsed period as claimed, permanently locking an unrelated future withdrawal by the same account
* \#88688 \[SC-High] CoverOrderAllocator validates capacity against a stale period-start snapshot, letting cover settle after its collateral backing is paid out and causing protocol insolvency
* \#89811 \[SC-High] A completed compliance rescue permanently poisons the source's withdrawal slot and freezes fresh principal after unblocking
* \#88395 \[SC-High] Claim-state poisoning in blocklist rescue permanently freezes a later withdrawal
* \#88338 \[SC-High] Stale vault capacity lets Firelight sell cover against collateral owed to prior claims
* \#90086 \[SC-High] Inclusive period-boundary checkpoint substitution in FirelightVault.withdraw() lets a withdrawing shareholder retain funds owed to an approved claimant
* \#89222 \[SC-Medium] A withdrawal at the exact period-start second halves the cover payout
* \#90488 \[SC-High] Exact-Boundary Withdrawal Retroactively Shrinks the Period Snapshot After
* \#90484 \[SC-High] Mutable period-start exposure snapshot in FirelightVault lets a redeeming LP retain slashable assets and underpays a valid cover claim
* \#89834 \[SC-Medium] An attacker can grief `commitAllocation` when the `matchingCapacity` is calculated at `block.timestamp = periodStart timestamp`
* \#90489 \[SC-Low] FirelightVault.maxMint() returns a share count larger than mint() will accept, violating EIP-4626's "MUST NOT be higher than the actual maximum" — the standard mint(maxMint(u), u) in...
* \#89714 \[SC-High] A rescued account can permanently freeze fresh principal in an already-claimed withdrawal bucket
* \#90493 \[SC-High] Incorrect claim-flag update in the `rescueWithdrawFromBlocklisted` function leads to permanent freezing of restored users' funds
* \#89590 \[SC-High] Missing terminal-key guard in FirelightVault permanently freezes withdrawals after same-period rescue and restoration
* \#88835 \[SC-Low] `maxDeposit` can advertise an amount that `deposit` always rejects
* \#90429 \[SC-High] A period-start withdrawal can remove collateral already committed to cover, causing a zero vault payout on a valid claim
* \#88651 \[SC-High] Inclusive period-start checkpoint lets an exact-boundary withdrawal erase incident payout exposure and later reclaim the pending funds
* \#90357 \[SC-High] `rescueWithdrawFromBlocklisted` latches a withdrawal bucket that is still accepting credits, freezing the account's next withdrawal permanently
* \#89929 \[SC-High] Inclusive period-start lookup lets exact-boundary withdrawals reduce incident payouts and potentially cause protocol insolvency
* \#89579 \[SC-High] rescueWithdrawFromBlocklisted flags a still-open period as claimed without paying anything out, permanently bricking the rescued user's next legitimate withdrawal into that same period
* \#89904 \[SC-Medium] rescueWithdrawFromBlocklisted marks a still-future withdrawal period as claimed, permanently locking any withdrawal the rescued account later makes into that period
* \#90013 \[SC-High] Permanent freezing of user funds due to state corruption in `rescueWithdrawFromBlocklisted` when resolving compliance false positives
* \#90105 \[SC-High] A routine, non-malicious blocklist rescue permanently poisons the rescued account's *next* withdrawal, with no recovery path for any role
* \#90153 \[SC-Low] CR-01 fix bypass: maxMint() converts the absolute cap instead of remaining capacity, causing mint(maxMint) to revert after rewards
* \#90203 \[SC-High] Missing period bound in rescueWithdrawFromBlocklisted flags an unstarted period as claimed, permanently freezing the account's next withdrawal
* \#90310 \[SC-Low] `maxMint()` can return an amount that reverts when passed to `mint()`
* \#90033 \[SC-Medium] Missing share burn in payout() allows a 1-wei deposit to clone the entire LP supply and steal half of all subsequent vault yield
* \#90087 \[SC-High] Same-timestamp checkpoint overwrite in FirelightVault lets a permissionless boundary withdrawal underpay a valid same-period payout
* \#89234 \[SC-High] rescueWithdrawFromBlocklisted tombstones an open bucket, permanently locking the user's next withdrawal
* \#88992 \[SC-High] Protocol Insolvency and LP Fund Drain via Stale Underwriting Capacity Calculation Following Cross-Period Payouts
* \#88712 \[SC-High] Mutable period-start snapshot lets LPs remove collateral after it has been committed to cover
* \#88410 \[SC-High] Rescue terminalizes a still-open withdrawal bucket and permanently freezes post-unblock principal
* \#88425 \[SC-High] Rescue marks an open withdrawal bucket claimed and permanently freezes later post-unblock principal
* \#88477 \[SC-Medium] A rescued (period, user) slot remains writable after being marked claimed, so a later withdrawal burns the user's shares into a position no one can ever claim
* \#88597 \[SC-High] Same-boundary withdrawal mutates committed collateral and causes protocol insolvency
* \#88973 \[SC-Low] FirelightVault.maxMint() returns a share amount that mint() reverts on with DepositLimitExceeded, because the two functions round in opposite directions across depositLimit
* \#88553 \[SC-High] CoverOrderAllocator uses stale collateral after payouts
* \#89114 \[SC-Medium] Permanent denial-of-service of FirelightVault.deposit() via multiplicative share inflation once totalAssets() reaches zero through a legitimate payout()

</details>
