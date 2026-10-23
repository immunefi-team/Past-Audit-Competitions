# 52446 sc low withdrawing unsold tokens desynchronizes sale accounting

**Submitted on Aug 10th 2025 at 18:26:11 UTC by @Afriauditor for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52446
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

In `ArcTokenPurchase`, the admin can withdraw unsold `ArcTokens` from the contract, but the sale ledger used by `getMaxNumberOfTokens` isn’t updated. After a withdrawal, the contract may report more tokens “available” than it actually holds, causing failed buys.

### Vulnerability Details

The withdraw function only transfers tokens out; it does not update the sale counters:

```solidity
function withdrawUnsoldArcTokens(address _tokenContract, address to, uint256 amount)
    external onlyTokenAdmin(_tokenContract)
{
    // ...checks...
    ArcToken token = ArcToken(_tokenContract);
    uint256 bal = token.balanceOf(address(this));
    if (bal < amount) revert InsufficientUnsoldTokens();
    bool ok = token.transfer(to, amount);
    if (!ok) revert ArcTokenWithdrawalFailed();
}
```

But the getter used by buyers ignores withdrawals and returns the original pool minus sold:

```solidity
function getMaxNumberOfTokens(address _tokenContract) external view returns (uint256) {
    TokenInfo storage info = _getPurchaseStorage().tokenInfo[_tokenContract];
    return info.totalAmountForSale - info.amountSold;
}
```

### Impact Details

The contract can report more available tokens than it actually holds. Buyers relying on the getter may submit larger purchases and hit reverts when inventory isn’t actually there, wasting gas. The contract does not lose tokens or value, but it can fail to deliver promised amounts.

## Proof of Concept

{% stepper %}
{% step %}

### Step 1 — Setup

* Deploy `ArcToken` and `ArcTokenPurchase`.
* Set the purchase token (e.g., USDC).
* Transfer 1,000 ARC to the purchase contract.
  {% endstep %}

{% step %}

### Step 2 — Start sale

* Enable the sale with `totalAmountForSale = 1,000` and `amountSold = 0`.
  {% endstep %}

{% step %}

### Step 3 — Partial purchases

* Buyers purchase 600 ARC total → `amountSold = 600`; contract now holds 400 ARC.
  {% endstep %}

{% step %}

### Step 4 — Admin withdraws unsold tokens

* Admin calls `withdrawUnsoldArcTokens(token, treasury, 300)` → contract balance drops to 100 ARC.
* Note: `totalAmountForSale` and `amountSold` remain 1,000 and 600 respectively.
  {% endstep %}

{% step %}

### Step 5 — Off-chain or buyer query

* `getMaxNumberOfTokens(token)` returns 400 (computed as `1,000 - 600`), ignoring the 300 withdrawn.
  {% endstep %}

{% step %}

### Step 6 — Buyer attempts purchase

* Buyer attempts to buy 400 ARC based on that getter.
* Inside `buy()`:
  * `remainingForSale = totalAmountForSale - amountSold = 400` → passes the “NotEnoughTokensForSale” check.
  * Actual on-chain balance is 100 ARC → triggers `ContractBalanceInsufficient()` and the tx reverts (buyer wastes gas).
    {% endstep %}
    {% endstepper %}

## References

(No additional references provided.)
