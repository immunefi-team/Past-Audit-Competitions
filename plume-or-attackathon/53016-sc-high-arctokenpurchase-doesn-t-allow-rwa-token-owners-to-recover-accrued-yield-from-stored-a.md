# 53016 sc high arctokenpurchase doesn t allow rwa token owners to recover accrued yield from stored arctokens waiting for sale&#x20;

**Submitted on Aug 14th 2025 at 17:03:38 UTC by @valkvalue for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53016
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenPurchase.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

`ArcTokenPurchase.sol` can store many `ArcToken` which can accrue rewards during their period. However, there is no mechanism to recover or properly track yield accrued to those stored ArcTokens, resulting in permanently frozen funds.

### Vulnerability Details

`ArcTokenPurchase.sol` provides functionality for owners of `ArcToken` (tokenized RWA) to sell their `ArcToken` for a specific price.

* By default every `ArcToken` can accrue rewards (see `distributeYield`):\
  <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/ArcToken.sol#L388-L460>
* `ArcTokenPurchase.sol` uses a purchase token (set by an admin) as the "currency" to buy different ArcTokens (likely a stablecoin).
* To enable sales, an ArcToken admin calls `enableToken` to deposit tokens and set a price:

```solidity
    function enableToken(
        address _tokenContract,
        uint256 _numberOfTokens,
        uint256 _tokenPrice
@>> ) external onlyTokenAdmin(_tokenContract) {
.....
```

* The same token-admin can withdraw unsold tokens:

```solidity
    function withdrawUnsoldArcTokens(
        address _tokenContract,
        address to,
        uint256 amount
@>> ) external onlyTokenAdmin(_tokenContract) {
        if (to == address(0)) {
            revert CannotWithdrawToZeroAddress();
        }
        if (amount == 0) {
            revert AmountMustBePositive();
        }

        ArcToken token = ArcToken(_tokenContract);
        uint256 contractBalance = token.balanceOf(address(this));
        if (contractBalance < amount) {
            revert InsufficientUnsoldTokens();
        }

        bool success = token.transfer(to, amount);
        if (!success) {
            revert ArcTokenWithdrawalFailed();
        }
    }
```

* There is no logic in `ArcTokenPurchase` to track or recover accrued yield from stored ArcTokens. Yield can be denominated in different tokens (including the `ArcToken` itself), and `yieldTokenAddr` may be different per ArcToken. Without tracking which rewards came from which ArcToken, even recovered funds cannot be properly distributed to original sellers — leading to permanent losses.

### Impact Details

Funds that accrue as yield to ArcTokens deposited in `ArcTokenPurchase` can become irrevocably frozen. Because yield tokens can differ per ArcToken and there is no per-token tracking, sellers cannot be made whole and funds may be permanently lost.

## References

* Contract reference: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/ArcTokenPurchase.sol>

## Proof of Concept

{% stepper %}
{% step %}

### Step 1

ArcToken-owner calls `enableToken` and deposits X ArcTokens.
{% endstep %}

{% step %}

### Step 2

The ArcToken accrues yield for every token holder via `distributeYield`.
{% endstep %}

{% step %}

### Step 3

The `ArcTokenPurchase` contract now holds the yielded funds.
{% endstep %}

{% step %}

### Step 4

There is no mechanism in `ArcTokenPurchase` to recover or attribute these yielded funds back to the ArcToken owners. Additionally, `yieldTokenAddr` can differ per ArcToken, increasing the difficulty and impact of recovery.
{% endstep %}
{% endstepper %}

## Additional notes

<details>

<summary>Full relevant code references</summary>

* ArcToken yield distribution:\
  <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/ArcToken.sol#L388-L460>
* ArcTokenPurchase contract:\
  <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/arc/src/ArcTokenPurchase.sol>

</details>
