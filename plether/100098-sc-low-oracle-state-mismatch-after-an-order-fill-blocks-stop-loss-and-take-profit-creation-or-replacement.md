# \#100098 [SC-Low] Oracle state mismatch after an order fill blocks stop-loss and take-profit creation or replacement

**Submitted on Nov 16th 2026 at 08:18:00 UTC by @Shubadu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100098
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
When a keeper executes an ordinary order using a price tick newer than the oracle’s last accepted live basket, the execution advances the Engine mark but leaves that accepted basket unchanged. PositionProtectionBook then asks PletherOracle for the latest price. The oracle rejects the older accepted basket as PriceOutOfOrder, so traders cannot create or replace stop-loss and take-profit protection until someone performs a live mark refresh. The issue requires valid prices and no privileged action. It causes a temporary failure of protection functions; it does not itself lose or freeze funds.

## Vulnerability Details
1. OrderRouterExecutionSidecar._updateEngineMark installs the execution tick as the Engine mark when its publish time is newer than the previous mark. This historical order execution does not also advance the oracle’s accepted live basket.

2. PletherOracle.getLatestPrice(MarkRefresh) replays the accepted basket and checks its publish time against the Engine mark. If the basket is older, it reverts PletherOracle__PriceOutOfOrder.

3. PositionProtectionBook._freshProtectionObservation calls that price view when creating protection, replacing protection, or committing an open order with attached protection. The call reverts before the protection book reaches its own _acceptedMarkInstalled() guard. That guard expressly accepts an accepted-basket time older than the Engine mark, but cannot help because the preceding price view has already reverted.

The runnable PoC reproduces the sequence with an initial live mark at T0, followed by a keeper-executed order using a valid tick at T0+1. Creating protection then reverts PriceOutOfOrder(T0, T0+1). A live OrderRouter.updateMarkPrice with current signed data restores creation. The accompanying tests cover replacement, recovery and unaffected paths; all 7 tests passed at the frozen audit commit.

## Impact Details
A trader whose position has just been filled may be unable to add a stop-loss or take-profit. A trader with existing protection may be unable to change it after another account’s order executes. Committing an open order with attached protection also reverts in this state.

Anyone can restore these functions by submitting a current signed basket through the permissionless updateMarkPrice function, but that requires a separate paid update. An externally owned account must refresh and then create protection in separate transactions; another intervening order execution can recreate the condition. Existing protection can still be cancelled, manual closes remain available, and armed protections can trigger with fresh data. No funds are shown to be lost or frozen.

The selected impact is Low: “Contract fails to deliver promised returns, but doesn't lose value.”

## References
Order execution updating the Engine mark:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1053-L1061

Oracle publish-order rejection:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L892-L896

Protection observation and guard:
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L862-L895


## Link to Proof of Concept

https://gist.github.com/0xulver/7e5765c3449a14af4ccc6a2cf58dda24

## Proof of Concept

## Proof of Concept
The runnable Foundry tests are in the secret Gist linked above. The two files needed for the main proof are `ProtectionCreationAfterFill.t.sol` and `ProtectionAfterFillBounds.t.sol`. They run in Plether’s own test suite at the frozen audit commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. No public testnet transactions are required.

1. Refresh the live oracle mark at time T0.
2. Commit an ordinary open order and have a keeper execute it with a valid price tick published at T0+1. Execution advances the Engine mark to T0+1, while the oracle’s accepted live basket remains at T0.
3. Call `createPositionProtection`. It reverts `PletherOracle__PriceOutOfOrder(T0, T0+1)`. `commitOpenOrderWithProtection` also reverts in this state.
4. The second test file shows that replacing an armed protection also reverts. Cancelling protection and other unaffected paths continue to work.
5. Submit a current signed basket through `OrderRouter.updateMarkPrice`, then retry protection creation. It succeeds after the refresh.

To run the tests from the root of a clean checkout at the frozen commit:

```sh
mkdir -p packages/perps/test/perps/poc/protection-after-fill
curl -fL 'https://gist.githubusercontent.com/0xulver/7e5765c3449a14af4ccc6a2cf58dda24/raw/0e99fe6bdfae598dd9856a0a474556eda91d74d4/ProtectionCreationAfterFill.t.sol' -o packages/perps/test/perps/poc/protection-after-fill/ProtectionCreationAfterFill.t.sol
curl -fL 'https://gist.githubusercontent.com/0xulver/7e5765c3449a14af4ccc6a2cf58dda24/raw/254bff144d13442176f2c052e5f3dfa60a0e5295/ProtectionAfterFillBounds.t.sol' -o packages/perps/test/perps/poc/protection-after-fill/ProtectionAfterFillBounds.t.sol
cd packages/perps
forge test --offline --threads 1 --skip script --match-path 'test/perps/poc/protection-after-fill/*.t.sol' -vv
```

The recorded run at the frozen commit completed with **7 tests passed, 0 failed and 0 skipped**. The failing protection calls demonstrate the availability issue; the successful call after a live mark refresh demonstrates its recovery limit.
