# \#97595 [SC-Critical] An LP who also trades can take other junior LPs' principal by moving junior NAV around its own epoch deposit or redemption

**Submitted on Nov 2nd 2026 at 14:08:24 UTC by @janneh2000 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97595
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

I found that HousePool prices junior deposits and redemptions at epoch settlement on a NAV that already includes VPI collected or rebated on open positions, even though the lifetime clamp guarantees the same position hands that VPI back when it closes. This is in audit-comp-plether at a82d4fef (packages/perps/src). An LP who also trades can open a skew-flipping position just before its own LP request is priced, let the permissionless settlement price it, then close. The clamp returns the VPI into a pool the attacker now owns a different share of, and the other junior LPs lose principal in USDG. In the PoC's 10M pool, a victim holding 2M of junior redeems 13,857.69 USDG less than in the control after one ~3.5 h cycle, the attacker nets +9,185.43 USDG after every fee, and three cycles cost the victim 42,614.94 USDG. Senior is untouched.

The two pieces that disagree:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:116-118 @ a82d4fef  (lifetime clamp)
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}

// packages/perps/src/libraries/HousePoolAccountingLib.sol:63-65 @ a82d4fef  (buildReconcileSnapshot, used for LP entry and exit)
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
```

A skew-reducing open is paid its rebate out of pool cash at once (CfdEngineSettlementSidecar.sol:442-444). The trader must lock the same amount in a non-withdrawable reserve whose only job is to pay it back (sidecar :466-477, SECURITY.md:866). A skew-increasing open's VPI goes into pool cash at once (sidecar :454-463), and the clamp lets that position take it back on a skew-reducing close, paid from pool cash (:621-622). The snapshot that prices both tranches in settleLpEpoch (HousePool.sol:797-841) carries no receivable for the first case and no liability for the second. Settlement is permissionless through OrderRouter.settleLpEpoch (OrderRouter.sol:479), and a trader can execute its own order. So anyone holding an LP request and a trade leg picks the NAV that request is priced at, for two 4 bp execution fees.

Simplest form, at full depth with no depth change (test 5): the attacker holds 75% of junior and has a full redemption pending. 90 s before it is priced, it opens a 3.9M LONG (about 7.6k of VPI into junior NAV). The keeper settles, the redemption is paid at the pumped NAV, and the attacker closes; the clamp refunds its VPI out of what the remaining LPs own. Attacker +2,581.66, victim -4,139.59. Test 4 is the entry mirror with no prior stake: 20M deposited into 8M of junior around a SHORT flatten of an honest trader's skew. Attacker +2,306.67, victim -5,425.99.

Repeatable form (tests 1-3): each cycle the attacker redeems its whole junior stake, requests a deposit of the payout, and 90 s before activation opens a 3.9M SHORT against the existing LONG skew. Open VPI uses current depth (CfdEnginePlanLib.sol:448-449, :487), and the skew cap doesn't stop a skew-reducing open whose side doesn't become the heavy side (:474-476). So the rebate is 18,993.85 USDG, about 2.5x its full-depth value. The deposit mints at the depressed NAV, and the close repays the whole rebate into a NAV the attacker now owns more of. Test 3 does this from one account against an honest trader's skew.

PoC: https://gist.github.com/janneh2000/7ab3bf2ed4f5e9496f85d8391264c4a2 (one Foundry file using the repo's BasePerpTest; real engine, clearinghouse, sidecars, HousePool, both TrancheVaults, OrderRouter and PletherOracle on MockPyth; src unmodified).

**Reproduce**:

```
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
curl -sL https://gist.githubusercontent.com/janneh2000/7ab3bf2ed4f5e9496f85d8391264c4a2/raw/VPI_LP_NAV_Combined_Submit.t.sol -o packages/perps/test/perps/VPI_LP_NAV_Combined_Submit.t.sol
forge test --root packages/perps --match-path test/perps/VPI_LP_NAV_Combined_Submit.t.sol -vv
```

Each test reverts an ATTACK branch and a CONTROL branch to one vm.snapshotState(). Both have the same honest keeper, the same LP flows and a price of 1.0 on every tick; ATTACK only adds the attacker's trade legs, and the attacker ends flat. "Attacker net gain" counts every LP, fee, carry and VPI leg. Output (Foundry 1.5.1, solc 0.8.35, trimmed):

```
[PASS] test_1_G9_Rotation_OneCycle_WithAtt2OnlyControl() (gas: 26096281)
    rebate R paid out of junior NAV at shrunk depth (cycle 1) 18993.85 USDG
    ATT2-ONLY (persistent skew + rotation, no flip) vs CONTROL:
      attacker net gain (all legs)    -1961.19 USDG
      victim junior LP loss (mark)    -400.98 USDG
    ATTACK (full rotation) vs CONTROL:
      attacker net gain (all legs)    9185.43 USDG
      victim junior LP loss (mark)    13858.51 USDG
      senior LP loss                  -0.01 USDG
      treasury fee gain               4680.00 USDG
    victim USDG actually redeemed 24 h later, control 1999853.81 USDG
    victim USDG actually redeemed 24 h later, attack  1985996.11 USDG
    victim realised USDG loss (permanent)             13857.69 USDG
