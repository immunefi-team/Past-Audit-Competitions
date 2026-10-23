# 26502 - \[SC - Insight] DeGate Exodus mode forcing study

Submitted on Dec 4th 2023 at 05:09:53 UTC by @Merkle\_Bonsai for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26502

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x9C07A72177c5A05410cA338823e790876E79D73B#code>

Impacts:

* Study report

## Description

This report is made by Berg's proposal to share the study with DeGate.

***

Exodus/withdrawal mode of DeGate is specific state that can be permissionlessly toggled if Operator is not executing his responsibility for at least 15 days. Only code-based way to trigger it is to mess up with `forceWithdraw` function. It is maxxed-permissionless:

* any ETH address may request {address, tokenID} pair for any registered token via `forceWithdraw`.
* any ETH address - not only initial requester - may report to the contract that specific {address, tokenID} pair has timestamp associated too old via `notifyForcedRequestTooOld`

***

However, it is needed to be additionally noted that `S.pendingForcedWithdrawals[accountID][tokenID]` memory address location may be predicted, and since there is a memory area (`struct State: Token[] normalTokens;`) that is possible to be fully filled with values, the actual attack complexity on searching the `keccak256(h(accountID) . tokenID)` can be lowered from 2^256 to 2^224 (as 2^32 memory area can theoretically be fully filled with values). This attack is 100% impractical, especially with quite small economic win for attacker, so it’s just good to note that it is good that other structures are mappings, not arrays, and only small memory range is allocatable.

***

Every {address, tokenID} pair is processed via specific kind of transaction inside `WithdrawTransaction.sol` withdraw transaction processor. Transaction processor is executing `deposit`, `withdraw` and `account update` transactions only, as all other transactions are handled solely in ZK, which acts as a guarantee of honest interactions between users. It may not execute some transactions, but if he does, they are ZK-backed honest.

I cannot be sure about this assumption, but everything looks like for me that ZK engine is able to process `deposit`, `withdraw` and `account update` transactions in any order within block.

However, this does not affect the flow, as worst thing that can possibly happen is `forceWithdraw` in same block will be only partial, and this is what user accepts if he's crazy enough to deposit assets after making force withdrawal.

This is also backed by specific order of transactions processing, when all the `deposits` are done before `withdraw`, that makes emergent state “block processor is trying to withdraw more than is deposited” impossible - balance is positive in every moment of time. I have to note it as simple yet very smart design decision.

Since token registration is permissionless, any token can be used for `deposits` and `withdraws`, and `submitBlocks` may be broken via e.g. deposit of contract like this:

```
contract ERC20 {
    function balanceOf(...) public { revert(); }
}
```

However, this will only possibly delay the block processing, as operator is able to re-pack the block by throwing away bad transactions - both in forked chain simulator and live: the whole block will be reverted, making it unable to get in impossible state.

On other hand, only way to prevent Exodus/Withdrawal mode after request record is created is inside the block processing.

Block processing can be possibly broken in 2 areas: Solidity block processor and ZK circuit.

Both areas should be directly related to specific withdrawal transaction, as otherwise it can be thrown away from block by Operator without any consequences impactful for protocol.

ZK circuit transaction can be made non-executable by multiple ways:

* impossible equation: division by zero or making `uint248` variable store more than it can
* failing `require`s

I wasn’t able to discover any ways to broke the circuit. What has been tried as input combinations:

* existing/non-existing account/max possible account (maxUint32) as sometimes borderline values can be impactful
* existing/non-existing token (non-existing means before any deposit happens, so ZK is not aware about it, only registration in contract happened)
* withdrawal type 2/3
* full/zero/non-matching amount

All scenarios are working fine, as ZK is able to process not yet registered users and tokens sanely. Only scenario that was not tested was system accounts processing (as far as I know, at least accounts 1 and 2, as account 0 is not triggerable via `forceWithdraw`), that is private state, as this calls may have unexpected behavior.

Since modification of ZK circuit is probably the last thing DeGate would want to do, and I see no issues that can be impactful for the protocol, I will abstain from recommendations like "`isProtocolFeeWithdrawal(pb, state.constants._1, state.constants._0, FMT(prefix, ".isProtocolFeeWithdrawal"))` is redundant and may be simplified" and focus on Solidity part.

