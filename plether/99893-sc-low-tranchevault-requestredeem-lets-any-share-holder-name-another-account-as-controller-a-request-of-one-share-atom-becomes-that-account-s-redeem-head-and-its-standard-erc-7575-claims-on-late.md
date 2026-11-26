# \#99893 [SC-Low] TrancheVault.requestRedeem lets any share holder name another account as controller: a request of one share atom becomes that account's redeem head, and its standard ERC-7575 claims on later funded lots report 0 and revert

**Submitted on Nov 15th 2026 at 17:01:26 UTC by @mhkarazeybek for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99893
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
`TrancheVault.requestRedeem(shares, controller, owner)` authorises only the share owner. Any address can be named as `controller` without its consent. A stranger that holds one share atom files a request with `controller = victim`. When settlement refunds that dust-only epoch, the lot stays at the front of the victim's own redeem queue (`controllerRedeemHead`). The standard claim surface (`maxRedeem`, `maxWithdraw`, `redeem`, `withdraw`) reads only that head, so the victim's later funded lots report `claimableRedeemRequest > 0` while `maxRedeem == maxWithdraw == 0` and `redeem` / `withdraw` revert. Only the victim or an operator it approved can move the head, and only through the vault's custom call `claimRedeemRefund`, which starts a 3,600 s lock on the victim's wallet shares. In the PoC an ordinary account loses the standard claims on a funded lot of 2,499.76 USDG until it makes that call. A holder contract that uses only the standard surface (a test contract of ours; the protocol deploys none) is locked out by one plant: 0 of 719 claims succeed over 720 hourly keeper passes and 30 of 30 funded lots stay unreachable. The attacker spends one share atom and gas, and gains nothing. Nothing is lost for a holder that can call the custom explicit-id claim.

## Vulnerability Details
Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.

Site 1, the request. Only `owner` is authenticated; `controller` only has to be nonzero.

```solidity
// TrancheVault.sol:606-632, requestRedeem (trimmed and reflowed)
if (controller == address(0) || owner == address(0)) revert TrancheVault__ZeroAddress();   // :611
...
if (msg.sender != owner && !isOperator[owner][msg.sender]) {   // :625
    _spendAllowance(owner, msg.sender, shares);
}
_transfer(owner, address(this), shares);
requestId = _enqueueRedeemRequest(shares, controller);          // :630
```

`_enqueueRedeemRequest` appends the epoch to the named controller's queue (`:701-704`, `_appendControllerRedeem` at `:1783-1797`). If that queue is empty, the new lot is its head (`:1791`).

Site 2, the standard claim surface reads only the head.

```solidity
// TrancheVault.sol:1306-1311 (reflowed)
function maxRedeem(address controller) public view override returns (uint256) {
    uint256 requestId = controllerRedeemHead[controller];
    return requestId == 0 ? 0 : claimableRedeemRequest(requestId, controller);
}
```

`maxWithdraw` is the same (`:1299-1304`). `redeem` and `withdraw` pass `controllerRedeemHead[controller]` to the claim routines (`:1003`, `:1011`), which revert when the request exceeds what the head holds (`:1160`, `:1181`).

Site 3, who can move the head. The head is advanced only in `_removeControllerRedeem` (`:1799-1822`, write at `:1810`). It is reached from `cancelRedeemRequest` (`:904`) and from `_maybeCloseRedeemPosition` (`:1225`). For a refunded epoch the second needs `position.refundClaimed` (`:1223`), which only `claimRedeemRefund` sets (`:932`). Both entrypoints start with `_requireControllerAuth(controller)` (`:889`, `:921`; check at `:1603-1609`), and operator approvals are written only for `msg.sender` (`:309`). So a refunded lot at the head can be removed by exactly one call: `claimRedeemRefund(junkId, controller, controller)` sent by the controller or an operator the controller approved.

How the junk lot becomes a refunded head.