[PASS] test_2_G9_Rotation_ThreeCycles()                 attacker 31716.23 / victim 42614.94 USDG
[PASS] test_3_G9H_OrganicSkewRotation_NoUnhedgedLeg()   attacker 11157.32 / victim 14270.19 USDG
[PASS] test_4_G2V_EntryMirror_NoPriorStake()            attacker 2306.67 / victim 5425.99 USDG
[PASS] test_5_G2V_ExitPump_75pct()                      attacker 2581.66 / victim 4139.59 USDG
[PASS] test_6_Contrast_PassiveLp_vs_Manufactured()      same LP-side transfer, 5701.89 USDG, organic vs manufactured
Suite result: ok. 6 passed; 0 failed; 0 skipped
```

The no-flip control in test 1 matters: holding the skew across the LP flow leaves the victim slightly better off (+400.98). The loss comes only from flipping skew around the settlement. In every test, the victim's loss is about the attacker's gain plus treasury fees. Two setup notes: the PoC lifts the senior capacity cap, which is irrelevant with 2M senior (the deployment cap is 40M). Test 1's treasury shows three fees, not four, because the pool keeps the fee on a rebate close (sidecar :615-616).

**Preconditions**: no privileged role, no oracle trick (one price throughout), an honest keeper, and permissionless settlement and own-order execution. The attacker needs a large share of junior or a large deposit relative to it: 75% of junior in tests 1-3 and 5, and a deposit of 2.5x junior in test 4. The exit form pays when the attacker's junior share is above 0.16·D/S, which is about 41% at a 39% skew/depth ratio (two 4 bp fees against a share of k·S²/2D). A separate sweep, not in the attached file, put rotation break-even at roughly 37% of junior. Required capital scales with pool size, and the cycle repeats every ~3.5 h (1 h epochs, 1 h cooldown at TrancheVault.sol:38). Margin is about 30-40k USDG per 3.9M leg. The same mis-booking also moves junior NAV when an honest trader closes a rebated position or is liquidated. So an LP with only 10% of junior gains just by choosing the order of two permissionless calls, executing that pending action and settling its own matured request, with no trade of its own. Additional tests (available on request) measured +678.94 to +719.74 USDG per event, with an equal loss to the remaining LPs.

**Affected**: immunefi-team/audit-comp-plether @ a82d4fefb02768e5b3bddcc97dee371eee24fea8, packages/perps/src. The PoC uses the risk params of script/DeployPerpsArbitrumSepolia.s.sol:434-446 (vpiFactor 0.01, maxSkew 0.4).

Why this isn't a known issue: the AI scan's two "temporary pool liquidity" items cover a trader moving depth to change its own close VPI. Here the value moves through LP epoch pricing, and tests 4-6 never change depth, so a depth fix (commit-time or TWAP depth) leaves them working. SC-H1 in the pre-audit is a passive deposit ahead of a real senior yield sync; here the attacker creates the jump on junior itself, and the jump is guaranteed to reverse. The docs keep anticipated VPI out of NAV (SECURITY.md:871-872, ACCOUNTING_SPEC.md:1064-1066), and they accept collection-time timing only for carry, which is real revenue (ACCOUNTING_SPEC.md:731-740). VPI under the clamp is not revenue: it comes back. The clamp comment at CloseAccountingLib.sol:112-114 describes protection against LP sandwiching of the trader's own rebate, not of LP entry and exit pricing.

**Fix** **(tested locally**; VPI_LP_NAV_fix.diff in the gist, 2 files, +9/-1): keep a running `int256 totalOpenVpiAccruedAssets` in CfdEngine, updated in the only two places a position's vpiAccrued is stored or deleted (settlementWritePosition and settlementDeletePosition). Subtract it from terminalLpPriceDeltaAssets in CfdEngineProtocolLens, which feeds HousePool's reconcile snapshot. With the fix applied, all six tests fail their "attack is profitable" assertion. The attacker ends at minus its own execution fees (test 1: +9,185.43 → -5,081.31; tests 3-5: about -3,120), and no victim loses (test 1 victim: -13,858.51 → +401.10). The repo's LpEntryExitNavParity (4/4) and VpiRoundTripAccounting (2/2) suites still pass. The fix is conservative: positive accruals count at the most the clamp could refund, so LP NAV can read slightly low while those positions are open, and the difference is recognised at close.

Local testing only; no transaction was sent to any network. Proof label: LOCAL-POC-CONFIRMED.



## Link to Proof of Concept

https://gist.github.com/janneh2000/7ab3bf2ed4f5e9496f85d8391264c4a2

## Proof of Concept

## Proof of Concept

PoC: https://gist.github.com/janneh2000/7ab3bf2ed4f5e9496f85d8391264c4a2 (one Foundry file using the repo's BasePerpTest; real engine, clearinghouse, sidecars, HousePool, both TrancheVaults, OrderRouter and PletherOracle on MockPyth; src unmodified).

**Reproduce**:

```
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
curl -sL https://gist.githubusercontent.com/janneh2000/7ab3bf2ed4f5e9496f85d8391264c4a2/raw/VPI_LP_NAV_Combined_Submit.t.sol -o packages/perps/test/perps/VPI_LP_NAV_Combined_Submit.t.sol
forge test --root packages/perps --match-path test/perps/VPI_LP_NAV_Combined_Submit.t.sol -vv
```

Each test reverts an ATTACK branch and a CONTROL branch to one vm.snapshotState(). Both have the same honest keeper, the same LP flows and a price of 1.0 on every tick; ATTACK only adds the attacker's trade legs, and the attacker ends flat. "Attacker net gain" counts every LP, fee, carry and VPI leg. Output (Foundry 1.5.1, solc 0.8.35, trimmed):

```
[PASS] test_1_G9_Rotation_OneCycle_WithAtt2OnlyControl() (gas: 26096281)
    rebate R paid out of junior NAV at shrunk depth (cycle 1) 18993.85 USDG
    ATT2-ONLY (persistent skew + rotation, no flip) vs CONTROL:
      attacker net gain (all legs)    -1961.19 USDG
      victim junior LP loss (mark)    -400.98 USDG
    ATTACK (full rotation) vs CONTROL:
      attacker net gain (all legs)    9185.43 USDG
      victim junior LP loss (mark)    13858.51 USDG
      senior LP loss                  -0.01 USDG
      treasury fee gain               4680.00 USDG
    victim USDG actually redeemed 24 h later, control 1999853.81 USDG
    victim USDG actually redeemed 24 h later, attack  1985996.11 USDG
    victim realised USDG loss (permanent)             13857.69 USDG
