# \#99463 [SC-Low] LOW-40 fix incomplete: a 1-atom requestRedeem naming another controller hides its funded USDG from redeem / withdraw

**Submitted on Nov 13th 2026 at 19:08:41 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99463
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
The fix of Cecuro LOW-39 / LOW-40 (commit d757d0b1, PR #123) states its purpose in the vault's interface and in the audit comments:

- `IAsyncTrancheVault.sol:12-14`: "Deposit claims and redemption cancellations/refunds require receiver == controller, including zero-share acknowledgements, so share delivery cannot impose another account's cooldown or depend on an unsolicited receiver balance."
- LOW-40 comment: "A controller therefore cannot use these paths to send unsolicited shares and impose a cooldown on an unrelated empty address."
- LOW-39 comment: "Eligibility no longer depends on the receiver's share balance, so unsolicited dust cannot disable the supported self-delivery path."
- The project's own tests: `test_ThirdPartyRedeemZeroCannotResetCooldown`, `test_ThirdPartyWithdrawZeroCannotResetCooldown` ("Zero redeem must not reset cooldown", "Alice should retain request capacity"), `test_ControllerClaimSurvivesUnsolicitedShareDust`.
- `CANONICAL_ENTRYPOINTS.md:191-192`: "A holder calls `requestRedeem(shares, controller, owner)`; funded requests are exposed by `pendingRedeemRequest` / `claimableRedeemRequest` and claimed through ERC-4626 `redeem` or `withdraw`."

The fix restricted the **receiver** of claims, cancellations and refunds. The **controller** argument of `requestRedeem` was left as it was: the function authorises only the share `owner`, and appends the request to the claim queue of whatever controller the caller names.

```solidity
// TrancheVault.sol:606-632 (requestRedeem): the only authorisation is for `owner`
if (msg.sender != owner && !isOperator[owner][msg.sender]) {
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);
requestId = _enqueueRedeemRequest(shares, controller);
// TrancheVault.sol:701-704 (_enqueueRedeemRequest)
RedeemPosition storage position = redeemRequests[controller][requestId];
if (!position.queued) {
    _appendControllerRedeem(controller, requestId);
}
// TrancheVault.sol:1006-1012, 1299-1311: the standard claims and limits read only the head of that queue
return _claimRedeem(controllerRedeemHead[controller], shares, receiver, controller);
uint256 requestId = controllerRedeemHead[controller];
return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
// TrancheVault.sol:942-945 (claimRedeemRefund, the only call that removes a refunded head)
if (shares != 0) {
    lastDepositTime[receiver] = block.timestamp;
    _transfer(address(this), receiver, shares);
}
```

So an account that holds one share atom (about 0.000000001 USDG) can do the following with two transactions and no approval of any kind by the LP: move the atom to a second address, and from there call `requestRedeem(1, LP, secondAddress)`. The request is now the head of the LP's claim queue. One atom quotes to zero assets, so the next settlement turns that epoch into a zero-value refund. From then on, measured in the PoC:

- The LP's own later request is funded as usual (49,999.999982 USDG held for it in the vault's escrow, the same amount as in the reference run), but `maxRedeem(LP)` and `maxWithdraw(LP)` are 0 and `redeem` / `withdraw` revert with `ERC4626ExceededMaxRedeem(LP, shares, 0)` / `ERC4626ExceededMaxWithdraw(LP, assets, 0)`. Unchanged after 24 hours and after 30 days. No other account can remove the head (`NotControllerOrOperator`).
- The only call that removes the head is the LP's own `claimRedeemRefund`. It delivers the other account's atom to the LP and restarts the LP's one-hour cooldown on its whole remaining balance: `requestRedeem` reverts `DepositCooldown`, `transfer` reverts `TransferDuringCooldown`, the limit is 0 at 3,599 s and free at 3,600 s, and the LP's remaining 50 % is funded 3,600 s later than in the reference run. This is the end state the LOW-40 fix was written against, reached through the argument the fix did not touch. Cancelling the other account's request before it matures has the same effect on the cooldown.
- A holder contract that uses only the standard ERC-7540 / ERC-4626 functions (`requestRedeem`, `redeem`, `withdraw`) has 49,999.999996 USDG funded and cannot reach it: both claim functions revert at once, after 24 hours and after 30 days. The reference holder, same blocks, no such request, is paid by `redeem(maxRedeem)`.

Ways out, stated as measured facts:

- `claimRedeem(requestId, shares, receiver, controller)`, the vault's explicit-id claim (documented in the interface, see "Payout paths the documents give a holder"), pays the funded amount in the funding block, 0 s later than the reference; an account that uses it and never removes the head loses nothing and waits for nothing (same total and same timing as the reference). The standard limits stay 0 for as long as the head is not removed.
- An LP that requests (or moves to another wallet) its remaining shares before it removes the head keeps the reference timing; the cooldown then covers only the one atom.
- A holder contract that can call the standard `setOperator` gets out when an approved operator sends `claimRedeem(requestId, ...)` or `claimRedeemRefund` for it. A holder contract without that call has no call that reaches the funds.

The acting account gains nothing: its balances change by minus one share atom and zero USDG.

## Vulnerability Details

### Root cause
`requestRedeem(shares, controller, owner)` lets the share owner pick any address as controller (`TrancheVault.sol:606-632`; the authorisation at `:625` is for `owner` only), and `_enqueueRedeemRequest` links the request into that controller's queue (`:688-708`, `:1783-1797`). The controller did nothing and approved nothing. The standard claim functions `redeem` / `withdraw` and the limits `maxRedeem` / `maxWithdraw` use `controllerRedeemHead[controller]` only (`:998-1012`, `:1299-1311`). A head that ended as a zero-value refund has no claimable shares, so they return 0 and revert, whatever is funded behind it. The head leaves the queue only through `claimRedeemRefund` (controller or its operator, `receiver == controller`), which writes `lastDepositTime[controller] = block.timestamp` when it returns a nonzero share amount (`:916-947`).

A request of one atom is accepted because a request of a wallet's whole requestable balance is exempt from the minimum size (`:1611-1618`); `SECURITY.md:1409-1410` lists that exemption. That is why the atom is first moved to an address that holds nothing else.

The other account's request becomes a separate head only when its epoch is EARLIER than the LP's own request. In the same epoch it merges into the LP's own position, and in a later epoch it sits behind it; in both cases the standard claim pays (test 8). The request can be filed at any time before the LP's request, also against an address that holds nothing yet (test 4), and it stays until the controller acts.

The sibling request route does ask for the controller's authority: `requestRedeemFromClaimableDeposit` starts with `_requireControllerAuth(controller)` (`:658`), and `CANONICAL_ENTRYPOINTS.md:193-195` says of it "The controller or its approved operator must call, the source lot's activation-aged cooldown must have elapsed, and the redemption keeps the same controller." `requestRedeem` has no such check.

### Payout paths the documents give a holder
- `CANONICAL_ENTRYPOINTS.md:191-192` names one claim route: "claimed through ERC-4626 `redeem` or `withdraw`". No `.md` file of the repository mentions `claimRedeem` or an explicit-id claim (searched all `*.md`).
- The in-scope interface documents the second route: `IAsyncTrancheVault.sol:8-10` "explicit-id claim methods preserve deterministic epoch selection for integrations", and `:121-126` on `claimRedeem`: "Claims a specified share amount from one funded redemption request."
- docs.plether.com ("Withdraw liquidity") describes the holder's step as the per-request action **Move USDG to wallet** ("These are separate owner-wallet transactions"); the public front end sends `claimRedeem(requestId, ...)` for it.

So a funded request has two documented payout paths. The effect of this report is on the first one (`redeem` / `withdraw`, and the limits `maxRedeem` / `maxWithdraw`). The second one pays, and every statement about time below is given per path.

### Why this is the open half of LOW-40 / LOW-39
The fix commit says: "Restrict deposit share claims and redemption cancellations/refunds to the controller (LOW-39 and LOW-40), preserving operator execution and USDC recipient selection." Its change in `TrancheVault.sol`:

```
-        // With one cooldown timestamp per account, accepting unsolicited dust into an existing balance would let any
-        // controller repeatedly lock that receiver's entire pre-existing balance. New receivers and self-claims are safe.
-        if (receiver != controller && balanceOf(receiver) != 0) {
-            revert TrancheVault__ThirdPartyDepositForExistingHolder();
+        // Keep share delivery independent of receiver balances and require the controller's authority over cooldowns.
+        if (receiver != controller) {
+            revert TrancheVault__ReceiverNotController();
```

`requestRedeem` and `_enqueueRedeemRequest` are outside the diff. On the path of this report the receiver IS the controller, so the new check passes, and the result is the one the removed comment describes: unsolicited dust delivered into an existing balance together with a cooldown on that whole balance. The difference from LOW-40, stated plainly: here the cooldown is written by the controller's own call (or its operator's), which is the call the interface comment prescribes for a refundable head (`IAsyncTrancheVault.sol:8-10`: "A rejected deposit or refundable redeem head must be cancelled or reclaimed before standard claims advance to a later request").

### The sequence, cheapest form (test 1)
Junior vault; LP holds 100,000 USDG of shares, the other account 1,000 USDG; both past their cooldown; epoch `e0`, 300 s after its start.

1. Other account, transaction 1: `juniorVault.transfer(secondAddress, 1)`.
2. Other account, transaction 2 (from the second address): `juniorVault.requestRedeem(1, LP, secondAddress)`. Request id `e0+1`; `controllerRedeemHead(LP) == e0+1`; `pendingRedeemRequest(e0+1, LP) == 1`. The LP's wallet shares and cooldown timestamp are unchanged. The other account cannot take the request back (`NotControllerOrOperator`).
3. Start of epoch `e0+1`: the regular permissionless settlement processes that epoch as a zero-value refund (`redeemRefundPending(e0+1, LP)`, 1 refundable share atom).
4. 300 s later the LP requests redemption of half its shares (request id `e0+2`); the head stays `e0+1`.
5. Start of epoch `e0+2`: settlement funds the LP's request: `claimableRedeemAssets(e0+2, LP)` = 49,999.999982 USDG, `withdrawalEscrowAssets` = the same amount. `maxRedeem(LP) == 0`, `maxWithdraw(LP) == 0`, `redeem` and `withdraw` revert.
6. The LP calls `claimRedeemRefund(e0+1, LP, LP)`: it receives the atom, `lastDepositTime(LP)` becomes the current time, `maxRedeem` / `maxWithdraw` show the funded claim and `redeem` pays 49,999.999982 USDG. `maxRequestRedeem(LP)` is 0 for the next 3,599 s; the remaining half is requested at 3,600 s, joins epoch `e0+4` instead of `e0+3`, and is funded 3,600 s later than in the reference (total paid 99,999.999946 USDG against 99,999.999955 USDG).

Gas of the two transactions of the other account: 79,858 and 439,702 (each sent as its own transaction); of the LP's cleanup: 169,466.

### Other measurements
- **Repeated form (test 2):** the other account files one 1-atom request 300 s into each of 26 epochs; the LP wants to exit 1/30 of its position every hour. An LP that removes the head before it requests gets none of 26 requests through (each reverts `DepositCooldown`; 0 USDG paid against 83,333.333160 USDG in the reference; its limit is free at each epoch boundary and 0 at +300 s, +1,800 s and +3,599 s). An LP that requests first and removes the head afterwards is paid exactly as the reference (one cleanup; the later atoms merge into its own positions). An LP that never removes the head and claims by request id is paid exactly as the reference, with `redeem` reverting on each of its 25 funded claims.
- **No cap on queue entries (test 3):** 30 epochs with one 1-atom request each (31 atoms), none of the LP's own. The LP's next request is funded (49,999.999852 USDG) and the standard limits are 0 until the LP has sent 30 `claimRedeemRefund` calls (5,281,180 gas in total). Two requests in one epoch share one entry. The settlement in this test processes one zero-value epoch per regular call; the report does not rest on settlement work.
- **Holder contract (test 4):** described above; the 1-atom requests were filed before the contracts held anything.
- **Senior vault (test 6):** the same form: 25,000.684935 USDG funded for the Senior LP, standard limits 0, both claims revert; cleanup restarts its cooldown (0 at 3,599 s, free at 3,600 s).
- **Deposit side (test 5):** a 1 USDG deposit request naming an address becomes the head of its deposit claims: `maxDeposit` shows 1 USDG instead of the address's own 10,000 USDG lot and `deposit(10,000 USDG)` reverts. One extra standard `deposit(1 USDG)` clears it, the cooldown timestamp is the same as in the reference, and the named address keeps the 1 USDG lot. No lasting effect on this side.
- **Funded-epoch variant (test 7):** when the atom's epoch also holds a real request, the epoch is funded, `maxRedeem` is 1 and `maxWithdraw` is 0; the standard `redeem(1)` clears it with no cooldown change.