1. The stranger moves one share atom to a fresh account and files `requestRedeem(1, victim, thatAccount)`. A request equal to the owner's full balance passes the minimum size check (`:1611-1618`).
2. An epoch that holds only this atom quotes zero assets, so the pool refunds it instead of funding it (`HousePool.sol:960-970`).
3. `refundRedeemEpochRemainder` marks the epoch refundable and removes it from the global epoch queue (`TrancheVault.sol:1520-1522`). The controller's queue is not touched, so the lot stays the victim's head with zero claimable shares.

Every lot the victim files later queues behind it. Settlement funds those lots normally: `claimableRedeemRequest(id, victim)` is positive and the USDG has already left the pool and sits in the vault's claim escrow. The standard views and claims still answer for the head and return 0.

The cost of clearing. `claimRedeemRefund` returns the stranger's atom to the victim and, because shares arrive, writes `lastDepositTime[victim] = block.timestamp` (`:942-943`). For the next 3,600 s every `requestRedeem` (`:617`) and every share transfer (`:326`) of that wallet reverts.

What is and is not standard here. ERC-7540 authorises requests by owner, and the vault's NatSpec says so (`interfaces/IAsyncTrancheVault.sol:11`). We do not claim that an unauthenticated controller alone is a bug. The defect is the combination: a third party can append to a queue that the standard claim surface consumes strictly from the head and that only the queue's owner can advance. `requestDeposit` has the same shape (`:569-575`, owner-only check at `:1582`). We did not test the deposit side.

## Impact Details
All figures are from the PoC. The ordinary-account, holder and busy-vault cases each have a control from the same snapshot. Amounts are USDG.

Ordinary account. The victim files one lot. The stranger's atom was planted one epoch earlier. After one settlement pass, per tranche:

| | Junior | Senior |
|---|---|---|
| Victim lot, funded and claimable by id | 2,499.763524 | 2,500.273991 |
| `maxRedeem(victim)`, `maxWithdraw(victim)` | 0, 0 | 0, 0 |
| `redeem`, `withdraw` | revert | revert |
| Gas of the clearing call | 103,718 | 103,679 |
| Lock after clearing | 3,600 s | 3,600 s |

The reverts are `ERC4626ExceededMaxRedeem(victim, shares, 0)` and `ERC4626ExceededMaxWithdraw(victim, assets, 0)`. After the clearing call the victim's remaining wallet shares (7,500.07 Junior shares in the test) are refused by `requestRedeem` (`TrancheVault__DepositCooldown`) and `transfer` (`TrancheVault__TransferDuringCooldown`) at +0 s and +3,599 s; at +3,600 s a transfer succeeds. In the control the standard `redeem` pays the same lot, writes no lock, and `requestRedeem` and `transfer` of the wallet are accepted at once.

Standard-only holder contract. The test holder uses ERC-20, the ERC-7540 Redeem fragment and the ERC-7575 claim surface. It has no `setOperator` call, no call into the custom interface and no upgrade path. It files one exit of about 250 USDG every 24 hours. A keeper pass runs every hour and a user asks the holder to claim whenever one of its lots reports claimable shares. One plant before its first exit, 720 hours:

| Arm | Claims succeeded | Paid to users | Funded but unreachable at end |
|---|---|---|---|
| Junior, control (both styles) | 30 of 30 | 7,460.072129 | 0 |
| Junior, one plant, redeem-style | 0 of 719 | 0 | 7,460.072126 (30 of 30 lots) |
| Junior, one plant, withdraw-style | 0 of 719 | 0 | 7,460.072126 (30 of 30 lots) |
| Senior, control (both styles) | 30 of 30 | 7,519.996997 | 0 |
| Senior, one plant, redeem-style | 0 of 719 | 0 | 7,519.996999 (30 of 30 lots) |
| Senior, one plant, withdraw-style | 0 of 719 | 0 | 7,519.996999 (30 of 30 lots) |

