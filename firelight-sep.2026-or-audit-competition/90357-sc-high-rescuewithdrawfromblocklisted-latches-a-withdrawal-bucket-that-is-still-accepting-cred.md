# 90357 sc high rescuewithdrawfromblocklisted latches a withdrawal bucket that is still accepting credits freezing the account s next withdrawal permanently

**Submitted on Aug 24th 2026 at 16:30:54 UTC by @Shchepetov for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90357
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

`isWithdrawClaimed[period][account]` is a one shot boolean guarding a balance that keeps accumulating. `rescueWithdrawFromBlocklisted` sets that boolean on a period bucket that is still open for new withdrawal requests, and nothing in the contract ever clears it. Any withdrawal the same account books into that period afterwards burns its shares into a bucket it can never claim from. One such sequence freezes a holder's entire position.

## Vulnerability details

The finding is in `contracts/core/FirelightVault.sol` at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, the upgrade implementation for the deployed predeposit vault proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`, which holds 56,302,995.498517 FXRP at Flare block 68095000.

Two functions write `isWithdrawClaimed`, and no function anywhere clears it: `claimWithdraw` at line 806, and `rescueWithdrawFromBlocklisted` at line 870.

For `claimWithdraw` the flag is load bearing: that function leaves `withdrawSharesOf` intact and uses the flag at line 794 as its only spent marker. For the rescue it is dead weight: line 869 has already zeroed `withdrawSharesOf[periods[i]][from]`, so both a second rescue and a claim would revert `NoWithdrawalAmount` (lines 866 and 803) without it.

The balance it latches is not one shot. `_requestWithdraw` books into `period = currentPeriod() + 1` (line 1008) with `withdrawSharesOf[period][receiver] += sharesWithdraw` (line 1017), so bucket `p` keeps accepting credits for the whole of period `p - 1`, which is exactly when a rescue on bucket `p` is performed.

The sequence runs as follows. The account requests a withdrawal into bucket `p`; BLOCKLIST\_ROLE calls `addToBlocklist` (line 550); RESCUER\_ROLE calls `rescueWithdrawFromBlocklisted`, moving the pending withdrawal to a safe address and latching bucket `p` for the account; BLOCKLIST\_ROLE calls `removeFromBlocklist` (line 560), the documented inverse of the blocklisting; the account then makes an ordinary `withdraw` call, still inside the same period, so the credit lands in bucket `p` again and its shares are burned at line 1025. `claimWithdraw(p)` reverts `AlreadyClaimedPeriod` from then on. Re-blocklisting and rescuing again reverts on the same flag at line 864, and `rescueSharesFromBlocklisted` reverts `InsufficientShares` at line 828 because the shares are already burned, so neither rescue path can undo it.

The frozen assets also stay in `pendingWithdrawAssets` (line 1019) forever, and `totalAssets()` (line 428) subtracts that value, so the vault's reported capital is understated by the same amount.

## Impact

Permanent freezing of funds. The loss is realised by the account itself, through the permissionless `withdraw`.

The contract states the constraint this breaks, at lines 747 to 749: "Initiates a withdrawal request from the vault... The request becomes claimable starting from the period after the next full period. The calculated shares are burned." Shares are burned in exchange for a claim that becomes payable. In the latched case the shares are still burned at line 1025 and the claim becomes payable never.

The rescue is the reason. Line 862 reads the pending balance once, lines 868 and 869 move exactly that amount to `to`, and then line 870 latches the whole bucket, which keeps accreting for the rest of the period. So the only value the latch ever reaches is value the rescue was never handed, and it is neither paid to `to` nor left with `from`: it leaves the accounting entirely, while `totalAssets` at line 428 subtracts it from the vault's reported capital for good.

The shipped PoC measures one sequence against the real shareholder `0x558C734506EaA244d8313908c0060EdeA42Fbd26` on live vault state: 8,989,000.1496 FXRP frozen, its `balanceOf` driven to 0. That is the holder's entire position. With the latch write at line 870 removed and nothing else changed, the identical sequence pays the account 8,989,000.1496 FXRP and freezes 0.

There are two bounds. The un-blocklisting and the account's next request must fall inside the same period, 86,400 seconds on the live configuration, which `periodConfigurations(0).duration` returns; a later request lands in an unlatched bucket. And nothing clears the flag: none of the contract's thirteen role-gated functions touches it, and no amount of waiting lifts it.

## Recommendation

Delete line 870. It is redundant for the rescue path, since the zeroed balance already makes both a second rescue and a claim revert `NoWithdrawalAmount`, and it leaves the rescue's own effect untouched: the counterfactual phase shows the rescued withdrawal still sitting with the destination address after the line is neutralised. A guard in `_requestWithdraw` rejecting a credit into a latched bucket also closes it, at the cost of turning the silent loss into a hard revert of the account's whole withdrawal for that period.

## Link to Proof of Concept

<https://gist.github.com/Shchepetov/b0678aaaeeeaaa63edc6b9cdbc93f1e6>

## Proof of Concept

## Proof of Concept

Hardhat test, one test function, on a local Flare mainnet fork pinned at block 68095000. It asserts `contractVersion() == 1`, upgrades the real proxy to a freshly compiled in-scope `FirelightVault` through the real ProxyAdmin, then drives the sequence with the real role holders and a real shareholder. The counterfactual reverts to the same fork state, clears only the latched storage word, and re-runs it.

Gist: <https://gist.github.com/Shchepetov/b0678aaaeeeaaa63edc6b9cdbc93f1e6>
