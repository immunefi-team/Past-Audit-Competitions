# \#98603 [SC-Low] Stranger-planted redeem head blinds a victim's ERC-7540 claim and imposes the LP cooldown

**Submitted on Nov 8th 2026 at 23:42:44 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98603
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
`TrancheVault.requestRedeem(shares, controller, owner)` lets any share owner open a request for any controller without that controller's consent. An attacker with 1 wei of Junior share opens `requestRedeem(1, victim, attacker)` in a quiet epoch. Because of the 3-decimal share offset, 1 wei quotes 0 assets, so the epoch settles with a refund and the request becomes the victim's `controllerRedeemHead`. From then on the victim's standard ERC-7540 surface reads the planted head: `maxRedeem(victim)` and `maxWithdraw(victim)` return 0, and `redeem`/`withdraw` revert, while the victim's real request (≈50,000 USDG in the PoC) is funded and claimable. The only way to clear the head is `claimRedeemRefund`, which delivers the attacker's 1 wei to the victim and sets `lastDepositTime[victim] = now`. That locks the victim's `requestRedeem` and share transfers for 1 hour, the exact outcome `IAsyncTrancheVault.sol:12-13` says the receiver==controller rule prevents. The attacker can repeat this for 1 wei of share plus two transactions per epoch.

## Vulnerability Details
Code at the competition commit a82d4fe (identical to the Arbitrum Sepolia v2.0.0 release 1209ca8 for `packages/perps/src`):

1. **No controller consent on request.** `TrancheVault.sol:606-631` authorises only the share **owner** (`msg.sender == owner`, an operator of owner, or an allowance). The `controller` is free. `_requireMinimumRedeemSize(shares, maxShares)` (`:624`) passes for 1 wei when the attacker's whole balance is 1 wei (`shares == maxShares`).
2. **The 1-wei request becomes a refundable head.** The share offset (`_decimalsOffset` = 3, `:279`) makes 1 wei quote 0 assets. HousePool settlement refunds zero-quote redeem epochs and sets `refundEnabled` (`HousePool.sol:957-985`, `:1520`). `_enqueueRedeemRequest` makes the oldest request the controller's head (`TrancheVault.sol:1791`).
3. **Standard ERC-7540 claims and views read only the head.** `withdraw` (`:1003`) and `redeem` (`:1011`) claim from `controllerRedeemHead[controller]`. `maxWithdraw` (`:1299-1303`) and `maxRedeem` (`:1306-1310`) return the head's claimable amount, which is 0 for a refund-only request. The interface doc (`IAsyncTrancheVault.sol:8-10`) states the head rule for the controller's own requests: "A rejected deposit or refundable redeem head must be cancelled or reclaimed before standard claims advance". It does not contemplate a head planted by someone else.
4. **Clearing the head imposes the cooldown.** `claimRedeemRefund` (`:916-947`) requires `receiver == controller` (`_requireShareReceiver`, `:1620-1628`) and then sets `lastDepositTime[receiver] = block.timestamp` (`:943`) before delivering the attacker's share. `requestRedeem` (`:617`) and share transfers (`:326`) then revert for `DEPOSIT_COOLDOWN` (1h).

`IAsyncTrancheVault.sol:12-13` promises: "Deposit claims and redemption cancellations/refunds require receiver == controller ... so share delivery cannot impose another account's cooldown". The receiver==controller rule was the fix for Cecuro LOW-39/LOW-40. It stops a stranger choosing the *receiver*, but not a stranger choosing the *controller* at request time. Here the shares that trigger the cooldown are the attacker's, delivered to the victim through the victim's mandatory refund claim.

### Relation to prior audits and known issues
Cecuro LOW-39/LOW-40 (fixed in efcf629aed) closed the cross-account cooldown through the *receiver* argument by requiring `receiver == controller`. The fix did not cover the *controller* argument of `requestRedeem`, which ERC-7540 leaves to the owner. Combined with the head-only standard claims and the zero-quote refund of 1-wei requests, a stranger's request still reaches the victim's head and its cooldown. This is an incomplete fix of LOW-40. No prior audit, known issue or public repo issue describes a planted controller head (checked the Cecuro report, including LOW-14, LOW-39, LOW-40 and LOW-67, and the pre-audit report, including H-2).

## Impact Details
Program impact: **Low: "Contract fails to deliver promised returns, but doesn't lose value"**.

- A victim LP, or an integrator such as a vault, router or AA wallet that uses only the standard ERC-7540/7575 surface (`maxRedeem`, `maxWithdraw`, `redeem`, `withdraw`), sees 0 and reverts on a funded claim (≈50,000 USDG in the PoC). It cannot reach its assets until it makes a non-standard call (`claimRedeemRefund` or an explicit-id `claimRedeem`).
- Clearing the planted head forces a 1-hour cooldown on the victim's own shares (no new redeem requests, no transfers). This is the cross-account cooldown the interface says cannot happen.
- The attack repeats every quiet epoch for 1 wei of share plus 2 transactions. The victim pays a refund-claim transaction each time and restarts its cooldown each time.

No value is lost: the explicit-id path `claimRedeem(E1, ...)` still pays the victim (PoC: 49,999.999983 USDG), which is why this is filed as Low.