In all 719 probes of each attack arm one of the holder's lots reported `claimableRedeemRequest > 0` while `maxRedeem` and `maxWithdraw` were 0. At the end the head is the refunded junk id and the vault's `withdrawalEscrowAssets` equals the unreachable sum. A plant after the holder's first exit cuts off every later lot: 1 claim succeeds (249.978635) and 29 lots (7,210.093485) are unreachable.

Nobody else can clear it. The stranger, the keeper, the pool owner and another LP each called `claimRedeemRefund`, `cancelRedeemRequest`, `claimRedeem`, `redeem` and `withdraw` for the holder: 20 calls, all `TrancheVault__NotControllerOrOperator`. The stranger cannot cancel its own lot before maturity either. Five more keeper passes, a share gift to the holder and a new lot of the holder leave the head on the junk id. The vault is not upgradeable (`:31`) and its owner-gated functions cover only the maintenance fee configuration (`:365`, `:375`, `:405`).

Busy vault. If another LP redeems in the junk epoch, that epoch is funded and the junk lot is worth 1 share and 0 assets instead of being refunded. Another LP redeems 2 USDG in every epoch, 168 hours, Junior:

- Withdraw-style holder, one plant: 0 of 167 claims succeed, 7 of 7 lots (1,744.949407) unreachable, against 7 of 7 paid in the control. The head reports `maxWithdraw = 0`, and `withdraw(0)` reverts (`:1181`).
- Redeem-style holder, one plant: nothing is lost. One extra zero-asset claim removes the junk (8 claims instead of 7) and users receive the same 1,744.949407.
- Redeem-style holder, a plant in every epoch in which the holder files nothing (161 plants): a delay only. 1,497.162955 of 1,744.949407 was paid within the window (145 claims against 7 in the control); the newest lot (247.786452) was funded and not yet paid when the window ended.

Attacker cost. One share atom per plant, which redeems for 0 USDG atoms, plus 217,765 to 380,156 execution gas for the two calls (transfer of the atom and `requestRedeem`) in the single-plant runs; the 161 repeated plants average about 188,289. The atom becomes refundable to the victim; the attacker receives no USDG and cannot take it back.

Limits of the result.

- No loss for a holder that can call the custom explicit-id claim: `claimRedeem(id, ...)` paid the victim's lot in full (2,499.763524) with the junk still at the head, and wrote no lock. Such a holder never needs to clear.
- A controller that approved an operator recovers: the operator calls `claimRedeemRefund` and the standard `redeem` then pays (tested).
- The lock that follows clearing is 3,600 s and follows the victim's own call. An account that files its requests and transfers before clearing avoids its effect.
- The permanent lock-out is measured for a holder design (standard surface only, immutable) that the protocol itself does not deploy, which is why the claim is Griefing and not a freezing row.
- No protocol contract in `src/` calls the vault's redeem or claim functions, so no protocol-own balance sits behind a planted head (code read, not a test).
- The refund needs the junk epoch to hold no valued redeem of anyone. Where another account redeems in it, only withdraw-style claims stay blocked. Where the victim itself files in it, the atom merges into the victim's own lot and nothing is blocked (code read, `:701`, `:706`).
- No attacker profit. The attacker needs an existing share balance past its own cooldown, 5 USDG of shares in the test.
- Gas figures are execution gas inside the test and exclude the 21,000 base cost per transaction.

Who the attacker is. Any share holder. No role, no keeper access, no timing against the victim: the plant can be placed before the victim files anything.

What the PoC obtains and what it is given. Given: a test contract of ours as the standard-only holder (not a deployed or third-party contract), the repository's mock price feed and mock USDG, and time warps. Obtained through public calls only: the planted head, the zero views, the reverts and the claim counts above.

Closest exclusions and documents.

