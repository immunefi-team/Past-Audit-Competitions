# \#98539 [SC-Insight] Reuse the deposit preflight's terminal-wipe result to save 49,256 gas on a Senior request

**Submitted on Nov 8th 2026 at 14:47:27 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98539
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**

## Description

## Description

`TrancheVault._requireRequestDepositPreflight()` evaluates
`_isTerminallyWiped()` and rejects a wiped tranche with the dedicated
`TrancheVault__TerminallyWiped` error. If that check succeeds, the function
continues without changing relevant state and calls `maxRequestDeposit()`, which
evaluates `_isTerminallyWiped()` again.

Each evaluation calls `totalAssets()`, which invokes the four-return
`HousePool.getPendingTrancheState()` path and builds a current HousePool context.
The second evaluation is therefore redundant on a successful deposit preflight:
the same view predicate was already false in the same call frame before any
state-changing operation.

An isolated successor keeps public `maxRequestDeposit(controller)` behavior
unchanged by routing it through a private helper with terminal-wipe checking
enabled. Only `_requireRequestDepositPreflight()`, after its explicit terminal
guard has passed, calls the helper with that duplicate check disabled.

On an identical local healthy-Senior fixture at frozen commit
`a82d4fefb02768e5b3bddcc97dee371eee24fea8`, the measured gas interval around
`seniorVault.requestDeposit(assets, lp, lp)` decreased from `655,206` to
`605,950` gas. The matched delta is `49,256` gas, or `7.517636%`. Both versions
returned request id `474872` and passed identical pending-assets, escrow, and
Senior-reservation postconditions. Six existing boundary tests also passed on
the successor, including four terminal-wipe rejections, zero-controller
handling, and the dedicated oversized-request error.

The recommendation is to preserve the public view and explicit preflight error,
but reuse the already-established preflight result through a private helper.
This report does not propose caching or removing the other HousePool context
constructions in admission, capacity, reservation, or settlement.


## Proof of Concept

## Proof of Concept / Conditions under which the Insight is valuable

The improvement is valuable when all of the following hold:

1. A caller submits a non-terminally-wiped deposit request, so the first
   `_isTerminallyWiped()` check returns false rather than reverting.
2. Controller and owner authorization, trading state, and minimum deposit checks
   allow execution to reach `maxRequestDeposit()`.
3. The request path pays for the second identical terminal-wipe projection even
   though no relevant state changed between the two calls.

The retained fixture measures one healthy Senior request because that path has a
bounded stateful success result to compare. It does not claim the same absolute
saving for Junior requests, reverts, every pool state, or production frequency.

Minimal correction shape:

```solidity
function maxRequestDeposit(address controller) public view returns (uint256) {
    return _maxRequestDeposit(controller, true);
}

function _maxRequestDeposit(address controller, bool checkTerminalWipe)
    private
    view
    returns (uint256)
{
    if (controller == address(0)) return 0;
    if ((checkTerminalWipe && _isTerminallyWiped())
        || !POOL.canAcceptTrancheDeposits(IS_SENIOR)) return 0;
    return IS_SENIOR ? _seniorDepositCapacity() : type(uint256).max;
}

// _requireRequestDepositPreflight(), after its explicit wipe guard:
uint256 maxAssets = _maxRequestDeposit(controller, false);
```

No public RPC, live target, credential, or submission action was used.
`NO_OUTCOME_ESTIMATES`.