**On ERC-7540's arbitrary controller.** ERC-7540 lets an owner name any controller, but it does not require two things this vault adds: standard claims that read only the controller's oldest request, and a cooldown set on refund delivery. Together they let a stranger's request override the victim's claim surface and cooldown. The vault's own interface comment rules out that cross-account cooldown. The report keeps the Low framing: funds stay reachable through explicit-id claims.

### Attack scenario and preconditions
- Junior tranche live with release params; the attacker deposits once (≥ the minimum deposit, 10,000 USDG in the PoC), waits its own cooldown, and sends 1 wei of share to each throwaway account B1, B2, ...
- Victim V holds Junior shares (100,000 USDG in the PoC).
1. Quiet epoch E0: `B1.requestRedeem(1, V, B1)`. It succeeds with no consent from V.
2. E0 settles: `redeemRefundPending(E0, V) = true`, `refundableRedeemRequest = 1`, `controllerRedeemHead[V] = E0`.
3. V requests a redeem of half its position in E1. E1 is funded: `claimableRedeemRequest(E1, V) = 50,077,720,544,564` shares ≈ 49,999.999987 USDG.
4. `maxRedeem(V) = 0`, `maxWithdraw(V) = 0`, and `redeem`/`withdraw` revert. In the control without the plant: `maxRedeem = 50,077,720,544,564` and `redeem` pays 49,999,999,987.
5. V clears with `claimRedeemRefund(E0, V, V)`: it receives the attacker's 1 wei and `lastDepositTime[V] = now`. `V.requestRedeem` reverts `TrancheVault__DepositCooldown` and `V.transfer` reverts `TrancheVault__TransferDuringCooldown` for 3,600 s. The control has no cooldown.
6. B2 re-plants E2. V's next funded request E3 again shows `maxRedeem = 0`.

## Recommendation
Smallest fix (the gist's `fix.diff`): require the controller's authority when a redeem request names a controller, using the contract's existing helper:

```diff
@@ TrancheVault.requestRedeem
         if (shares == 0) { revert TrancheVault__WithdrawalTooSmall(); }
+        _requireControllerAuth(controller);
         if (block.timestamp < lastDepositTime[owner] + DEPOSIT_COOLDOWN) {
```

With the fix, `B1.requestRedeem(1, V, B1)` reverts `TrancheVault__NotControllerOrOperator`, so no head is planted and both PoC tests fail at the plant step. The control path (V redeeming its own funded request) is unchanged.

If the team wants to keep ERC-7540's freedom to open requests for an arbitrary controller, two other changes close the impact: (a) let standard `redeem`/`withdraw`/`max*` skip refund-only heads whose refund basis came from another owner, and (b) do not set `lastDepositTime[receiver]` in `claimRedeemRefund` for shares the controller never owned. Option (b) is what the interface comment already promises.

Note from the PoC: the plant needs a quiet epoch. If another redeemer shares epoch E0, the 1-wei request is funded with 1 share instead of refunded, and V clears it with an ordinary `redeem(1)`, with no cooldown (`test_H006_busyEpochControl`). Redeem epochs are hourly, so quiet epochs are the common case at launch volumes.

## Link to Proof of Concept

https://gist.github.com/the-red-ace/09e999a055ad6c217347e97d5f76d000

## Proof of Concept

## Proof of Concept

**Gist:** https://gist.github.com/the-red-ace/09e999a055ad6c217347e97d5f76d000. It contains the test file, its fixture files, `fix.diff` and a README with the exact folder for each file.

**How the tests run:** in the repo's own Foundry suite at a82d4fe, with release parameters. No in-scope contract is modified. The only mocks are Pyth price updates and mock USDG, both allowed by the program. Cheat codes: `vm.warp`, mark refresh, snapshot/revert; no privileged pranks beyond the base setup.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/{h006,g1,x2}
cp H006PlantedHead.t.sol packages/perps/test/perps/h006/
cp G1Base.sol packages/perps/test/perps/g1/
cp X2GridBase.sol X2ErrNames.sol packages/perps/test/perps/x2/
forge test --root packages/perps --match-path test/perps/h006/H006PlantedHead.t.sol -vv
```

| Test | What it asserts |
|---|---|
| `test_H006_attack_vs_control` | Steps 1-6 above, side by side with a control timeline without the plant (snapshot/revert): `maxRedeem`/`maxWithdraw` = 0 and `redeem`/`withdraw` revert on the funded E1 claim; the refund clear sets the victim's cooldown (`requestRedeem` and `transfer` revert for 3,600 s); a second throwaway re-plants; the explicit-id `claimRedeem(E1)` still pays 49,999.999983 USDG. Control: `maxRedeem = 50,077,720,544,564`, `redeem` pays 49,999,999,987, no cooldown. |
| `test_H006_busyEpochControl` | Boundary: when another redeemer shares E0, the 1-wei request is funded instead of refunded and the victim clears it with an ordinary `redeem(1)`, without a cooldown. |

Expected output (reproduced on a clean a82d4fe checkout with only the gist files):
```
[PASS] test_H006_attack_vs_control()
[PASS] test_H006_busyEpochControl()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```
With `fix.diff` applied, both tests fail at the plant step with `TrancheVault__NotControllerOrOperator()`.