## Preconditions and limits
- The acting account needs one share atom of the tranche and one spare address. No role, no approval, no price move, no timing against the LP beyond "an epoch earlier than the LP's request".
- The LP's funds are never lost and never priced differently. What is taken away is the standard claim route and, on cleanup, one hour of the remaining shares.
- An account that claims by request id (a documented path) is not delayed at all: 0 s in test 1, and the same 83,333.333160 USDG as the reference over 26 hours in test 2. The project's web front end (public `plether-app` repository) claims with `claimRedeem(requestId, ...)`, so its users receive funded USDG as usual; they would see the other account's request as an extra "Shares ready to return" item whose button restarts their cooldown (docs.plether.com, "LP troubleshooting": "Returning shares restarts the one-hour cooldown for every share in the wallet's position in that tranche.").
- The cooldown on cleanup is avoidable by an LP that knows to request or move its remaining shares first.
- The freeze of a holder contract depends on that contract: with `setOperator` it ends when an operator acts; a contract with only `requestRedeem` / `redeem` / `withdraw` has no call that ends it. The contracts in the PoC are test contracts written for this report; no deployed integration was examined.
- The spare address is one extra transfer made necessary by the minimum-size rule, not a use of many identities.

## Impact Details

All 13 impact rows of the program, element by element, with the time assertion of the PoC next to each freezing row:

