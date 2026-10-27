# 50677 sc insight redundant code in dexaggregatorwrapperwithpredicateproxy impairs readability and potentially increases gas costs

**Submitted on Jul 27th 2025 at 12:14:16 UTC by @KKam86 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50677
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol>
* **Impacts:** (see details below)

## Description

### Brief/Intro

One of the functions in `DexAggregatorWrapperWithPredicateProxy` - `depositOneInch` has redundant code which unnecessarily complicates function logic. Redundant code impairs readability, produces additional gas costs or even can introduce new errors in contract logic.

### Vulnerability Details

{% stepper %}
{% step %}

### Redundant vault address validation (multiple checks)

Function `depositOneInch` validates `vaultAddress` three separate times:

1. First time inside call to `_oneInchHelper`:

```solidity
uint256 supportedAssetAmount = _oneInchHelper(
    supportedAsset,
    address(teller),
    executor,
    desc,
    data,
    nativeValueToWrap
);
```

Inside `_oneInchHelper` before approving assets to `vaultAddress`:

```solidity
address vaultAddress = address(
    TellerWithMultiAssetSupport(payable(teller)).vault()
);
if (vaultAddress == address(0)) {
    revert("DexAggregatorWrapper: Invalid vault address for approval");
}

supportedAsset.safeApprove(vaultAddress, supportedAssetAmount);
```

2. Second check immediately in `depositOneInch` after calling `_oneInchHelper` and before `deposit`:

```solidity
address vaultAddress = address(teller.vault());
if (vaultAddress == address(0)) {
    // Handle error: Vault address cannot be zero if we need to transfer shares
    revert("DexAggregatorWrapper: Invalid vault address");
}
```

3. Third check at the end of `depositOneInch` inside `_calcSharesAndEmitEvent` call path:

```solidity
_calcSharesAndEmitEvent(
    supportedAsset,
    CrossChainTellerBase(address(teller)),
    address(desc.srcToken),
    desc.amount,
    supportedAssetAmount
);
```

And inside `_calcSharesAndEmitEvent` it again obtains and checks the vault address:

```solidity
// Get vault address
address vaultAddress = address(teller.vault());
if (vaultAddress == address(0)) {
    revert("DexAggregatorWrapper: Invalid vault address");
}
```

These repeated checks for the same invariant are redundant and increase code complexity and gas usage.
{% endstep %}

{% step %}

### Unnecessary shares recalculation in `_calcSharesAndEmitEvent`

The call to `_calcSharesAndEmitEvent` in `depositOneInch` recomputes shares solely to emit a `Deposit` event, but `teller.deposit` already returns the shares calculated using the same logic.

Example in `depositOneInch`:

```solidity
shares = teller.deposit(
    supportedAsset,
    supportedAssetAmount,
    minimumMint
);
```

In `TellerWithMultiAssetSupport.sol` the `deposit` function returns shares via internal `_erc20Deposit`:

```solidity
function deposit(
    ERC20 depositAsset,
    uint256 depositAmount,
    uint256 minimumMint
) external requiresAuth nonReentrant returns (uint256 shares)
```

And `_erc20Deposit` calculates shares as:

```solidity
shares = depositAmount.mulDivDown(
    ONE_SHARE,
    accountant.getRateInQuoteSafe(depositAsset)
);
```

Where `ONE_SHARE` and `accountant` are initialized in `TellerWithMultiAssetSupport` constructor.

Meanwhile, shares calculation inside `_calcSharesAndEmitEvent` duplicates the same computation:

```solidity
uint256 shares = supportedAssetAmount.mulDivDown(
    10 ** teller.vault().decimals(),
    AccountantWithRateProviders(teller.accountant()).getRateInQuoteSafe(
        supportedAsset
    )
);
```

This redundant recalculation is unnecessary since the shares were already computed and returned by `teller.deposit`.
{% endstep %}
{% endstepper %}

## Impact Details

1. Increased gas costs — redundant operations consume additional gas which the user must pay for.
2. Reduced readability — redundant and repeated checks make the code harder to reason about, maintain, and audit, increasing risk of errors.

## References

* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L100>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L275-L278>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L106-L110>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L114-L120>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L397-L400>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol?utm\\_source=immunefi#L103>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/base/Roles/TellerWithMultiAssetSupport.sol#L353>
* <https://github.com/immunefi-team/attackathon-plume-network-nucleus-boring-vault/blob/main/src/helper/DexAggregatorWrapperWithPredicateProxy.sol#L401-L404>

## Proof of Concept

(Empty)

## Proof of Concept

(Empty)