- By design. The custom interface's NatSpec says a "refundable redeem head must be cancelled or reclaimed before standard claims advance" (`interfaces/IAsyncTrancheVault.sol:8-10`). That sentence describes the head rule for a controller's own lots. It does not say that a stranger can create the head for someone else. The integration guide recommends "`IERC7540` and `IERC7575` for standard-compatible integrations" (`CANONICAL_ENTRYPOINTS.md:237`). Those two files carry no such caveat (no occurrence of "refund", "head" or "oldest"), and `interfaces/IERC7575.sol:95` describes `maxRedeem`: "The result is funded claim capacity." Here 7,460 USDG is funded and the result is 0.
- Known limitations. `SECURITY.md:1399` says "Redemption cancellation or terminal refund still begins a new cooldown". That is the cost of clearing one's own refund, not a third party forcing the refund into another account's queue. `SECURITY.md:1409` says "Per-controller flooring can produce zero entitlement for a dust-sized request." That is the zero value of the atom, not the blocking of later lots.
- Listed known issues. The programme page lists Cecuro MED-24 and MED-29 ("LP refund epochs and redemption rounding"). MED-24 is zero-payout refund epochs using up the 16-epoch settlement work cap and delaying the funding of later epochs. Here the victim's lot is funded in the same pass; the per-controller head, placed by a stranger, hides it from the standard claims. MED-29 is the zero entitlement of a tiny lot.
- Fixed audit findings. LOW-40 was a controller delivering dust shares to a third-party receiver; its fix requires `receiver == controller` on share-delivering paths, so that a controller "cannot use these paths to send unsolicited shares and impose a cooldown". The request path's free `controller` is a different mechanism and that fix does not touch it. LOW-43 is a documentation fix about which address the `max*` views take.
- Programme rules. No exclusion names integrating contracts. "Any testing with pricing oracles or third-party smart contracts" is a testing rule; the holder here is our own test contract.
- Griefing ratio. The programme's ratio baseline applies to High and Critical findings. We claim Medium.

## Recommendation
Either of:

- Require the controller's consent when a request names a controller other than the owner, in `requestRedeem` and in the deposit twin:

```solidity
// TrancheVault.sol, requestRedeem, after the zero-address check (:611-613)
if (controller != owner && msg.sender != controller && !isOperator[controller][msg.sender]) {
    revert TrancheVault__NotControllerOrOperator();
}
```

- Or let the standard surface skip a refundable head: have `maxRedeem`, `maxWithdraw`, `redeem` and `withdraw` walk past head lots whose epoch is `refundEnabled` to the first lot with funded shares.

The first option narrows ERC-7540's request authorisation for third-party controllers. We have not built or tested a patch.

## References
`TrancheVault.sol:31`, `:309`, `:326`, `:365`, `:375`, `:405`, `:569-575`, `:606-632`, `:701-704`, `:706`, `:889`, `:904`, `:921`, `:932`, `:942-943`, `:1003`, `:1011`, `:1160`, `:1181`, `:1223-1225`, `:1299-1311`, `:1520-1522`, `:1582`, `:1603-1609`, `:1611-1618`, `:1783-1797`, `:1799-1822`; `HousePool.sol:960-970`; `interfaces/IAsyncTrancheVault.sol:8-11`; `interfaces/IERC7575.sol:95`; `packages/perps/CANONICAL_ENTRYPOINTS.md:237`; `packages/perps/SECURITY.md:1399`, `:1409`.

## Link to Proof of Concept

https://gist.github.com/mhkarazeybek/d9cb82ffdfd3bbf21d58fbc607385ba1

## Proof of Concept

## Proof of Concept
Two files for the repository's Foundry suite at the in-scope commit: `ControllerJunkHeadBase.sol` (fixture and the holder contract) and `ControllerJunkHead.t.sol` (11 tests). Nothing in `src/` is modified. Expected: `11 passed; 0 failed`.

```
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/poc
cp ControllerJunkHeadBase.sol ControllerJunkHead.t.sol packages/perps/test/poc/
forge test --root packages/perps --match-path 'test/poc/ControllerJunkHead*.t.sol' -vv
```