| program row (verbatim) | elements | met? test, value and time assertion |
|---|---|---|
| high \| Temporary freezing of funds for at least 24 hours | (a) funds; (b) unavailable on every payout path of the holder; (c) at least 24 hours; (d) temporary | **Holder contract limited to the standard functions (test 4):** (a) 49,999.999996 USDG funded in escrow. (b), (c) asserted 86,400 s and 2,592,000 s after funding: `redeem(maxRedeem)` and `withdraw(maxWithdraw)` through the contract revert; the explicit-id claim, the refund call and a cancellation sent by its owner revert `NotControllerOrOperator`; 0 USDG paid. (d) nothing in the vault ends it by time; a holder with `setOperator` gets out when its owner approves an operator and that operator sends a non-standard call (asserted), so its length is then up to that owner; a holder without that call has no way out. **Account that can send `claimRedeem(requestId, ...)`: not met**, paid 0 s after funding (test 1) |
| critical \| Permanent freezing of funds | (a) funds; (b) cannot be moved; (c) permanently | (a), (b) as in the row above. (c) asserted to 2,592,000 s for a holder contract with no `setOperator` call (test 4, H1); the vault has no time-based release. Not met for a holder with `setOperator` (the operator's call pays, test 4, H2) or for an account that can send `claimRedeem(requestId, ...)` (0 s, test 1) |
| medium \| Temporary freezing of funds for at least 1 hour | (a) funds; (b) unavailable; (c) at least 1 hour | **Holder that follows the `redeem` / `withdraw` path and therefore removes the head first:** the funded half is paid 0 s after funding; its remaining shares (50,000 USDG) are closed on both wallet paths, `requestRedeem` (`DepositCooldown`) and `transfer` (`TransferDuringCooldown`), asserted at 0 s and at 3,599 s after the cleanup, open at 3,600 s, and funded 3,600 s later than the reference (test 1; Senior limit 0 at 3,599 s, test 6); with a request of the other account in every epoch, 0 of 26 requests in 26 hours (test 2, A). **Not met** for an account that claims by request id and never removes the head (0 s; test 1, test 2 B), or that requests / moves its remaining shares before the cleanup (0 s; test 1, test 2 A') |
| medium \| Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol) | (a) no profit motive; (b) damage to users, with a number | (a) met: the acting account's balances change by -1 share atom and 0 USDG; its cost is 2 transactions, 519,560 gas (79,858 + 439,702) (test 1). (b) for a holder on the `redeem` / `withdraw` path, in the cheapest scenario of the other account: `maxRedeem` / `maxWithdraw` 0 and both claims reverting with 49,999.999982 USDG funded; 1 extra transaction (`claimRedeemRefund`, 169,466 gas); the remaining 50,000 USDG funded 3,600 s later (test 1). After 30 epochs: 30 extra transactions, 5,281,180 gas (test 3). For a holder on the explicit-id path: 0 s and no extra transaction (`claimRedeem` 197,104 gas against 194,453 for the reference `redeem`) |
| low \| Contract fails to deliver promised returns, but doesn't lose value | (a) a documented promise is not delivered; (b) no value is lost | (a) met: "claimed through ERC-4626 `redeem` or `withdraw`" and "share delivery cannot impose another account's cooldown" (tests 1, 4, 6). (b) met: totals equal the reference in the ways out (99,999.999955 USDG, test 1) |
| critical \| Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield | funds end with another party | not met: nobody receives the LP's funds; the LP receives the atom |
| critical \| Protocol insolvency | liabilities above assets | not met |
| high \| Theft of unclaimed yield | yield ends with another party | not met |
| high \| Permanent freezing of unclaimed yield | yield frozen | not met as yield: the amounts are principal redemptions |
| medium \| Smart contract unable to operate due to lack of token funds | contract short of tokens | not met |
| medium \| Theft of gas | - | not applicable |
| medium \| Block stuffing | - | not applicable |
| medium \| Unbounded gas consumption | gas without bound | not met: the cleanup is one bounded call per epoch entry (30 calls for 30 epochs, test 3) |

- **Claimed row: Medium, "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".** It is met for every kind of account that uses the documented `redeem` / `withdraw` route: the acting account spends one share atom and two transactions and gains nothing; the holder's standard limits are 0 and both standard claims revert while 49,999.999982 USDG is funded for it; the holder pays one extra transaction and has its remaining 50,000 USDG funded 3,600 s later (test 1).
- **The freezing rows, as measured.** "Temporary freezing of funds for at least 24 hours" (High) is met for a holder contract limited to the vault's standard ERC-7540 / ERC-4626 functions: 49,999.999996 USDG funded and every payout path of that holder closed 86,400 s and 30 days after funding (test 4). Its way out is an approved operator that sends a non-standard call; a holder contract without `setOperator` has none. For an account that can send `claimRedeem(requestId, ...)` no freezing row is met.
- **Cost and gain of the acting account:** two transactions and one share atom; no gain. The program page compares cost and damage for no-profit findings ("spending USD 1 to cause USD 10 or less in damage is Griefing, while spending USD 1 to cause USD 100 or more in damage is not"). The cost here is one share atom and the gas of two transactions (79,858 and 439,702); the amount put out of reach of the standard functions is the controller's whole funded claim (49,999.999996 USDG in test 4).
- **Likelihood:** any share holder can do it to any address at any time, including before that address has a position.

## Known-issue review
Sources searched (2026-11-13): the program page copy (known issues, out-of-scope list); `packages/perps/SECURITY.md` including "Known Limitations"; `ACCOUNTING_SPEC.md`, `WHITEPAPER.md`, `CANONICAL_ENTRYPOINTS.md`, `README.md` and the other package documents; the interface NatSpec; the Cecuro AI audit of 2026-10-26, the AI scan of 2026-10-15, the pre-audit report, the collaborative audit of 2026-05-12 and the Cantina report; the project's own tests and test map; git history; issues and pull requests of `Plether-Fi/plether-core` (bodies of #61, #62, #79, #123, #128); docs.plether.com; the public front-end repository.

Closest items and how they differ:
- **Cecuro LOW-40** "Controller can force a zero-balance receiver into LP cooldown" and **LOW-39** "Front-runner can block third-party vault share claims by dusting the receiver" (both Addressed, fix d757d0b1): the same two effects (another account's dust blocks a claim route; another account's dust ends in a cooldown), through the `receiver` argument of claims, cancellations and refunds. This report is the path the fix left open: the `controller` argument of `requestRedeem`. Program rule for fixed items: "a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug."
- **`IAsyncTrancheVault.sol:8-10`**: "Standard ERC-7540 claims consume the controller's oldest outstanding request. A rejected deposit or refundable redeem head must be cancelled or reclaimed before standard claims advance to a later request; explicit-id claim methods preserve deterministic epoch selection for integrations." This documents head-first claims for a controller's requests. It does not say that an account with no authority over the controller can create that head, and the next sentences of the same comment say that share delivery "cannot impose another account's cooldown".
- **`SECURITY.md` Known Limitations** (`:1396-1399`, `:1403-1414`): "Redemption cancellation or terminal refund still begins a new cooldown when shares return to a wallet."; "Per-controller flooring can produce zero entitlement for a dust-sized request. A full-balance redemption bypasses the minimum request size"; "Zero-payout epochs can consume bounded settlement work and delay later epochs." These are the mechanisms for a holder's own request and for settlement work. None concerns a request that names another account as controller, the head of a controller's queue, or the standard claim limits.
- **Cecuro MED-24** "LP can delay other redeemers with zero-payout refund epochs" (program known issue): the same primitive (a 1-share request that becomes a zero-payout epoch) used against the settlement work budget of 16 epochs per call. This report does not use that effect: in the PoC every epoch boundary is settled by a regular call that processes one such epoch.
- **Cecuro MED-29, MED-5, LOW-3, LOW-4, LOW-5, LOW-13, LOW-63, LOW-67** (rounding, partial claims, dust ownership, seed receiver): none has a request whose controller is not its owner. **LOW-1** and **LOW-43** note that a controller "can differ from the funding account"; neither is about the queue head.
- **Pre-audit SC-H2** "Zero-Amount Withdraw/Redeem Can Perpetually Reset TrancheVault Cooldown and Lock Funds" (fixed in the earlier synchronous vault): the same class of effect on code that has since been replaced.
- **The project's tests:** `AsyncTrancheVaultAllocation.t.sol:231-249` tests head-first claims with a controller's own two requests. No project test files a `requestRedeem` whose controller is neither the owner nor an account that approved the caller; the test map states the limit of the redemption fixtures as "At most two epochs and one unfinished redemption per controller" (`TEST_MAP.md:330`).
- AI scan, GitHub issues and pull requests, docs.plether.com: no item on a request made by another account in a controller's queue.

## Recommended fix
Give the controller the authority over what enters its queue, in line with the LOW-40 fix ("require the controller's authority over cooldowns"): in `requestRedeem`, when `controller != owner`, require `msg.sender == controller || isOperator[controller][msg.sender]`. Alternatively, or in addition, let `redeem` / `withdraw` / `maxRedeem` / `maxWithdraw` pass over a head that is only refundable, and return refund shares of a request to the account that supplied them, so that removing such a head does not write the controller's cooldown.

## References
- Code (pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`): `packages/perps/src/TrancheVault.sol#L606-L632`, `#L688-L708`, `#L916-L947`, `#L998-L1012`, `#L1299-L1311`, `#L1611-L1628`, `#L1783-L1797`; `packages/perps/src/interfaces/IAsyncTrancheVault.sol#L7-L17`, `#L133-L137`.
- Fix commit of LOW-39 / LOW-40: d757d0b1 "fix(perps): resolve LP vault, tranche, and claim findings (#123)".
- Documents: `packages/perps/CANONICAL_ENTRYPOINTS.md:186-208`; `packages/perps/SECURITY.md:1396-1414`; `packages/perps/ACCOUNTING_SPEC.md:603-604`; `packages/perps/test/perps/TEST_MAP.md:222`, `:330`; docs.plether.com "Withdraw liquidity", "LP troubleshooting".
- Project tests: `test/perps/spec/lp/DepositCooldownBehavior.t.sol:199-232`, `test/perps/spec/lp/TrancheShareReceiver.t.sol:61-78`, `test/perps/spec/lp/AsyncTrancheVaultAllocation.t.sol:231-249`.

## Proof of Concept

## Proof of Concept

Runnable Foundry test in the repository's own suite at the pinned commit. No in-scope contract is modified; the only mocks are the fixture's MockPyth and mock settlement token (outside the scope). No `vm.store`, no `vm.etch`, no protocol address impersonated, no position opened; Senior cap 40M / 80 %.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test below as packages/perps/test/PocS10_ForeignRedeemHead.t.sol
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/PocS10_*' -vv   # Foundry 1.5.1
```
No env vars or RPC needed. Expected: `8 passed; 0 failed` (194 files, about 60 s cold). With `--isolate` every top-level call is its own transaction; the gas figures of the description are from that mode.

**Accounts.** Every `vm.prank` is for an account without any role: `LP`, `LP2` (the LP and its second wallet), `NEWLP`, `SLP` (Senior LP), `JB`, `SB`, `OTHER` (the other account) with its spare addresses `HELPER`, `HELPER2`, `BEN` (owner of the holder contracts), `OP` (operator approved by holder `H2`), `NOBODY`, `KEEPER` (any caller: `router.updateMarkPrice`, `pool.settleLpEpoch(0, 0)`). The test contract owns the fixture and acts only in `setUp` (deployment, the two 1 USDG seeds, `_setSeniorCapacity`); in the tests it only mints mock USDG to depositors.

**Setup (USDG):** `LP` deposits 100,000 and `JB` 1,000 into Junior (request, settlement, claim); both are past the one-hour cooldown; the tests start 300 s into epoch `e0`.

**Steps of the leading form (test 1)**
1. `JB`: `juniorVault.transfer(HELPER, 1)`. `HELPER`: `juniorVault.requestRedeem(1, LP, HELPER)` (id `e0+1`): head of `LP`'s queue. `LP` unchanged; `JB`: -1 share atom, 0 USDG.
2. Start of `e0+1`: settlement turns the 1-atom epoch into a zero-value refund. 300 s later `LP` requests redemption of half its shares (id `e0+2`).
3. Start of `e0+2`: 49,999.999982 USDG is funded for `LP` (reference: the same). `maxRedeem(LP)` = `maxWithdraw(LP)` = 0; `redeem`, `withdraw` revert; unchanged after 24 hours and 30 days; nobody else can remove the head.
4. Ways out (snapshots): (a) `claimRedeem(e0+2, ...)` pays in the funding block, total 99,999.999955 USDG and timing as the reference; (b) `LP` requests its rest first, then `claimRedeemRefund`: as the reference; (c) `LP` moves its rest to `LP2` first.
5. Main line: `LP` calls `claimRedeemRefund(e0+1, LP, LP)` first: cooldown restarts; `requestRedeem` and `transfer` of the rest revert at 0 s and 3,599 s, open at 3,600 s; the rest is funded 3,600 s later than the reference; total 99,999.999946 USDG.

**Other tests:** 2 = 26 epochs, three ways the LP can act; 3 = 30 epochs, 30 cleanup calls; 4 = holder contracts limited to the standard functions (`H1` without, `H2` with `setOperator`, `H3` reference in the same blocks): every payout path closed 86,400 s and 30 days after funding; 5 = deposit side; 6 = Senior vault; 7 = the atom in a funded epoch; 8 = control: same or later epoch does not hide the claim. Tests 1, 2, 3, 5, 6 run a reference branch without the other account's request.

**Source-change control (not in the file):** three single-comparison changes of `TrancheVault.sol` in a working copy (`:617` `<` to `<=`; `:927` `== 0` to `!= 0`; `:942` `!= 0` to `== 0`): the suite failed each time (3, 5, 5 failing tests); copy restored. **Not done:** a run with a fix installed.

Output (clean copy of the pinned tree, cold build):
```
Ran 8 tests for test/PocS10_ForeignRedeemHead.t.sol:PocS10_ForeignRedeemHead
[PASS] test_S10_1_OneAtomRequestNamingLp_StandardClaimsZeroWhileFunded_CleanupRestartsCooldown() (gas: 5761797)
  T1 REF gas: redeem(shares, receiver, controller) 149485
  T1 REF: first half paid (atoms) / total paid (atoms) 49999999982 99999999955
  T1 gas: transfer of 1 atom / requestRedeem(1, LP, helper) 54502 403094
  T1: funded for the LP (atoms) / reference (atoms) 49999999982 49999999982
  T1 gas: claimRedeem(requestId, ...) 147072
  T1 claim-by-id: total paid (atoms) / reference total (atoms) 99999999955 99999999955
  T1 request-first: total paid (atoms) / reference total (atoms) 99999999955 99999999955
  T1 gas: claimRedeemRefund (cleanup) 115234
  T1 clean-first: total paid (atoms) / reference total (atoms) 99999999946 99999999955
[PASS] test_S10_2_Repeated26Epochs_ThreeBehavioursAgainstReference() (gas: 82967156)
  REF
    requests accepted / reverted DepositCooldown 26 0
    claims redeem() / claimRedeem(id) / cleanups 25 0 0
    redeem() reverts with a funded claim 0
    samples / funded, maxRedeem 0 / shares, maxRequestRedeem 0 104 0 0
    boundaries with a free request limit 26
    USDG paid (atoms) / wallet shares 83333333160 13354419941346
  A  (clean first)
    requests accepted / reverted DepositCooldown 0 26
    claims redeem() / claimRedeem(id) / cleanups 0 0 26
    redeem() reverts with a funded claim 0
    samples / funded, maxRedeem 0 / shares, maxRequestRedeem 0 104 0 78
    boundaries with a free request limit 26
    USDG paid (atoms) / wallet shares 0 100158149560030
  A' (request first)
    requests accepted / reverted DepositCooldown 26 0
    claims redeem() / claimRedeem(id) / cleanups 25 0 1
    redeem() reverts with a funded claim 0
    samples / funded, maxRedeem 0 / shares, maxRequestRedeem 0 104 0 3
    boundaries with a free request limit 26
    USDG paid (atoms) / wallet shares 83333333160 13354419941347
  B  (never clean, claim by id)
    requests accepted / reverted DepositCooldown 26 0
    claims redeem() / claimRedeem(id) / cleanups 0 25 0
    redeem() reverts with a funded claim 25
    samples / funded, maxRedeem 0 / shares, maxRequestRedeem 0 104 25 0
    boundaries with a free request limit 26
    USDG paid (atoms) / wallet shares 83333333160 13354419941346
[PASS] test_S10_3_ThirtyEpochsNamingLp_ThirtyCleanupCalls() (gas: 18147396)
  T3: funded for the LP (atoms) / reference (atoms) 49999999852 49999999850
  T3: cleanup calls needed / gas sum 30 3377820
  T3: cleanup gas per call min / max 112434 117234
[PASS] test_S10_4_StandardOnlyHolderContract_FundedClaimUnreachable() (gas: 4984156)
  T4: funded USDG of H1 / H2 not reachable by the standard functions after 30 days (atoms) 49999999996 49999999996
[PASS] test_S10_5_DepositSide_OneExtraStandardClaim() (gas: 2828171)
  T5: gift shares to the named address / own shares 1001581495 10015814959570
[PASS] test_S10_6_SeniorVault_SameForm() (gas: 3586019)
  T6: funded for the Senior LP (atoms) / reference (atoms) 25000684935 25000684931
[PASS] test_S10_7_OneAtomInAFundedEpoch_StandardRedeemClearsIt() (gas: 1699822)
[PASS] test_S10_8_Control_SameOrLaterEpoch_StandardClaimPays() (gas: 1477996)
Suite result: ok. 8 passed; 0 failed; 0 skipped; finished in 120.77ms (164.06ms CPU time)
Ran 1 test suite in 139.35ms (120.77ms CPU time): 8 tests passed, 0 failed, 0 skipped (8 total tests)
[omitted from this paste: compiler warnings, 6 "Logs:" label lines, and the second run with --isolate (same copy; 8 passed, 0 failed) except its gas lines, which follow]
  T1 REF gas: redeem(shares, receiver, controller) 194453
  T1 gas: transfer of 1 atom / requestRedeem(1, LP, helper) 79858 439702
  T1 gas: claimRedeem(requestId, ...) 197104
  T1 gas: claimRedeemRefund (cleanup) 169466
  T3: cleanup calls needed / gas sum 30 5281180
  T3: cleanup gas per call min / max 166666 179066
```

### Full test code (`packages/perps/test/PocS10_ForeignRedeemHead.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// A redemption request of 1 share atom that ANOTHER account files with `controller = LP` joins the LP's claim queue.
// In an earlier epoch than the LP's own request it ends as a zero-value refund at the head of that queue, and the
// standard ERC-4626 / ERC-7540 claims and limits of the LP read that head: 0 / revert while the LP's request is funded.
// Each test has a reference branch without the other account's request. Expected values (project documents):
//   CANONICAL_ENTRYPOINTS.md:191-192 "claimed through ERC-4626 redeem or withdraw."
//   IAsyncTrancheVault.sol:12-14 "... so share delivery cannot impose another account's cooldown or depend on an
//     unsolicited receiver balance."
// Run: FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/PocS10_*' -vv
// Fixture: BasePerpTest (MockPyth, MockSettlementToken: outside the scope). After setUp: no owner-only call, no
// vm.store, no vm.etch. The contracts in scope are not modified.

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC4626} from "@openzeppelin/contracts/token/ERC20/extensions/ERC4626.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

/// @dev A holder that uses only the standard functions: requestDeposit, deposit, requestRedeem, redeem, withdraw.
contract StdOnlyHolder {

	TrancheVault public immutable vault;
	IERC20 public immutable token;
	address public immutable beneficiary;

	error NotBeneficiary();

	modifier onlyBeneficiary() {
		if (msg.sender != beneficiary) {
			revert NotBeneficiary();
		}
		_;
	}

	constructor(TrancheVault vault_, IERC20 token_, address beneficiary_) {
		vault = vault_;
		token = token_;
		beneficiary = beneficiary_;
	}

	function requestEntry(uint256 assets) external onlyBeneficiary returns (uint256 id) {
		token.approve(address(vault), assets);
		id = vault.requestDeposit(assets, address(this), address(this));
	}

	function claimEntry() external onlyBeneficiary returns (uint256 shares) {
		shares = vault.deposit(vault.maxDeposit(address(this)), address(this));
	}

	function requestExit(uint256 shares) external onlyBeneficiary returns (uint256 id) {
		id = vault.requestRedeem(shares, address(this), address(this));
	}

	function claimExitByShares() external onlyBeneficiary returns (uint256 assets) {
		assets = vault.redeem(vault.maxRedeem(address(this)), beneficiary, address(this));
	}

	function claimExitByAssets() external onlyBeneficiary returns (uint256 shares) {
		shares = vault.withdraw(vault.maxWithdraw(address(this)), beneficiary, address(this));
	}

}

/// @dev The same holder with one more standard ERC-7540 call: setOperator.
contract StdHolderWithOperator is StdOnlyHolder {

	constructor(TrancheVault vault_, IERC20 token_, address beneficiary_) StdOnlyHolder(vault_, token_, beneficiary_) {}

	function approveOperator(address operator, bool approved) external onlyBeneficiary {
		vault.setOperator(operator, approved);
	}

}

contract PocS10_ForeignRedeemHead is BasePerpTest {

	address internal constant KEEPER = address(0x0E0E1); // any account: calls the permissionless settlement
	address internal constant LP = address(0xA0A1); // the Junior LP whose actions are measured
	address internal constant LP2 = address(0xA0A2); // a second wallet of the LP
	address internal constant NEWLP = address(0xA0A3); // an address with no position yet (deposit side)
	address internal constant JB = address(0xB0B1); // the other account (a Junior holder)
	address internal constant HELPER = address(0xB0B2); // second wallet of the other account
	address internal constant HELPER2 = address(0xB0B3); // third wallet of the other account
	address internal constant OTHER = address(0x5E01); // the other account on the deposit side
	address internal constant NOBODY = address(0xD0D1);
	address internal constant SLP = address(0xC0C1); // the Senior LP
	address internal constant SB = address(0xC0C2); // the other account (a Senior holder)
	address internal constant BEN = address(0xBE01); // beneficiary / owner of the holder contracts
	address internal constant OP = address(0x0BE2); // an operator the holder contract can approve

	uint256 internal constant MON = 1_710_151_200; // Mon 2024-03-11 10:00 UTC, a round hour, live market
	uint256 internal constant P0 = 1e8;
	uint256 internal constant HOUR = 3600;

	uint8 internal constant MODE_REF = 0; // no request of the other account
	uint8 internal constant MODE_A = 1; // the LP cleans the head first, then claims, then requests
	uint8 internal constant MODE_A2 = 2; // the LP requests first, then cleans, then claims (A')
	uint8 internal constant MODE_B = 3; // the LP never cleans and claims by request id

	uint256[] internal _ownIds; // the LP's own request ids in the repeated test

	function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
		return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
	}

	function _initialJuniorDeposit() internal pure override returns (uint256) {
		return 0;
	}

	function _initialSeniorDeposit() internal pure override returns (uint256) {
		return 0;
	}

	function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
		return 1e6;
	}

	function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
		return 1e6;
	}

	function setUp() public virtual override {
		super.setUp();
		_setSeniorCapacity(40_000_000e6, 8000);
		_warpTo(MON);
		_pushMark(P0);
		// LP book through the public async path: LP 100,000 Junior, JB 1,000 Junior.
		uint256 a = _reqDep(juniorVault, LP, 100_000e6);
		uint256 b = _reqDep(juniorVault, JB, 1000e6);
		assertEq(a, b);
		_warpTo(pool.lpEpochStart(a));
		_pushMark(P0);
		_settle();
		_claimDep(juniorVault, LP, a);
		_claimDep(juniorVault, JB, a);
		// past the 1 h holder cooldown, 5 minutes into a fresh epoch (well before the b - 300 s cutoff)
		_warpTo(pool.lpEpochStart(a) + 2 * HOUR + 300);
		_pushMark(P0);
	}

	function _now() internal view returns (uint256) {
		return vm.getBlockTimestamp();
	}

	function _warpTo(uint256 t) internal {
		vm.warp(t);
		vm.roll(vm.getBlockNumber() + 1);
	}

	function _liveData(uint256 price) internal view returns (bytes[] memory d) {
		bytes32[] memory ids = _basePythFeedIds();
		MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
		for (uint256 i; i < ids.length; ++i) {
			u[i] = MockPyth.FeedUpdate(
				ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), engine.lastMarkTime())
			);
		}
		d = new bytes[](1);
		d[0] = abi.encode(u);
	}

	function _pushMark(uint256 price) internal {
		bytes[] memory d = _liveData(price);
		vm.prank(KEEPER);
		router.updateMarkPrice(d);
	}

	/// @dev Permissionless cached-mark settlement (no open positions in this file), sent by KEEPER (no role).
	function _settle() internal returns (IHousePool.LpEpochSettlementResult memory r) {
		vm.prank(KEEPER);
		r = pool.settleLpEpoch(0, 0);
	}

	/// @dev The same call; the only failure accepted is "nothing to do".
	function _trySettle() internal returns (bool ok, IHousePool.LpEpochSettlementResult memory r) {
		vm.prank(KEEPER);
		(bool s, bytes memory ret) = address(pool).call(abi.encodeCall(pool.settleLpEpoch, (0, 0)));
		if (s) {
			return (true, abi.decode(ret, (IHousePool.LpEpochSettlementResult)));
		}
		assertEq(bytes4(ret), IHousePool.HousePool__NoLpEpochProgress.selector, "settlement fails only with no work");
	}

	/// @dev The keeper's regular work at an epoch boundary: fresh mark, then one settlement call.
	function _boundary(uint256 epoch) internal returns (bool ok, IHousePool.LpEpochSettlementResult memory r) {
		_warpTo(pool.lpEpochStart(epoch));
		_pushMark(P0);
		(ok, r) = _trySettle();
	}

	function _reqDep(TrancheVault v, address lp, uint256 amt) internal returns (uint256 id) {
		settlementToken.mint(lp, amt);
		vm.startPrank(lp);
		settlementToken.approve(address(v), amt);
		id = v.requestDeposit(amt, lp, lp);
		vm.stopPrank();
	}

	function _claimDep(TrancheVault v, address lp, uint256 id) internal returns (uint256 shares) {
		uint256 a = v.claimableDepositRequest(id, lp);
		vm.prank(lp);
		shares = v.claimDeposit(id, a, lp, lp);
	}

	function _usdg(address a) internal view returns (uint256) {
		return settlementToken.balanceOf(a);
	}

	// expected reverts, with full error data
	function _expectNotController() internal {
		vm.expectRevert(TrancheVault.TrancheVault__NotControllerOrOperator.selector);
	}

	function _expectCooldown() internal {
		vm.expectRevert(TrancheVault.TrancheVault__DepositCooldown.selector);
	}

	function _expectMaxRedeem(address controller, uint256 shares, uint256 max) internal {
		vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxRedeem.selector, controller, shares, max));
	}

	function _expectMaxWithdraw(address controller, uint256 assets, uint256 max) internal {
		vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxWithdraw.selector, controller, assets, max));
	}

	/// @dev The other account's two transactions: `src` moves 1 share atom to `helper`; `helper` requests its
	///      redemption and names `controller`. No role, no approval by `controller`.
	function _dust(TrancheVault v, address src, address helper, address controller) internal returns (uint256 id) {
		vm.prank(src);
		v.transfer(helper, 1);
		vm.prank(helper);
		id = v.requestRedeem(1, controller, helper);
	}

	// Test 1: the leading form. Two transactions of another account, 1 share atom.

	struct T1 {
		uint256 lpShares;
		uint256 half;
		uint256 rest;
		uint256 e0;
		uint256 lpLast0;
		uint256 refAssets;
		uint256 refRestFundedAt;
		uint256 refTotal;
		uint256 dustId;
		uint256 lpRid;
		uint256 funded;
		uint256 tFunded;
	}

	function test_S10_1_OneAtomRequestNamingLp_StandardClaimsZeroWhileFunded_CleanupRestartsCooldown() public {
		T1 memory t;
		t.lpShares = juniorVault.balanceOf(LP);
		t.half = t.lpShares / 2;
		t.rest = t.lpShares - t.half;
		t.e0 = pool.currentLpEpoch();
		t.lpLast0 = juniorVault.lastDepositTime(LP);
		assertEq(_now(), t.e0 * HOUR + 300, "start: 300 s into epoch e0");
		assertEq(_usdg(LP), 0);

		uint256 snap = vm.snapshotState();
		_t1Reference(t);
		vm.revertToState(snap);

		_t1OtherAccountTwoTransactions(t);

		snap = vm.snapshotState();
		_t1CancelBeforeMaturityRestartsCooldownToo(t);
		vm.revertToState(snap);

		_t1LpRequestIsFundedButStandardClaimsShowNothing(t);

		snap = vm.snapshotState();
		_t1TimeDoesNotClearAndNobodyElseCan(t);
		vm.revertToState(snap);

		snap = vm.snapshotState();
		_t1WayOutClaimByRequestId(t);
		vm.revertToState(snap);

		snap = vm.snapshotState();
		_t1WayOutRequestFirstThenClean(t);
		vm.revertToState(snap);

		snap = vm.snapshotState();
		_t1WayOutTransferFirstThenClean(t);
		vm.revertToState(snap);

		assertEq(_now(), t.tFunded);
		_t1CleanFirst(t);
	}

	/// Reference branch: no request of the other account.
	function _t1Reference(T1 memory t) internal {
		_warpTo((t.e0 + 1) * HOUR + 300);
		_pushMark(P0);
		vm.prank(LP);
		uint256 rid = juniorVault.requestRedeem(t.half, LP, LP);
		assertEq(rid, t.e0 + 2, "REF: request id");
		assertEq(juniorVault.controllerRedeemHead(LP), rid, "REF: the head is the LP's own request");
		(bool ok, IHousePool.LpEpochSettlementResult memory r) = _boundary(t.e0 + 2);
		assertTrue(ok);
		assertEq(r.juniorFundedShares, t.half, "REF: funded");
		assertEq(r.juniorProcessedEpochs, 1);
		// CANONICAL_ENTRYPOINTS.md:191-192: funded requests are claimed through ERC-4626 redeem or withdraw
		assertEq(juniorVault.maxRedeem(LP), t.half, "REF: maxRedeem == funded shares");
		t.refAssets = juniorVault.maxWithdraw(LP);
		assertEq(t.refAssets, r.juniorFundedAssets, "REF: maxWithdraw == funded assets");
		assertGt(t.refAssets, 49_990e6, "REF: about 50,000 USDG");
		vm.prank(LP);
		uint256 paid = juniorVault.redeem(t.half, LP, LP);
		console.log("T1 REF gas: redeem(shares, receiver, controller)", vm.lastCallGas().gasTotalUsed);
		assertEq(paid, t.refAssets);
		assertEq(_now(), (t.e0 + 2) * HOUR, "REF: paid in the funding block");
		assertEq(_usdg(LP), t.refAssets, "REF: LP paid by redeem()");
		assertEq(juniorVault.lastDepositTime(LP), t.lpLast0, "REF: cooldown timestamp untouched");
		// the LP's remaining shares: requested at once, funded one epoch later
		assertEq(juniorVault.maxRequestRedeem(LP), t.rest);
		vm.prank(LP);
		uint256 rid2 = juniorVault.requestRedeem(t.rest, LP, LP);
		assertEq(rid2, t.e0 + 3, "REF: remaining shares join epoch e0+3");
		(ok, r) = _boundary(t.e0 + 3);
		assertTrue(ok);
		assertEq(r.juniorFundedShares, t.rest);
		t.refRestFundedAt = _now();
		assertEq(juniorVault.maxRedeem(LP), t.rest);
		vm.prank(LP);
		juniorVault.redeem(t.rest, LP, LP);
		t.refTotal = _usdg(LP);
		assertEq(juniorVault.balanceOf(LP), 0);
		console.log("T1 REF: first half paid (atoms) / total paid (atoms)", t.refAssets, t.refTotal);
	}

	function _t1OtherAccountTwoTransactions(T1 memory t) internal {
		uint256 jbShares = juniorVault.balanceOf(JB);
		uint256 escrowShares = juniorVault.balanceOf(address(juniorVault));
		uint256 supply = juniorVault.totalSupply();
		uint256 vaultUsdg = _usdg(address(juniorVault));
		assertEq(_usdg(JB), 0);
		assertEq(_usdg(HELPER), 0);
		assertEq(juniorVault.balanceOf(HELPER), 0);
		assertEq(juniorVault.controllerRedeemHead(LP), 0);
		assertFalse(juniorVault.isOperator(LP, HELPER));
		assertFalse(juniorVault.isOperator(LP, JB));
		assertEq(juniorVault.allowance(LP, HELPER), 0);

		// transaction 1: JB moves 1 share atom to its second wallet
		vm.prank(JB);
		juniorVault.transfer(HELPER, 1);
		uint256 gasTransfer = vm.lastCallGas().gasTotalUsed;
		// transaction 2: the second wallet requests redemption of its whole balance (1 atom) and names the LP
		vm.prank(HELPER);
		t.dustId = juniorVault.requestRedeem(1, LP, HELPER);
		uint256 gasRequest = vm.lastCallGas().gasTotalUsed;
		console.log("T1 gas: transfer of 1 atom / requestRedeem(1, LP, helper)", gasTransfer, gasRequest);
		assertLt(gasTransfer, 200_000);
		assertLt(gasRequest, 600_000);

		assertEq(t.dustId, t.e0 + 1);
		assertEq(juniorVault.balanceOf(JB), jbShares - 1, "other account: -1 share atom");
		assertEq(juniorVault.balanceOf(HELPER), 0);
		assertEq(_usdg(JB), 0, "other account: USDG unchanged");
		assertEq(_usdg(HELPER), 0);
		assertEq(juniorVault.balanceOf(address(juniorVault)), escrowShares + 1);
		assertEq(juniorVault.totalSupply(), supply);
		assertEq(_usdg(address(juniorVault)), vaultUsdg);
		assertEq(juniorVault.balanceOf(LP), t.lpShares);
		assertEq(_usdg(LP), 0);
		assertEq(juniorVault.controllerRedeemHead(LP), t.dustId, "the other account's request is the LP's queue head");
		assertEq(juniorVault.pendingRedeemRequest(t.dustId, LP), 1);
		assertEq(juniorVault.pendingRedeemRequest(t.dustId, HELPER), 0);
		assertEq(juniorVault.controllerRedeemHead(HELPER), 0);
		assertEq(juniorVault.lastDepositTime(LP), t.lpLast0);
		assertEq(juniorVault.maxRequestRedeem(LP), t.lpShares);
		vm.prank(HELPER);
		_expectNotController();
		juniorVault.cancelRedeemRequest(t.dustId, LP, LP);
		vm.prank(HELPER);
		vm.expectRevert(TrancheVault.TrancheVault__NoPendingRedeem.selector);
		juniorVault.cancelRedeemRequest(t.dustId, HELPER, HELPER);
	}

	/// Before maturity the LP can cancel the other account's request; that returns the atom to the LP and
	/// restarts the LP's cooldown as well.
	function _t1CancelBeforeMaturityRestartsCooldownToo(T1 memory t) internal {
		vm.prank(LP);
		vm.expectRevert(TrancheVault.TrancheVault__ReceiverNotController.selector);
		juniorVault.cancelRedeemRequest(t.dustId, NOBODY, LP);
		vm.prank(LP);
		uint256 back = juniorVault.cancelRedeemRequest(t.dustId, LP, LP);
		assertEq(back, 1);
		assertEq(juniorVault.balanceOf(LP), t.lpShares + 1);
		assertEq(juniorVault.controllerRedeemHead(LP), 0);
		assertEq(juniorVault.lastDepositTime(LP), _now(), "cancelling the other account's request restarts the cooldown");
		assertEq(juniorVault.maxRequestRedeem(LP), 0);
		vm.prank(LP);
		_expectCooldown();
		juniorVault.requestRedeem(t.lpShares, LP, LP);
	}

	function _t1LpRequestIsFundedButStandardClaimsShowNothing(T1 memory t) internal {
		// keeper's regular settlement at the boundary of epoch e0+1: the 1-atom epoch quotes to zero assets
		(bool ok, IHousePool.LpEpochSettlementResult memory r) = _boundary(t.e0 + 1);
		assertTrue(ok);
		assertEq(r.juniorProcessedEpochs, 1);
		assertEq(r.juniorFundedShares, 0);
		assertEq(r.juniorFundedAssets, 0);
		assertTrue(juniorVault.redeemRefundPending(t.dustId, LP));
		assertEq(juniorVault.refundableRedeemRequest(t.dustId, LP), 1);
		assertEq(juniorVault.claimableRedeemRequest(t.dustId, LP), 0);
		assertEq(juniorVault.lastDepositTime(LP), t.lpLast0);

		_warpTo((t.e0 + 1) * HOUR + 300);
		_pushMark(P0);
		vm.prank(LP);
		t.lpRid = juniorVault.requestRedeem(t.half, LP, LP);
		assertEq(t.lpRid, t.e0 + 2);
		assertEq(juniorVault.controllerRedeemHead(LP), t.dustId, "the head stays on the other account's request");
		(ok, r) = _boundary(t.e0 + 2);
		assertTrue(ok);
		assertEq(r.juniorFundedShares, t.half);
		assertEq(r.juniorProcessedEpochs, 1);
		assertFalse(r.juniorBacklog);
		t.tFunded = _now();

		// funded and held in the vault's escrow for the LP ...
		t.funded = juniorVault.claimableRedeemAssets(t.lpRid, LP);
		console.log("T1: funded for the LP (atoms) / reference (atoms)", t.funded, t.refAssets);
		assertEq(t.funded, r.juniorFundedAssets);
		assertEq(t.funded, t.refAssets, "same funded amount as in the reference");
		assertEq(juniorVault.claimableRedeemRequest(t.lpRid, LP), t.half);
		assertEq(juniorVault.pendingRedeemRequest(t.lpRid, LP), 0);
		assertEq(juniorVault.withdrawalEscrowAssets(), t.funded);
		assertGe(_usdg(address(juniorVault)), t.funded);
		// ... but the standard claim limits and functions show nothing
		// EXPECTED (CANONICAL_ENTRYPOINTS.md:191-192): "claimed through ERC-4626 redeem or withdraw"
		assertEq(juniorVault.maxRedeem(LP), 0, "OBSERVED: maxRedeem(LP) == 0 with a funded request");
		assertEq(juniorVault.maxWithdraw(LP), 0, "OBSERVED: maxWithdraw(LP) == 0 with a funded request");
		vm.prank(LP);
		_expectMaxRedeem(LP, t.half, 0);
		juniorVault.redeem(t.half, LP, LP);
		vm.prank(LP);
		_expectMaxWithdraw(LP, t.funded, 0);
		juniorVault.withdraw(t.funded, LP, LP);
		vm.prank(LP);
		_expectMaxRedeem(LP, 1, 0);
		juniorVault.redeem(1, LP, LP);
		assertEq(_usdg(LP), 0);
		assertEq(juniorVault.balanceOf(LP), t.rest);
		assertEq(juniorVault.lastDepositTime(LP), t.lpLast0);
	}

	function _t1TimeDoesNotClearAndNobodyElseCan(T1 memory t) internal {
		address[3] memory others = [NOBODY, HELPER, JB];
		for (uint256 i; i < 3; ++i) {
			vm.prank(others[i]);
			_expectNotController();
			juniorVault.claimRedeemRefund(t.dustId, LP, LP);
		}
		vm.prank(HELPER);
		vm.expectRevert(TrancheVault.TrancheVault__RedeemRefundUnavailable.selector);
		juniorVault.claimRedeemRefund(t.dustId, HELPER, HELPER);
		_warpTo(t.tFunded + 24 * HOUR);
		assertEq(juniorVault.maxRedeem(LP), 0, "still 0 after 24 h");
		assertEq(juniorVault.maxWithdraw(LP), 0);
		assertEq(juniorVault.claimableRedeemAssets(t.lpRid, LP), t.funded);
		_warpTo(t.tFunded + 30 days);
		assertEq(juniorVault.maxRedeem(LP), 0, "still 0 after 30 days");
		assertEq(juniorVault.maxWithdraw(LP), 0);
		vm.prank(LP);
		_expectMaxRedeem(LP, t.half, 0);
		juniorVault.redeem(t.half, LP, LP);
		assertEq(juniorVault.claimableRedeemAssets(t.lpRid, LP), t.funded);
		assertEq(juniorVault.controllerRedeemHead(LP), t.dustId);
	}

	/// Way out 1: the claim by request id pays at once; nothing of the LP is delayed.
	function _t1WayOutClaimByRequestId(T1 memory t) internal {
		vm.prank(LP);
		uint256 got = juniorVault.claimRedeem(t.lpRid, t.half, LP, LP);
		console.log("T1 gas: claimRedeem(requestId, ...)", vm.lastCallGas().gasTotalUsed);
		assertEq(got, t.funded);
		assertEq(_now() - t.tFunded, 0, "TIME: the explicit-id path pays in the funding block, 0 s of delay");
		assertEq(_usdg(LP), t.funded);
		assertEq(juniorVault.lastDepositTime(LP), t.lpLast0, "no cooldown restart");
		assertEq(juniorVault.maxRequestRedeem(LP), t.rest);
		assertEq(juniorVault.controllerRedeemHead(LP), t.dustId, "the head stays where it was");
		vm.prank(LP);
		uint256 rid2 = juniorVault.requestRedeem(t.rest, LP, LP);
		assertEq(rid2, t.e0 + 3);
		(bool ok, IHousePool.LpEpochSettlementResult memory r) = _boundary(t.e0 + 3);
		assertTrue(ok);
		assertEq(r.juniorFundedShares, t.rest);
		assertEq(_now(), t.refRestFundedAt);
		assertEq(juniorVault.maxRedeem(LP), 0);
		vm.prank(LP);
		juniorVault.claimRedeem(rid2, t.rest, LP, LP);
		console.log("T1 claim-by-id: total paid (atoms) / reference total (atoms)", _usdg(LP), t.refTotal);
		assertEq(_usdg(LP), t.refTotal, "same total as the reference");
		assertEq(juniorVault.controllerRedeemHead(LP), t.dustId);
		assertEq(juniorVault.maxRedeem(LP), 0);
	}

	/// Way out 2: the LP requests its remaining shares FIRST and cleans the head afterwards.
	function _t1WayOutRequestFirstThenClean(T1 memory t) internal {
		vm.prank(LP);
		uint256 rid2 = juniorVault.requestRedeem(t.rest, LP, LP);
		assertEq(rid2, t.e0 + 3);
		vm.prank(LP);
		assertEq(juniorVault.claimRedeemRefund(t.dustId, LP, LP), 1);
		assertEq(juniorVault.lastDepositTime(LP), _now(), "cooldown restarted");
		assertEq(juniorVault.balanceOf(LP), 1, "only the other account's atom is in the wallet");
		assertEq(juniorVault.controllerRedeemHead(LP), t.lpRid);
		vm.prank(LP);
		assertEq(juniorVault.redeem(t.half, LP, LP), t.funded);
		(bool ok, IHousePool.LpEpochSettlementResult memory r) = _boundary(t.e0 + 3);
		assertTrue(ok);
		assertEq(r.juniorFundedShares, t.rest);
		assertEq(_now(), t.refRestFundedAt);
		assertEq(juniorVault.maxRedeem(LP), t.rest);
		vm.prank(LP);
		juniorVault.redeem(t.rest, LP, LP);
		console.log("T1 request-first: total paid (atoms) / reference total (atoms)", _usdg(LP), t.refTotal);
		assertEq(_usdg(LP), t.refTotal, "same total as the reference");
		assertEq(juniorVault.maxRequestRedeem(LP), 1, "the atom is free again exactly 3600 s after the cleanup");
	}

	/// Way out 3: the LP moves its remaining shares to a second wallet FIRST and cleans afterwards.
	function _t1WayOutTransferFirstThenClean(T1 memory t) internal {
		vm.prank(LP);
		juniorVault.transfer(LP2, t.rest);
		assertEq(juniorVault.lastDepositTime(LP2), t.lpLast0);
		vm.prank(LP);
		assertEq(juniorVault.claimRedeemRefund(t.dustId, LP, LP), 1);
		assertEq(juniorVault.lastDepositTime(LP), _now());
		assertEq(juniorVault.maxRequestRedeem(LP), 0);
		assertEq(juniorVault.maxRequestRedeem(LP2), t.rest);
		vm.prank(LP2);
		assertEq(juniorVault.requestRedeem(t.rest, LP2, LP2), t.e0 + 3);
		vm.prank(LP);
		assertEq(juniorVault.redeem(t.half, LP, LP), t.funded);
	}

	/// The LP cleans the head first (the call the interface comment asks for): the standard claims come back and
	/// the LP's whole remaining balance is in a new 1 h cooldown.
	function _t1CleanFirst(T1 memory t) internal {
		assertEq(juniorVault.maxRequestRedeem(LP), t.rest);
		vm.prank(LP);
		vm.expectRevert(TrancheVault.TrancheVault__ReceiverNotController.selector);
		juniorVault.claimRedeemRefund(t.dustId, NOBODY, LP);
		vm.prank(LP);
		uint256 refunded = juniorVault.claimRedeemRefund(t.dustId, LP, LP);
		console.log("T1 gas: claimRedeemRefund (cleanup)", vm.lastCallGas().gasTotalUsed);
		assertEq(refunded, 1, "the other account's atom lands with the LP");
		assertEq(juniorVault.balanceOf(LP), t.rest + 1);
		assertEq(juniorVault.controllerRedeemHead(LP), t.lpRid, "head now on the LP's own request");
		assertEq(juniorVault.maxRedeem(LP), t.half);
		assertEq(juniorVault.maxWithdraw(LP), t.funded);
		// IAsyncTrancheVault.sol:12-14: "... so share delivery cannot impose another account's cooldown"
		assertEq(juniorVault.lastDepositTime(LP), t.tFunded, "OBSERVED: the cleanup restarts the LP's cooldown");
		assertEq(juniorVault.maxRequestRedeem(LP), 0, "OBSERVED: request limit 0 for the LP's remaining shares");
		vm.prank(LP);
		_expectCooldown();
		juniorVault.requestRedeem(t.rest, LP, LP);
		vm.prank(LP);
		vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
		juniorVault.transfer(LP2, t.rest);
		vm.prank(LP);
		assertEq(juniorVault.redeem(t.half, LP, LP), t.funded);
		assertEq(_usdg(LP), t.funded);
		assertEq(_now() - t.tFunded, 0, "TIME: the funded half is paid in the funding block once the head is removed");

		// TIME (remaining shares, every path of a wallet holder: request for redemption, transfer): closed through
		// second 3599 after the cleanup
		_warpTo(t.tFunded + HOUR - 1);
		assertEq(_now() - t.tFunded, 3599);
		assertEq(juniorVault.maxRequestRedeem(LP), 0, "still 0 at 3599 s");
		vm.prank(LP);
		_expectCooldown();
		juniorVault.requestRedeem(t.rest, LP, LP);
		vm.prank(LP);
		vm.expectRevert(TrancheVault.TrancheVault__TransferDuringCooldown.selector);
		juniorVault.transfer(LP2, t.rest);
		// exactly 3600 s later = the next epoch boundary
		(bool ok,) = _boundary(t.e0 + 3);
		assertFalse(ok);
		assertEq(_now(), t.tFunded + HOUR);
		assertEq(_now(), t.refRestFundedAt);
		assertEq(juniorVault.maxRequestRedeem(LP), t.rest + 1, "free again after exactly 3600 s");
		vm.prank(LP);
		uint256 rid2 = juniorVault.requestRedeem(t.rest + 1, LP, LP);
		assertEq(rid2, t.e0 + 4, "OBSERVED: the remaining shares join epoch e0+4, one epoch later than the reference");
		IHousePool.LpEpochSettlementResult memory r;
		(ok, r) = _boundary(t.e0 + 4);
		assertTrue(ok);
		assertEq(r.juniorFundedShares, t.rest + 1);
		assertEq(_now() - t.refRestFundedAt, HOUR, "OBSERVED: funded 3600 s later than in the reference");
		vm.prank(LP);
		juniorVault.redeem(t.rest + 1, LP, LP);
		uint256 total = _usdg(LP);
		console.log("T1 clean-first: total paid (atoms) / reference total (atoms)", total, t.refTotal);
		assertEq(t.refTotal - total, 9);
		assertEq(juniorVault.balanceOf(LP), 0);
	}

	// Test 2: the repeated form over 26 epochs, with three ways the LP can behave.
	// The other account files a 1-atom request naming the LP 300 s into every epoch. The LP wants to exit
	// 1/30 of its position every hour.

	struct Run {
		uint256 reqOk;
		uint256 reqCooldown;
		uint256 stdClaims;
		uint256 idClaims;
		uint256 cleanups;
		uint256 stdBlockedWhileFunded;
		uint256 samples;
		uint256 samplesFundedButStdZero;
		uint256 samplesWalletButRequestZero;
		uint256 boundariesRequestFree;
		uint256 usdg;
		uint256 walletShares;
	}

	function test_S10_2_Repeated26Epochs_ThreeBehavioursAgainstReference() public {
		uint256 lpShares = juniorVault.balanceOf(LP);
		uint256 jbShares = juniorVault.balanceOf(JB);

		uint256 snap = vm.snapshotState();
		Run memory ref = _run26(MODE_REF, false);
		vm.revertToState(snap);
		snap = vm.snapshotState();
		Run memory a = _run26(MODE_A, true);
		assertEq(juniorVault.balanceOf(JB), jbShares - 27, "other account: 27 share atoms spent (one request per epoch, the last one still pending)");
		assertEq(_usdg(JB) + _usdg(HELPER), 0, "other account: no USDG in or out");
		vm.revertToState(snap);
		snap = vm.snapshotState();
		Run memory a2 = _run26(MODE_A2, true);
		vm.revertToState(snap);
		snap = vm.snapshotState();
		Run memory b = _run26(MODE_B, true);
		vm.revertToState(snap);

		_logRun("REF", ref);
		_logRun("A  (clean first)", a);
		_logRun("A' (request first)", a2);
		_logRun("B  (never clean, claim by id)", b);

		// reference: 26 requests, 25 funded and paid through redeem()
		assertEq(ref.reqOk, 26);
		assertEq(ref.reqCooldown, 0);
		assertEq(ref.stdClaims, 25);
		assertEq(ref.cleanups, 0);
		assertEq(ref.samplesFundedButStdZero, 0);
		assertEq(ref.samplesWalletButRequestZero, 0);
		assertGt(ref.usdg, 83_000e6, "REF: about 25/30 of 100,000 USDG paid in 26 h");
		assertEq(ref.walletShares, lpShares - 26 * (lpShares / 30));

		// A: an LP that cleans before it requests never gets a request through
		assertEq(a.reqOk, 0, "A: no request accepted in 26 epochs");
		assertEq(a.reqCooldown, 26, "A: every request reverts with DepositCooldown");
		assertEq(a.cleanups, 26);
		assertEq(a.stdClaims, 0);
		assertEq(a.usdg, 0, "A: nothing paid in 26 h");
		assertEq(a.walletShares, lpShares + 26, "A: all shares still in the wallet, plus 26 atoms");
		assertEq(a.samplesWalletButRequestZero, 78, "A: request limit 0 at +300, +1800, +3599 of every epoch");
		assertEq(a.boundariesRequestFree, 26, "A: at every boundary, before the LP's cleanup, the limit is free");

		// A': an LP that requests before it cleans is not held: same result as the reference
		assertEq(a2.reqOk, 26, "A': every request accepted");
		assertEq(a2.reqCooldown, 0);
		assertEq(a2.cleanups, 1, "A': one cleanup; later 1-atom requests merge into the LP's own positions");
		assertEq(a2.stdClaims, 25);
		assertEq(a2.usdg, ref.usdg, "A': same USDG as the reference");
		assertEq(a2.samplesWalletButRequestZero, 3, "A': request limit 0 only in the hour after the single cleanup");
		assertEq(a2.walletShares, ref.walletShares + 1, "A': the reference balance plus the first atom");
		assertEq(a2.samplesFundedButStdZero, 0);

		// B: an LP that never cleans and claims by request id is paid as in the reference; the standard limits
		//    are 0 at every boundary on which it has a funded claim
		assertEq(b.reqOk, 26);
		assertEq(b.reqCooldown, 0);
		assertEq(b.cleanups, 0);
		assertEq(b.stdClaims, 0);
		assertEq(b.idClaims, 25);
		assertEq(b.stdBlockedWhileFunded, 25, "B: redeem() reverted on each of the 25 funded claims");
		assertEq(b.samplesFundedButStdZero, 25);
		assertEq(b.usdg, ref.usdg, "B: same USDG as the reference");
		assertEq(b.walletShares, ref.walletShares);
		assertEq(b.samplesWalletButRequestZero, 0);
	}

	function _logRun(string memory name, Run memory s) internal view {
		console.log(name);
		console.log("  requests accepted / reverted DepositCooldown", s.reqOk, s.reqCooldown);
		console.log("  claims redeem() / claimRedeem(id) / cleanups", s.stdClaims, s.idClaims, s.cleanups);
		console.log("  redeem() reverts with a funded claim", s.stdBlockedWhileFunded);
		console.log("  samples / funded, maxRedeem 0 / shares, maxRequestRedeem 0", s.samples, s.samplesFundedButStdZero, s.samplesWalletButRequestZero);
		console.log("  boundaries with a free request limit", s.boundariesRequestFree);
		console.log("  USDG paid (atoms) / wallet shares", s.usdg, s.walletShares);
	}

	function _run26(uint8 mode, bool withDust) internal returns (Run memory s) {
		uint256 e0 = pool.currentLpEpoch();
		uint256 tranche = juniorVault.balanceOf(LP) / 30;
		if (withDust) {
			_dust(juniorVault, JB, HELPER, LP); // 300 s into e0: joins epoch e0+1
		}
		for (uint256 k = 1; k <= 26; ++k) {
			uint256 e = e0 + k;
			_boundary(e);
			_sample(s, true); // +0: after the keeper's settlement, before the LP acts
			_lpActs(mode, tranche, s);
			_warpTo(e * HOUR + 300);
			_sample(s, false);
			if (withDust) {
				_dust(juniorVault, JB, HELPER, LP); // joins epoch e+1
			}
			_warpTo(e * HOUR + 1800);
			_sample(s, false);
			_warpTo(e * HOUR + 3599);
			_sample(s, false);
		}
		s.usdg = _usdg(LP);
		s.walletShares = juniorVault.balanceOf(LP);
	}

	function _fundedOwn() internal view returns (uint256 assets) {
		for (uint256 i; i < _ownIds.length; ++i) {
			assets += juniorVault.claimableRedeemAssets(_ownIds[i], LP);
		}
	}

	function _sample(Run memory s, bool atBoundary) internal {
		s.samples += 1;
		if (_fundedOwn() != 0 && juniorVault.maxRedeem(LP) == 0) {
			assertEq(juniorVault.maxWithdraw(LP), 0);
			s.samplesFundedButStdZero += 1;
		}
		bool requestZero = juniorVault.balanceOf(LP) != 0 && juniorVault.maxRequestRedeem(LP) == 0;
		if (requestZero) {
			s.samplesWalletButRequestZero += 1;
		}
		if (atBoundary && !requestZero) {
			s.boundariesRequestFree += 1;
		}
	}

	function _lpActs(uint8 mode, uint256 tranche, Run memory s) internal {
		if (mode == MODE_A) {
			_cleanAndStandardClaim(s);
			_tryRequest(tranche, s);
		} else if (mode == MODE_B) {
			_tryRequest(tranche, s);
			_claimByIdOnly(s);
		} else {
			_tryRequest(tranche, s);
			_cleanAndStandardClaim(s);
		}
	}

	function _tryRequest(uint256 tranche, Run memory s) internal {
		vm.prank(LP);
		(bool ok, bytes memory ret) =
			address(juniorVault).call(abi.encodeCall(juniorVault.requestRedeem, (tranche, LP, LP)));
		if (ok) {
			s.reqOk += 1;
			_ownIds.push(abi.decode(ret, (uint256)));
		} else {
			assertEq(bytes4(ret), TrancheVault.TrancheVault__DepositCooldown.selector, "only the cooldown stops it");
			s.reqCooldown += 1;
		}
	}

	function _cleanAndStandardClaim(Run memory s) internal {
		for (uint256 guard; guard < 64; ++guard) {
			uint256 h = juniorVault.controllerRedeemHead(LP);
			if (h == 0) {
				break;
			}
			if (juniorVault.redeemRefundPending(h, LP)) {
				vm.prank(LP);
				juniorVault.claimRedeemRefund(h, LP, LP);
				s.cleanups += 1;
				continue;
			}
			uint256 m = juniorVault.maxRedeem(LP);
			if (m == 0) {
				break;
			}
			vm.prank(LP);
			juniorVault.redeem(m, LP, LP);
			s.stdClaims += 1;
		}
	}

	function _claimByIdOnly(Run memory s) internal {
		for (uint256 i; i < _ownIds.length; ++i) {
			uint256 c = juniorVault.claimableRedeemRequest(_ownIds[i], LP);
			if (c == 0) {
				continue;
			}
			assertEq(juniorVault.maxRedeem(LP), 0);
			vm.prank(LP);
			_expectMaxRedeem(LP, c, 0);
			juniorVault.redeem(c, LP, LP);
			s.stdBlockedWhileFunded += 1;
			vm.prank(LP);
			juniorVault.claimRedeem(_ownIds[i], c, LP, LP);
			s.idClaims += 1;
		}
	}

	// Test 3: no cap on open requests in a controller's queue. 30 epochs, each with one 1-atom request naming
	// the LP; the LP has no request of its own in those epochs.
	function test_S10_3_ThirtyEpochsNamingLp_ThirtyCleanupCalls() public {
		uint256 lpShares = juniorVault.balanceOf(LP);
		uint256 half = lpShares / 2;
		uint256 e0 = pool.currentLpEpoch();
		uint256 jbShares = juniorVault.balanceOf(JB);
		uint256 lpLast0 = juniorVault.lastDepositTime(LP);

		uint256 snap = vm.snapshotState();
		uint256 refAssets;
		{
			_boundary(e0 + 30);
			vm.prank(LP);
			uint256 rid = juniorVault.requestRedeem(half, LP, LP);
			assertEq(rid, e0 + 31);
			_boundary(e0 + 31);
			assertEq(juniorVault.maxRedeem(LP), half, "REF: maxRedeem == funded shares");
			refAssets = juniorVault.maxWithdraw(LP);
			vm.prank(LP);
			assertEq(juniorVault.redeem(half, LP, LP), refAssets);
		}
		vm.revertToState(snap);

		// two 1-atom requests in the SAME epoch share one queue entry
		uint256 first = _dust(juniorVault, JB, HELPER, LP);
		assertEq(_dust(juniorVault, JB, HELPER2, LP), first);
		assertEq(juniorVault.pendingRedeemRequest(first, LP), 2);
		// 29 more epochs, one 1-atom request each; the keeper settles every boundary (one refund epoch per call)
		for (uint256 k = 1; k <= 30; ++k) {
			(bool ok, IHousePool.LpEpochSettlementResult memory r) = _boundary(e0 + k);
			assertTrue(ok);
			assertEq(r.juniorProcessedEpochs, 1);
			assertFalse(r.juniorBacklog);
			if (k < 30) {
				_warpTo((e0 + k) * HOUR + 300);
				assertEq(_dust(juniorVault, JB, HELPER, LP), e0 + k + 1);
			}
		}
		assertEq(juniorVault.balanceOf(JB), jbShares - 31, "other account: 31 share atoms");
		assertEq(_usdg(JB) + _usdg(HELPER) + _usdg(HELPER2), 0);
		assertEq(juniorVault.controllerRedeemHead(LP), e0 + 1);
		assertEq(juniorVault.controllerRedeemTail(LP), e0 + 30);
		assertEq(juniorVault.lastDepositTime(LP), lpLast0);

		vm.prank(LP);
		uint256 lpRid = juniorVault.requestRedeem(half, LP, LP);
		assertEq(lpRid, e0 + 31);
		(bool ok2, IHousePool.LpEpochSettlementResult memory r2) = _boundary(e0 + 31);
		assertTrue(ok2);
		assertEq(r2.juniorFundedShares, half);
		uint256 funded = juniorVault.claimableRedeemAssets(lpRid, LP);
		assertEq(funded, r2.juniorFundedAssets);
		console.log("T3: funded for the LP (atoms) / reference (atoms)", funded, refAssets);
		assertApproxEqAbs(funded, refAssets, 100, "same funded amount as the reference within 100 atoms");
		assertEq(juniorVault.maxRedeem(LP), 0);
		assertEq(juniorVault.maxWithdraw(LP), 0);

		snap = vm.snapshotState();
		vm.prank(LP);
		assertEq(juniorVault.claimRedeem(lpRid, half, LP, LP), funded);
		assertEq(juniorVault.lastDepositTime(LP), lpLast0);
		vm.revertToState(snap);

		// the standard claims come back only after every one of the 30 entries is removed
		uint256 calls;
		uint256 gasMin = type(uint256).max;
		uint256 gasMax;
		uint256 gasSum;
		while (juniorVault.maxRedeem(LP) == 0) {
			uint256 h = juniorVault.controllerRedeemHead(LP);
			assertTrue(juniorVault.redeemRefundPending(h, LP));
			vm.prank(LP);
			_expectMaxRedeem(LP, half, 0);
			juniorVault.redeem(half, LP, LP);
			vm.prank(LP);
			juniorVault.claimRedeemRefund(h, LP, LP);
			uint256 g = vm.lastCallGas().gasTotalUsed;
			gasSum += g;
			if (g < gasMin) gasMin = g;
			if (g > gasMax) gasMax = g;
			calls += 1;
			require(calls <= 64, "guard");
		}
		console.log("T3: cleanup calls needed / gas sum", calls, gasSum);
		console.log("T3: cleanup gas per call min / max", gasMin, gasMax);
		assertEq(calls, 30, "OBSERVED: 30 claimRedeemRefund calls of the LP");
		assertEq(juniorVault.balanceOf(LP), lpShares - half + 31);
		assertEq(juniorVault.lastDepositTime(LP), _now(), "cooldown restarted by the cleanup");
		assertEq(juniorVault.maxRequestRedeem(LP), 0);
		assertEq(juniorVault.maxRedeem(LP), half);
		vm.prank(LP);
		assertEq(juniorVault.redeem(half, LP, LP), funded);
		assertEq(_usdg(LP), funded);
	}

	// Test 4: a holder contract that uses only the standard functions.
	//   H1: requestDeposit / deposit / requestRedeem / redeem / withdraw.
	//   H2: the same plus setOperator.
	//   H3: as H1, no request of the other account (reference, same blocks).
	// The other account names H1 and H2 BEFORE they hold anything.

	struct T4 {
		StdOnlyHolder h1;
		StdHolderWithOperator h2;
		StdOnlyHolder h3;
		uint256 e0;
		uint256 dustId;
		uint256 exitId;
		uint256 shares1;
		uint256 shares2;
		uint256 funded1;
		uint256 funded2;
		uint256 tFunded;
	}

	function test_S10_4_StandardOnlyHolderContract_FundedClaimUnreachable() public {
		T4 memory t;
		t.e0 = pool.currentLpEpoch();
		t.h1 = new StdOnlyHolder(juniorVault, IERC20(address(settlementToken)), BEN);
		t.h2 = new StdHolderWithOperator(juniorVault, IERC20(address(settlementToken)), BEN);
		t.h3 = new StdOnlyHolder(juniorVault, IERC20(address(settlementToken)), BEN);
		uint256 jbShares = juniorVault.balanceOf(JB);

		// the other account names the two holder addresses before they hold anything: 2 transactions each
		t.dustId = _dust(juniorVault, JB, HELPER, address(t.h1));
		assertEq(_dust(juniorVault, JB, HELPER, address(t.h2)), t.dustId);
		assertEq(t.dustId, t.e0 + 1);
		assertEq(juniorVault.controllerRedeemHead(address(t.h1)), t.dustId);
		assertEq(juniorVault.controllerRedeemHead(address(t.h2)), t.dustId);
		assertEq(juniorVault.controllerRedeemHead(address(t.h3)), 0);
		assertEq(juniorVault.balanceOf(JB), jbShares - 2);

		// the holders enter through the standard async path
		_t4Enter(t);
		_t4RequestExitAndFund(t);

		vm.prank(BEN);
		uint256 paid3 = t.h3.claimExitByShares();
		assertEq(_usdg(BEN), paid3, "REF holder: paid by redeem(maxRedeem)");
		assertGt(paid3, 49_990e6);

		// H1 and H2: funded, and the standard functions cannot reach it
		_t4AssertUnreachable(t, address(t.h1), t.funded1);
		_t4AssertUnreachable(t, address(t.h2), t.funded2);
		assertEq(_usdg(BEN), paid3);

		// nobody else can clear the head or claim for the holder
		vm.prank(NOBODY);
		_expectNotController();
		juniorVault.claimRedeemRefund(t.dustId, address(t.h1), address(t.h1));
		vm.prank(BEN);
		_expectNotController();
		juniorVault.claimRedeem(t.exitId, t.shares1, BEN, address(t.h1));
		vm.prank(BEN);
		_expectNotController();
		juniorVault.claimRedeemRefund(t.dustId, address(t.h1), address(t.h1));

		// TIME: 86,400 s and 30 days after funding, every payout path of the holders is still closed
		_warpTo(t.tFunded + 24 * HOUR);
		_t4AssertEveryPayoutPathClosed(t, address(t.h1), t.funded1, 86_400, paid3);
		_t4AssertEveryPayoutPathClosed(t, address(t.h2), t.funded2, 86_400, paid3);
		_warpTo(t.tFunded + 30 days);
		_t4AssertEveryPayoutPathClosed(t, address(t.h1), t.funded1, 30 * 86_400, paid3);
		_t4AssertEveryPayoutPathClosed(t, address(t.h2), t.funded2, 30 * 86_400, paid3);
		assertEq(_usdg(BEN), paid3);
		console.log("T4: funded USDG of H1 / H2 not reachable by the standard functions after 30 days (atoms)", t.funded1, t.funded2);

		// way out of H2 (a): approve an operator; the operator claims by request id
		uint256 snap = vm.snapshotState();
		vm.prank(BEN);
		t.h2.approveOperator(OP, true);
		vm.prank(OP);
		assertEq(juniorVault.claimRedeem(t.exitId, t.shares2, BEN, address(t.h2)), t.funded2);
		assertEq(_usdg(BEN), paid3 + t.funded2);
		vm.revertToState(snap);

		// way out of H2 (b): the operator removes the head; then the standard claim pays
		vm.prank(BEN);
		t.h2.approveOperator(OP, true);
		vm.prank(OP);
		vm.expectRevert(TrancheVault.TrancheVault__ReceiverNotController.selector);
		juniorVault.claimRedeemRefund(t.dustId, OP, address(t.h2));
		vm.prank(OP);
		assertEq(juniorVault.claimRedeemRefund(t.dustId, address(t.h2), address(t.h2)), 1);
		assertEq(juniorVault.lastDepositTime(address(t.h2)), _now(), "the holder's cooldown restarts");
		assertEq(juniorVault.balanceOf(address(t.h2)), 1);
		vm.prank(BEN);
		assertEq(t.h2.claimExitByShares(), t.funded2);
		assertEq(_usdg(BEN), paid3 + t.funded2);

		// H1 has no such call: its funded USDG is still in the vault's escrow
		_t4AssertUnreachable(t, address(t.h1), t.funded1);
		assertGe(juniorVault.withdrawalEscrowAssets(), t.funded1);
		assertGe(_usdg(address(juniorVault)), t.funded1);
	}

	function _t4Enter(T4 memory t) internal {
		address[3] memory hs = [address(t.h1), address(t.h2), address(t.h3)];
		for (uint256 i; i < 3; ++i) {
			settlementToken.mint(hs[i], 50_000e6);
			vm.prank(BEN);
			assertEq(StdOnlyHolder(hs[i]).requestEntry(50_000e6), t.e0 + 1);
		}
		(bool ok, IHousePool.LpEpochSettlementResult memory r) = _boundary(t.e0 + 1);
		assertTrue(ok);
		assertEq(r.juniorDepositAssets, 150_000e6);
		assertEq(r.juniorProcessedEpochs, 1);
		assertEq(r.juniorFundedShares, 0);
		vm.startPrank(BEN);
		t.shares1 = t.h1.claimEntry();
		t.shares2 = t.h2.claimEntry();
		uint256 shares3 = t.h3.claimEntry();
		vm.stopPrank();
		assertEq(t.shares1, shares3);
		assertEq(t.shares2, shares3);
		assertEq(juniorVault.balanceOf(address(t.h1)), t.shares1);
		assertEq(juniorVault.lastDepositTime(address(t.h1)), _now());
	}

	function _t4RequestExitAndFund(T4 memory t) internal {
		_boundary(t.e0 + 2); // 3600 s after activation: the holders' cooldown is over
		vm.startPrank(BEN);
		t.exitId = t.h1.requestExit(t.shares1);
		assertEq(t.h2.requestExit(t.shares2), t.exitId);
		assertEq(t.h3.requestExit(t.shares1), t.exitId);
		vm.stopPrank();
		assertEq(t.exitId, t.e0 + 3);
		assertEq(juniorVault.controllerRedeemHead(address(t.h1)), t.dustId, "H1: head is the other account's request");
		assertEq(juniorVault.controllerRedeemHead(address(t.h3)), t.exitId, "REF holder: head is its own request");
		(bool ok, IHousePool.LpEpochSettlementResult memory r) = _boundary(t.e0 + 3);
		assertTrue(ok);
		assertEq(r.juniorFundedShares, 3 * t.shares1);
		t.tFunded = _now();
		t.funded1 = juniorVault.claimableRedeemAssets(t.exitId, address(t.h1));
		t.funded2 = juniorVault.claimableRedeemAssets(t.exitId, address(t.h2));
		assertGt(t.funded1, 49_990e6);
		assertEq(t.funded2, t.funded1);
		assertEq(juniorVault.claimableRedeemAssets(t.exitId, address(t.h3)), t.funded1);
	}

	/// @dev Every payout path of holder `h`, at least `minElapsed` s after funding: the two standard claims through
	///      the contract; explicit-id claim, refund call and cancellation sent by its owner. Nothing was paid.
	function _t4AssertEveryPayoutPathClosed(T4 memory t, address h, uint256 funded, uint256 minElapsed, uint256 benExpected) internal {
		assertGe(_now() - t.tFunded, minElapsed, "TIME: seconds since the exit was funded");
		_t4AssertUnreachable(t, h, funded);
		vm.prank(BEN);
		_expectNotController();
		juniorVault.claimRedeem(t.exitId, t.shares1, BEN, h);
		vm.prank(BEN);
		_expectNotController();
		juniorVault.claimRedeemRefund(t.dustId, h, h);
		vm.prank(BEN);
		_expectNotController();
		juniorVault.cancelRedeemRequest(t.exitId, h, h);
		assertFalse(juniorVault.isOperator(h, BEN));
		assertFalse(juniorVault.isOperator(h, OP));
		assertEq(_usdg(BEN), benExpected, "TIME: nothing was paid to the holder's beneficiary");
		assertEq(_usdg(h), 0);
	}

	function _t4AssertUnreachable(T4 memory t, address h, uint256 funded) internal {
		assertEq(juniorVault.claimableRedeemAssets(t.exitId, h), funded);
		assertEq(juniorVault.maxRedeem(h), 0, "OBSERVED: maxRedeem(holder) == 0");
		assertEq(juniorVault.maxWithdraw(h), 0, "OBSERVED: maxWithdraw(holder) == 0");
		vm.prank(BEN);
		_expectMaxRedeem(h, 0, 0);
		StdOnlyHolder(h).claimExitByShares();
		vm.prank(BEN);
		_expectMaxWithdraw(h, 0, 0);
		StdOnlyHolder(h).claimExitByAssets();
	}

	// Test 5: the deposit side. Another account funds a 1 USDG deposit request (the minimum) naming an address
	// with no position, one epoch before that address deposits itself.
	function test_S10_5_DepositSide_OneExtraStandardClaim() public {
		uint256 e0 = pool.currentLpEpoch();
		uint256 minAssets = pool.minTrancheDepositAssets();
		assertEq(minAssets, 1e6);

		uint256 snap = vm.snapshotState();
		uint256 refShares;
		uint256 refLast;
		{
			_warpTo((e0 + 1) * HOUR + 300);
			_pushMark(P0);
			uint256 id = _reqDep(juniorVault, NEWLP, 10_000e6);
			assertEq(id, e0 + 2);
			_boundary(e0 + 2);
			assertEq(juniorVault.maxDeposit(NEWLP), 10_000e6, "REF: maxDeposit == own lot");
			vm.prank(NEWLP);
			refShares = juniorVault.deposit(10_000e6, NEWLP);
			refLast = juniorVault.lastDepositTime(NEWLP);
			assertEq(refLast, (e0 + 2) * HOUR);
		}
		vm.revertToState(snap);

		// the other account: 1 USDG, controller = NEWLP
		settlementToken.mint(OTHER, 1e6);
		vm.startPrank(OTHER);
		settlementToken.approve(address(juniorVault), 1e6);
		vm.expectRevert(TrancheVault.TrancheVault__DepositTooSmall.selector);
		juniorVault.requestDeposit(1e6 - 1, NEWLP, OTHER);
		uint256 gid = juniorVault.requestDeposit(1e6, NEWLP, OTHER);
		vm.stopPrank();
		assertEq(gid, e0 + 1);
		assertEq(_usdg(OTHER), 0, "other account: -1 USDG");
		assertEq(juniorVault.controllerDepositHead(NEWLP), gid);
		vm.prank(OTHER);
		_expectNotController();
		juniorVault.cancelPendingDeposit(gid, OTHER, NEWLP);
		_boundary(e0 + 1);

		_warpTo((e0 + 1) * HOUR + 300);
		_pushMark(P0);
		uint256 own = _reqDep(juniorVault, NEWLP, 10_000e6);
		assertEq(own, e0 + 2);
		_boundary(e0 + 2);
		assertEq(juniorVault.claimableDepositRequest(own, NEWLP), 10_000e6);
		assertEq(juniorVault.maxDeposit(NEWLP), 1e6, "OBSERVED: maxDeposit == the 1 USDG lot of the other account");
		vm.prank(NEWLP);
		vm.expectRevert(abi.encodeWithSelector(ERC4626.ERC4626ExceededMaxDeposit.selector, NEWLP, 10_000e6, 1e6));
		juniorVault.deposit(10_000e6, NEWLP);
		// one extra STANDARD call clears it; the lot is a gift
		vm.prank(NEWLP);
		uint256 giftShares = juniorVault.deposit(1e6, NEWLP);
		assertGt(giftShares, 0);
		assertEq(juniorVault.lastDepositTime(NEWLP), (e0 + 1) * HOUR);
		assertEq(juniorVault.maxDeposit(NEWLP), 10_000e6);
		vm.prank(NEWLP);
		uint256 ownShares = juniorVault.deposit(10_000e6, NEWLP);
		assertEq(ownShares, refShares);
		assertEq(juniorVault.lastDepositTime(NEWLP), refLast, "OBSERVED: same cooldown timestamp as the reference");
		assertEq(juniorVault.balanceOf(NEWLP), refShares + giftShares);
		assertEq(_usdg(OTHER), 0);
		console.log("T5: gift shares to the named address / own shares", giftShares, ownShares);
	}

	// Test 6: the same redemption form on the Senior vault.

	struct T6 {
		uint256 e0;
		uint256 sShares;
		uint256 half;
		uint256 dustId;
		uint256 rid;
		uint256 funded;
		uint256 refFunded;
		uint256 last0;
	}

	function test_S10_6_SeniorVault_SameForm() public {
		T6 memory t;
		t.e0 = pool.currentLpEpoch();
		// Senior book: SLP 50,000, SB 1,000
		uint256 a = _reqDep(seniorVault, SLP, 50_000e6);
		assertEq(_reqDep(seniorVault, SB, 1000e6), a);
		(bool ok, IHousePool.LpEpochSettlementResult memory r) = _boundary(t.e0 + 1);
		assertTrue(ok);
		assertEq(r.seniorDepositAssets, 51_000e6);
		t.sShares = _claimDep(seniorVault, SLP, a);
		_claimDep(seniorVault, SB, a);
		t.half = t.sShares / 2;
		t.last0 = seniorVault.lastDepositTime(SLP);
		_warpTo((t.e0 + 2) * HOUR + 300); // past the Senior holders' cooldown
		_pushMark(P0);

		uint256 snap = vm.snapshotState();
		{
			_warpTo((t.e0 + 3) * HOUR + 300);
			_pushMark(P0);
			vm.prank(SLP);
			uint256 rid = seniorVault.requestRedeem(t.half, SLP, SLP);
			assertEq(rid, t.e0 + 4);
			(ok, r) = _boundary(t.e0 + 4);
			assertTrue(ok);
			assertEq(r.seniorFundedShares, t.half);
			assertEq(seniorVault.maxRedeem(SLP), t.half, "REF: maxRedeem == funded shares");
			t.refFunded = seniorVault.maxWithdraw(SLP);
			vm.prank(SLP);
			assertEq(seniorVault.redeem(t.half, SLP, SLP), t.refFunded);
		}
		vm.revertToState(snap);

		// the other account: 2 transactions on the Senior vault
		uint256 sbShares = seniorVault.balanceOf(SB);
		t.dustId = _dust(seniorVault, SB, HELPER, SLP);
		assertEq(t.dustId, t.e0 + 3);
		assertEq(seniorVault.balanceOf(SB), sbShares - 1);
		assertEq(seniorVault.controllerRedeemHead(SLP), t.dustId);
		(ok, r) = _boundary(t.e0 + 3);
		assertTrue(ok);
		assertEq(r.seniorProcessedEpochs, 1);
		assertEq(r.seniorFundedShares, 0);
		assertTrue(seniorVault.redeemRefundPending(t.dustId, SLP));

		_warpTo((t.e0 + 3) * HOUR + 300);
		_pushMark(P0);
		vm.prank(SLP);
		t.rid = seniorVault.requestRedeem(t.half, SLP, SLP);
		assertEq(t.rid, t.e0 + 4);
		(ok, r) = _boundary(t.e0 + 4);
		assertTrue(ok);
		assertEq(r.seniorFundedShares, t.half);
		t.funded = seniorVault.claimableRedeemAssets(t.rid, SLP);
		console.log("T6: funded for the Senior LP (atoms) / reference (atoms)", t.funded, t.refFunded);
		assertApproxEqAbs(t.funded, t.refFunded, 100, "same funded amount as the reference within 100 atoms");
		assertGt(t.funded, 24_990e6);
		assertEq(seniorVault.maxRedeem(SLP), 0, "OBSERVED: maxRedeem(SLP) == 0 with a funded request");
		assertEq(seniorVault.maxWithdraw(SLP), 0, "OBSERVED: maxWithdraw(SLP) == 0 with a funded request");
		vm.prank(SLP);
		_expectMaxRedeem(SLP, t.half, 0);
		seniorVault.redeem(t.half, SLP, SLP);
		vm.prank(SLP);
		_expectMaxWithdraw(SLP, t.funded, 0);
		seniorVault.withdraw(t.funded, SLP, SLP);

		snap = vm.snapshotState();
		vm.prank(SLP);
		assertEq(seniorVault.claimRedeem(t.rid, t.half, SLP, SLP), t.funded);
		assertEq(seniorVault.lastDepositTime(SLP), t.last0);
		vm.revertToState(snap);

		// cleanup restarts the Senior LP's cooldown on its remaining shares
		uint256 tClean = _now();
		vm.prank(SLP);
		assertEq(seniorVault.claimRedeemRefund(t.dustId, SLP, SLP), 1);
		assertEq(seniorVault.lastDepositTime(SLP), tClean, "OBSERVED: cooldown restarted");
		assertEq(seniorVault.maxRequestRedeem(SLP), 0);
		vm.prank(SLP);
		_expectCooldown();
		seniorVault.requestRedeem(t.sShares - t.half, SLP, SLP);
		vm.prank(SLP);
		assertEq(seniorVault.redeem(t.half, SLP, SLP), t.funded);
		assertEq(_usdg(SLP), t.funded);
		_warpTo(tClean + HOUR - 1);
		assertEq(seniorVault.maxRequestRedeem(SLP), 0, "still 0 at 3599 s");
		_warpTo(tClean + HOUR);
		assertEq(seniorVault.maxRequestRedeem(SLP), t.sShares - t.half + 1, "free after exactly 3600 s");
	}

	// Test 7: the weaker variant. The 1-atom request sits in an epoch that also holds a real request, so the
	// epoch is funded, not refunded. A standard call clears it and the cooldown is not touched.
	function test_S10_7_OneAtomInAFundedEpoch_StandardRedeemClearsIt() public {
		uint256 lpShares = juniorVault.balanceOf(LP);
		uint256 half = lpShares / 2;
		uint256 e0 = pool.currentLpEpoch();
		uint256 jbShares = juniorVault.balanceOf(JB);
		uint256 dustId = _dust(juniorVault, JB, HELPER, LP);
		vm.prank(JB);
		juniorVault.requestRedeem(jbShares - 1, JB, JB); // a real request in the same epoch
		_boundary(e0 + 1);
		assertFalse(juniorVault.redeemRefundPending(dustId, LP));
		_warpTo((e0 + 1) * HOUR + 300);
		_pushMark(P0);
		vm.prank(LP);
		uint256 lpRid = juniorVault.requestRedeem(half, LP, LP);
		_boundary(e0 + 2);
		uint256 funded = juniorVault.claimableRedeemAssets(lpRid, LP);
		assertGt(funded, 49_990e6);
		assertEq(juniorVault.maxRedeem(LP), 1, "OBSERVED: only the other account's atom is visible");
		assertEq(juniorVault.maxWithdraw(LP), 0, "OBSERVED: maxWithdraw(LP) == 0 with a funded request");
		vm.prank(LP);
		_expectMaxRedeem(LP, half, 1);
		juniorVault.redeem(half, LP, LP);
		vm.prank(LP);
		_expectMaxWithdraw(LP, funded, 0);
		juniorVault.withdraw(funded, LP, LP);
		uint256 lastBefore = juniorVault.lastDepositTime(LP);
		// a standard call clears it: redeem(maxRedeem) pays 0 for the atom and moves the head
		vm.prank(LP);
		assertEq(juniorVault.redeem(1, LP, LP), 0);
		assertEq(juniorVault.lastDepositTime(LP), lastBefore, "no cooldown restart on this path");
		assertEq(juniorVault.controllerRedeemHead(LP), lpRid);
		assertEq(juniorVault.maxRedeem(LP), half);
		assertEq(juniorVault.maxWithdraw(LP), funded);
		vm.prank(LP);
		assertEq(juniorVault.redeem(half, LP, LP), funded);
		assertEq(juniorVault.maxRequestRedeem(LP), lpShares - half);
	}

	// Test 8 (control of the condition): a 1-atom request of the other account in the SAME epoch as the LP's
	// request merges into the LP's position, and one in a LATER epoch sits behind it. Neither hides the LP's
	// funded claim. Only a request in an EARLIER epoch does (tests 1-4, 6).
	function test_S10_8_Control_SameOrLaterEpoch_StandardClaimPays() public {
		uint256 lpShares = juniorVault.balanceOf(LP);
		uint256 half = lpShares / 2;
		uint256 e0 = pool.currentLpEpoch();
		vm.prank(LP);
		uint256 lpRid = juniorVault.requestRedeem(half, LP, LP);
		assertEq(lpRid, e0 + 1);
		assertEq(_dust(juniorVault, JB, HELPER, LP), lpRid);
		assertEq(juniorVault.pendingRedeemRequest(lpRid, LP), half + 1);
		assertEq(juniorVault.controllerRedeemHead(LP), lpRid);
		assertEq(juniorVault.controllerRedeemTail(LP), lpRid);
		(bool ok, IHousePool.LpEpochSettlementResult memory r) = _boundary(e0 + 1);
		assertTrue(ok);
		assertEq(r.juniorFundedShares, half + 1);
		_warpTo((e0 + 1) * HOUR + 300);
		assertEq(_dust(juniorVault, JB, HELPER, LP), e0 + 2);
		assertEq(juniorVault.controllerRedeemHead(LP), lpRid, "head is still the LP's own request");
		assertEq(juniorVault.controllerRedeemTail(LP), e0 + 2);
		assertEq(juniorVault.maxRedeem(LP), half + 1);
		uint256 funded = juniorVault.maxWithdraw(LP);
		assertGt(funded, 49_990e6);
		vm.prank(LP);
		assertEq(juniorVault.redeem(half + 1, LP, LP), funded);
		assertEq(_usdg(LP), funded);
		// the later 1-atom epoch then becomes the head; the LP has no funded claim behind it at this point
		(ok, r) = _boundary(e0 + 2);
		assertTrue(ok);
		assertEq(juniorVault.controllerRedeemHead(LP), e0 + 2);
		assertTrue(juniorVault.redeemRefundPending(e0 + 2, LP));
		assertEq(juniorVault.maxRedeem(LP), 0);
		assertEq(juniorVault.maxRequestRedeem(LP), lpShares - half);
	}

}
```
