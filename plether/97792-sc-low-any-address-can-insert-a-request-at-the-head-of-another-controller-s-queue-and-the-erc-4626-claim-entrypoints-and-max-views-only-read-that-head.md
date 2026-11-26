# \#97792 [SC-Low] Any address can insert a request at the head of another controller's queue, and the ERC-4626 claim entrypoints and `max*` views only read that head

**Submitted on Nov 3rd 2026 at 08:34:07 UTC by @W3B_SR_0xki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97792
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

**Target:** `packages/perps/src` tree at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (TrancheVault.sol)
**Impact:** Insight — Security best practices

## Description

Each TrancheVault controller has one queue of deposit requests and one queue of redeem requests, ordered by request epoch. The ERC-4626 entrypoints and the `max*` views read only the queue head:

- **Claims** use `controllerDepositHead` / `controllerRedeemHead` (TrancheVault.sol:953-1012):
  - `deposit(assets, receiver[, controller])`
  - `mint(shares, receiver[, controller])`
  - `withdraw(assets, receiver, controller)`
  - `redeem(shares, receiver, controller)`
- **Views** read the same head (TrancheVault.sol:1285-1311): `maxDeposit`, `maxMint`, `maxWithdraw`, `maxRedeem`.

The head can be created by an address other than the controller:

- `requestDeposit(assets, controller, owner)` checks only that the caller is `owner` or its operator (`_requireRequestDepositPreflight`, TrancheVault.sol:1574-1584). It does not check the controller.
- `requestRedeem(shares, controller, owner)` likewise authorizes only `owner` (TrancheVault.sol:606-631). The minimum request size does not apply when the owner redeems its full balance (`_requireMinimumRedeemSize`, TrancheVault.sol:1611-1618), so the inserted request can be dust.

A request created for a controller is added to that controller's queue (`_appendControllerDeposit` / `_appendControllerRedeem`, TrancheVault.sol:1742-1796). If it lands in an earlier epoch than the controller's own request, it becomes the head.

From then on, until the controller claims that head through a separate transaction:
- `maxWithdraw(controller)` and `maxRedeem(controller)` report only the head's claimable amount.
- `withdraw` and `redeem` act only on the head, so a call sized to the controller's own funded request reverts with `ERC4626ExceededMaxWithdraw` / `ERC4626ExceededMaxRedeem`.

The same applies on the deposit side.

## Impact

- Nothing is lost:
  - The controller receives the inserted request's value.
  - Its own request can always be claimed with the explicit-id functions `claimRedeem(requestId, …)` and `claimDeposit(requestId, …)`.
- Integrations that use only the standard ERC-4626/7540 surface misbehave in this state:
  - An integration that sizes claims from `maxWithdraw` / `maxRedeem` and claims through `withdraw` / `redeem` sees an understated amount, or a revert, until it handles the extra head entry.
  - Any address can put a controller in this state for the cost of a dust request.
- This is not documented. SECURITY.md "LP claim and allocation limitations" covers receiver rules and rounding, but not head-only aggregation or requests created by third parties.

## Recommendation

Any one of the following:
- Have the id-less entrypoints and `max*` views walk the controller's claimable requests in order, aggregating across them, instead of reading only the head.
- Require controller consent for requests created by another address: `msg.sender == controller` or an approved operator of the controller. ERC-7540 allows vaults to add this.
- Document that the id-less entrypoints and `max*` views reflect only the oldest request. Point integrators to `claimDeposit` / `claimRedeem` with an explicit request id, and to `PerpsPublicLens.getLpRequestState` for per-request balances.

## Proof of Concept

## Proof of Concept