[PASS] test_2_G9_Rotation_ThreeCycles()                 attacker 31716.23 / victim 42614.94 USDG
[PASS] test_3_G9H_OrganicSkewRotation_NoUnhedgedLeg()   attacker 11157.32 / victim 14270.19 USDG
[PASS] test_4_G2V_EntryMirror_NoPriorStake()            attacker 2306.67 / victim 5425.99 USDG
[PASS] test_5_G2V_ExitPump_75pct()                      attacker 2581.66 / victim 4139.59 USDG
[PASS] test_6_Contrast_PassiveLp_vs_Manufactured()      same LP-side transfer, 5701.89 USDG, organic vs manufactured
Suite result: ok. 6 passed; 0 failed; 0 skipped
```

The no-flip control in test 1 matters: holding the skew across the LP flow leaves the victim slightly better off (+400.98). The loss comes only from flipping skew around the settlement. In every test, the victim's loss is about the attacker's gain plus treasury fees. Two setup notes: the PoC lifts the senior capacity cap, which is irrelevant with 2M senior (the deployment cap is 40M). Test 1's treasury shows three fees, not four, because the pool keeps the fee on a rebate close (sidecar :615-616).



**Affected**: immunefi-team/audit-comp-plether @ a82d4fefb02768e5b3bddcc97dee371eee24fea8, packages/perps/src. The PoC uses the risk params of script/DeployPerpsArbitrumSepolia.s.sol:434-446 (vpiFactor 0.01, maxSkew 0.4).

Why this isn't a known issue: the AI scan's two "temporary pool liquidity" items cover a trader moving depth to change its own close VPI. Here the value moves through LP epoch pricing, and tests 4-6 never change depth, so a depth fix (commit-time or TWAP depth) leaves them working. SC-H1 in the pre-audit is a passive deposit ahead of a real senior yield sync; here the attacker creates the jump on junior itself, and the jump is guaranteed to reverse. The docs keep anticipated VPI out of NAV (SECURITY.md:871-872, ACCOUNTING_SPEC.md:1064-1066), and they accept collection-time timing only for carry, which is real revenue (ACCOUNTING_SPEC.md:731-740). VPI under the clamp is not revenue: it comes back. The clamp comment at CloseAccountingLib.sol:112-114 describes protection against LP sandwiching of the trader's own rebate, not of LP entry and exit pricing.


**LOCAL CLI RUN**: 

rivaldo@Alies-MacBook-Air audit-comp-plether % git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
curl -sL https://gist.githubusercontent.com/janneh2000/7ab3bf2ed4f5e9496f85d8391264c4a2/raw/VPI_LP_NAV_Combined_Submit.t.sol -o packages/perps/test/perps/VPI_LP_NAV_Combined_Submit.t.sol
forge test --root packages/perps --match-path test/perps/VPI_LP_NAV_Combined_Submit.t.sol -vv
Cloning into 'audit-comp-plether'...
remote: Enumerating objects: 20784, done.
remote: Counting objects: 100% (4247/4247), done.
remote: Compressing objects: 100% (797/797), done.
remote: Total 20784 (delta 3809), reused 3460 (delta 3450), pack-reused 16537 (from 2)
Receiving objects: 100% (20784/20784), 21.30 MiB | 20.06 MiB/s, done.
Resolving deltas: 100% (17563/17563), done.
Note: switching to 'a82d4fefb02768e5b3bddcc97dee371eee24fea8'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

Or undo this operation with:

  git switch -

Turn off this advice by setting config variable advice.detachedHead to false

HEAD is now at a82d4fef fix(ci): stabilize perps compilation and test execution (#135)
Submodule 'lib/forge-std' (https://github.com/foundry-rs/forge-std) registered for path 'lib/forge-std'
Submodule 'lib/morpho-blue' (https://github.com/morpho-org/morpho-blue) registered for path 'lib/morpho-blue'
Submodule 'lib/openzeppelin-contracts' (https://github.com/OpenZeppelin/openzeppelin-contracts) registered for path 'lib/openzeppelin-contracts'
Cloning into '/Users/rivaldo/Downloads/VPI-LP-NAV/audit-comp-plether/audit-comp-plether/lib/forge-std'...
Cloning into '/Users/rivaldo/Downloads/VPI-LP-NAV/audit-comp-plether/audit-comp-plether/lib/morpho-blue'...
Cloning into '/Users/rivaldo/Downloads/VPI-LP-NAV/audit-comp-plether/audit-comp-plether/lib/openzeppelin-contracts'...
Submodule path 'lib/forge-std': checked out 'f5495c9b993bfc6e79dd52adfdcb7dac0a1da918'
Submodule path 'lib/morpho-blue': checked out '55d2d99304fb3fb930c688462ae2ccabb1d533ad'
Submodule 'lib/forge-std' (https://github.com/foundry-rs/forge-std) registered for path 'lib/morpho-blue/lib/forge-std'
Cloning into '/Users/rivaldo/Downloads/VPI-LP-NAV/audit-comp-plether/audit-comp-plether/lib/morpho-blue/lib/forge-std'...
Submodule path 'lib/morpho-blue/lib/forge-std': checked out '2f112697506eab12d433a65fdc31a639548fe365'
Submodule 'lib/ds-test' (https://github.com/dapphub/ds-test) registered for path 'lib/morpho-blue/lib/forge-std/lib/ds-test'
Cloning into '/Users/rivaldo/Downloads/VPI-LP-NAV/audit-comp-plether/audit-comp-plether/lib/morpho-blue/lib/forge-std/lib/ds-test'...
Submodule path 'lib/morpho-blue/lib/forge-std/lib/ds-test': checked out 'e282159d5170298eb2455a6c05280ab5a73a4ef0'
Submodule path 'lib/openzeppelin-contracts': checked out 'fcbae5394ae8ad52d8e580a3477db99814b9d565'
Submodule 'lib/erc4626-tests' (https://github.com/a16z/erc4626-tests.git) registered for path 'lib/openzeppelin-contracts/lib/erc4626-tests'
Submodule 'lib/forge-std' (https://github.com/foundry-rs/forge-std) registered for path 'lib/openzeppelin-contracts/lib/forge-std'
Submodule 'lib/halmos-cheatcodes' (https://github.com/a16z/halmos-cheatcodes) registered for path 'lib/openzeppelin-contracts/lib/halmos-cheatcodes'
Cloning into '/Users/rivaldo/Downloads/VPI-LP-NAV/audit-comp-plether/audit-comp-plether/lib/openzeppelin-contracts/lib/erc4626-tests'...
Cloning into '/Users/rivaldo/Downloads/VPI-LP-NAV/audit-comp-plether/audit-comp-plether/lib/openzeppelin-contracts/lib/forge-std'...
Cloning into '/Users/rivaldo/Downloads/VPI-LP-NAV/audit-comp-plether/audit-comp-plether/lib/openzeppelin-contracts/lib/halmos-cheatcodes'...
Submodule path 'lib/openzeppelin-contracts/lib/erc4626-tests': checked out '232ff9ba8194e406967f52ecc5cb52ed764209e9'
Submodule path 'lib/openzeppelin-contracts/lib/forge-std': checked out '3b20d60d14b343ee4f908cb8079495c07f5e8981'
Submodule path 'lib/openzeppelin-contracts/lib/halmos-cheatcodes': checked out '7328abe100445fc53885c21d0e713b95293cf14c'
[⠊] Compiling...
[⠃] Compiling 194 files with Solc 0.8.35
[⠢] Solc 0.8.35 finished in 72.15s
Compiler run successful with warnings:



Ran 6 tests for test/perps/VPI_LP_NAV_Combined_Submit.t.sol:VpiLpNavCombinedSubmitTest
[PASS] test_1_G9_Rotation_OneCycle_WithAtt2OnlyControl() (gas: 26096281)
Logs:
  G9 ROTATION, 1 cycle, attacker 75% of junior, skew self-supplied by ATT2:
    rebate R paid out of junior NAV at shrunk depth (cycle 1) 18993.85 USDG
    ATT2-ONLY (persistent skew + rotation, no flip) vs CONTROL:
      attacker net gain (all legs)    -1961.19 USDG
      victim junior LP loss (mark)    -400.98 USDG
      senior LP loss                  -0.00 USDG
      treasury fee gain               1560.00 USDG
      victim value, control           1999968.00 USDG
      victim value, this branch       2000368.99 USDG
    ATTACK (full rotation) vs CONTROL:
      attacker net gain (all legs)    9185.43 USDG
      victim junior LP loss (mark)    13858.51 USDG
      senior LP loss                  -0.01 USDG
      treasury fee gain               4680.00 USDG
      victim value, control           1999968.00 USDG
      victim value, this branch       1986109.49 USDG
    flip increment, attacker gain (ATTACK - ATT2-ONLY) 11146.62 USDG
    flip increment, victim loss   (ATTACK - ATT2-ONLY) 14259.50 USDG
    victim USDG actually redeemed 24 h later, control 1999853.81 USDG
    victim USDG actually redeemed 24 h later, attack  1985996.11 USDG
    victim realised USDG loss (permanent)             13857.69 USDG

[PASS] test_2_G9_Rotation_ThreeCycles() (gas: 36278775)
Logs:
  G9 ROTATION, 3 cycles (~10 h), attacker 75% of junior, skew self-supplied by ATT2:
    rebate R paid out of junior NAV at shrunk depth (cycle 1) 18993.85 USDG
    ATT2-ONLY (persistent skew + rotation, no flip) vs CONTROL:
      attacker net gain (all legs)    -1980.37 USDG
      victim junior LP loss (mark)    -420.16 USDG
      senior LP loss                  -0.00 USDG
      treasury fee gain               1560.00 USDG
      victim value, control           1999913.20 USDG
      victim value, this branch       2000333.36 USDG
    ATTACK (full rotation) vs CONTROL:
      attacker net gain (all legs)    31716.23 USDG
      victim junior LP loss (mark)    42614.94 USDG
      senior LP loss                  -0.01 USDG
      treasury fee gain               10920.00 USDG
      victim value, control           1999913.20 USDG
      victim value, this branch       1957298.25 USDG

[PASS] test_3_G9H_OrganicSkewRotation_NoUnhedgedLeg() (gas: 12835423)
Logs:
  G9H ROTATION, 1 cycle, attacker 75% of junior, organic skew (honest BOB 3.9M LONG), no ATT2:
    rebate R paid out of junior NAV at shrunk depth (cycle 1) 18993.85 USDG
    ATTACK (flip only) vs CONTROL:
      attacker net gain (all legs)    11157.32 USDG
      victim junior LP loss (mark)    14270.19 USDG
      senior LP loss                  -0.00 USDG
      treasury fee gain               3120.00 USDG
      victim value, control           2001868.63 USDG
      victim value, this branch       1987598.44 USDG

[PASS] test_4_G2V_EntryMirror_NoPriorStake() (gas: 9790797)
Logs:
  G2-V ENTRY MIRROR (no prior stake; 20M deposit into 8M junior; organic 3.9M skew):
    ATTACK vs CONTROL:
      attacker net gain (all legs)    2306.67 USDG
      victim junior LP loss (mark)    5425.99 USDG
      senior LP loss                  0.00 USDG
      treasury fee gain               3120.00 USDG
      victim value, control           8007584.21 USDG
      victim value, this branch       8002158.21 USDG

[PASS] test_5_G2V_ExitPump_75pct() (gas: 7967804)
Logs:
  G2-V EXIT PUMP (attacker 6M of 8M junior = 75%; 3.9M skew round trip at full depth):
    ATTACK vs CONTROL:
      attacker net gain (all legs)    2581.66 USDG
      victim junior LP loss (mark)    4139.59 USDG
      senior LP loss                  0.00 USDG
      treasury fee gain               1560.00 USDG
      victim value, control           1999990.83 USDG
      victim value, this branch       1995851.24 USDG

[PASS] test_6_Contrast_PassiveLp_vs_Manufactured() (gas: 12381328)
Logs:
  CONTRAST (exiting LP owns 75% of junior):
    ORGANIC: passive LP exit payout gain (BOB's skew)    5701.89 USDG
    ORGANIC: victim loss                                 4139.59 USDG
    MANUFACTURED: attacker LP exit payout gain           5701.89 USDG
    MANUFACTURED: victim loss                            4139.59 USDG
    MANUFACTURED: attacker net gain after trade fees     2581.66 USDG

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 103.45ms (260.28ms CPU time)

Ran 1 test suite in 113.17ms (103.45ms CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)