Tested with forge 1.7.1.

What the tests use.

- Fixture: the repository's `BasePerpTest`, with the release Senior limits, oracle adverse confidence multiplier and Router opening minimum installed through the owner's timelocked paths in `setUp` and asserted. Pyth and USDG are the repository's mocks.
- After `setUp` every state change is a public call sent by the acting account. The planting call's `RedeemRequest` event is checked in full, so controller, owner and sender are asserted.
- Cheatcodes in the two files: `warp`, `prank`, `startPrank`, `stopPrank`, `expectRevert`, `expectEmit`, `snapshotState`, `revertToState`, `getBlockTimestamp`. No `store`, `etch`, `mockCall` or `deal`.
- Every figure quoted above is asserted or printed by a test; tests 1, 4 and the four 720 h holder tests run a control arm from the same snapshot. The whole suite runs in a few seconds once compiled.

Output, trimmed ("..." marks cuts, including line breaks):

```
[PASS] test_1_OrdinaryAccount_Junior()
  === 1. ordinary account, Junior, ATTACK (one planted lot)
    victim lot claimable USDG: 2499.763524 ... maxRedeem(victim): 0 ... maxWithdraw(victim): 0
    redeem and withdraw revert ERC4626ExceededMaxRedeem / ERC4626ExceededMaxWithdraw (max 0)
    LIMIT explicit-id claimRedeem(id) paid USDG, junk head not cleared: 2499.763524
    claimRedeemRefund(junk) clears the head, gas: 103718
    lock: DepositCooldown / TransferDuringCooldown at +0 s and +3599 s, free at +3600 s
  === 1. ordinary account, Junior, CONTROL (no plant)
    standard redeem paid USDG, no lock, requestRedeem and transfer accepted at once: 2499.763524
[PASS] test_2_Holder_Junior_RedeemStyle_720h()
  === 2. holder, Junior redeem-style, CONTROL 720 h
    claim() calls that succeeded: 30 ... USDG received by the holder's users: 7460.072129
  === 2. holder, Junior redeem-style, ATTACK one planted lot, 720 h
    claim() calls made while an own lot reported claimable shares: 719
    claim() calls that succeeded: 0
    USDG received by the holder's users: 0.000000
    USDG funded for the holder's lots but unreachable at end: 7460.072126
    own lots funded but unreachable at end: 30
    probes with own claimableRedeemRequest > 0 and maxRedeem == maxWithdraw == 0: 719
    attacker plants (1 share atom each): 1 ... attacker gas, all plants (transfer + requestRedeem): 380156
    holder controllerRedeemHead at end: 474883 ... vault withdrawalEscrowAssets at end: 7460.072126
[PASS] test_2_Holder_Senior_WithdrawStyle_720h()
    ... claim() calls that succeeded: 0 ... USDG funded for the holder's lots but unreachable at end: 7519.996999
[PASS] test_3_Escapes_NoThirdPartyCanClear()
    stranger, keeper, pool owner, other LP: claimRedeemRefund / cancelRedeemRequest / claimRedeem / redeem / withdraw for the holder -> NotControllerOrOperator (20 calls)
    controllerRedeemHead(holder) after every attempt and 5 more keeper passes (junk id): 474883
[PASS] test_4_BusyVault_WithdrawStyleFrozen_RedeemStyleDelayed_168h()
  === 4. busy vault, Junior withdraw-style, ATTACK one planted lot, 168 h
    claim() calls that succeeded: 0 ... USDG funded for the holder's lots but unreachable at end: 1744.949407
  === 4. busy vault, Junior redeem-style, ATTACK one planted lot, 168 h
    claim() calls that succeeded: 8 ... USDG received by the holder's users: 1744.949407
[PASS] test_5_AttackerCost()
    Junior: share atoms per plant: 1 ... Junior: gas, first plant (transfer + requestRedeem): 370162
Suite result: ok. 11 passed; 0 failed; 0 skipped
```