Solidity part is also generally secure. Since `distributeWithdrawal` is calling `transferTokens` that is able to process even failing transactions, it is impossible to break withdrawal process.

Operator is theoretically able to run a birthday paradox attack like described in famous 0x52 report: <https://github.com/sherlock-audit/2023-07-kyber-swap-judging/issues/90>, but currently it is too expensive and may only allow operator to steal assets in several years from now. However, as the second preimage collision is searched here in case of Operator as an attacker, assumption “especially when combined with validUntil” is invalid - and nobody but Operator may be an attacker in this scenario, as Operator is only one who can access this function call.

Additionally, Operator is able to mess up with transactions a bit. `maxFee` and `validUntil` are passed via `auxData` , so Operator can theoretically stash withdrawal operation and execute it later, but this has no practical impact, as `validUntil` is protected on ZK level (so ZK checks that `validUntil` is correct for inner timestamp). ZK is only guaranteed to be synced within +-7 days, so `validUntil` can be not very accurate both on ZK level and aux data level, but it does not seem that Operator can get any significant win from doing it.

I initially assumed that it is possible to withdraw everything in shutdown mode because of presence of this check:

```
if (withdrawal.withdrawalType == 2) {
                    require(withdrawal.from == forcedWithdrawal.owner, "INCONSISENT_OWNER");
                } else { //withdrawal.withdrawalType == 3
                    require(withdrawal.from != forcedWithdrawal.owner, "INCONSISENT_OWNER");
                    >> require(withdrawal.amount == 0, "UNAUTHORIZED_WITHDRAWAL");
                }
```

However, ZK circuit does not allow this behavior (tx type 3 and amount != 0), so now I consider `require(withdrawal.amount == 0, "UNAUTHORIZED_WITHDRAWAL")` check as implemented to make the extra layer of protection (e.g. against sha256 collision attack) and make contract look more secure. This check is basically redundant, as ZK provides guarantees, but extra gas amount is quite small and this check can make positive reputational impact, so keeping it looks reasonable.

No possible states or operations that can cause non-mitigatable revertion of `forceWithdraw` transactions are present, but some of security layers look more accidential rather than intentional. E.g. some possible attacks are covered via making all calls `nonReentrant` - it does not look like there was some security model to have a reason to make everything `nonReentrant` on functions like `setWithdrawalRecipient`, and this looks more like over-protection approach, which is also sane thing. Only issue with over-protection is that some security actions taken may be considered irrelevant later and removed - if you do not know the threat model, you may accidentally think that actual protection is useless and remove it.

Specifically, I need to make a accent on `mapping(uint32 => uint248) tokenIdToDepositBalance`. Looks like it is general practice to use `uint248` for all asset-related variables, and this acts as additional security measure. I would recommend to keep an eye on it, because if some misalignment in `uint` size will appear, consequences would be impactful.

E.g. if any specific account can hold up to `maxUint248`, but total deposited value is larger than `maxUint248`(as ZK is not verifying the total deposited amount, only per-account), which is currently protected by `tokenIdToDepositBalance`, exodus mode can be triggered by sequentially withdrawing multiple users to one specific target account some broken asset, that will make `S.amountWithdrawable[to][tokenID] = S.amountWithdrawable[to][tokenID].add(amount)` overflow (as `amountWithdrawable` stores `uint248`) at some moment of time, making specific `forceWithdraw` request impossible to execute, allowing attacker to make contract enter the exodus mode.

However, `tokenIdToDepositBalance` usage looks solid; any flow that causes deposit or withdraw is processed before the withdraw may happen, ensuring that this edge case may not appear:

* It is only reduced in `ExchangeWithdrawals.transferTokens` after the actual withdrawal happens, not after tokens are considered as “left ZK”, but are still on account
* It is specifically zero-initialized in `ExchangeTokens.registerToken` as additional security check
* It is incremented on actual `ExchangeDeposits.deposit`
* It is incremented on `DepositTransaction.process` for `deposit.depositType 1`

Increment seems slightly inconsistent, but reasons seems understandable and it does not look like it impacts security somehow.

As I'm not proficient enough I would say that research on this small area of DeGate was really interesting, as study on a mix of ZK and Solidity logic is significantly different and makes brains to think alternatively.

## Proof of concept

***
