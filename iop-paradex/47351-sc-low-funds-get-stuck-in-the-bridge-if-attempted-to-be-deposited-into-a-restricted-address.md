# #47351 \[SC-Low] Funds get stuck in the bridge if attempted to be deposited into a restricted address

**Submitted on Jun 12th 2025 at 22:29:48 UTC by @Kalogerone for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #47351
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When depositing from the bridge to an operator, sub-operator or auxiliary account, the transaction will revert and funds will remain stuck in the L1 bridge.

## Vulnerability Details

During bridge deposits, the registry gets called to check for restrictions:

```
        fn _deposit_from_bridge(
            ref self: ComponentState<TContractState>,
            recipient: ContractAddress,
            token_address: ContractAddress,
            amount_u256: u256,
        ) {
            let token_dispatcher = ERC20ABIDispatcher { contract_address: token_address };
            let decimals = token_dispatcher.decimals();

            // Sender is 0 (unknown) for bridge deposits
            self._detect_transfer_restriction(Zero::zero(), recipient, token_address, amount_u256);
```

However, according to registry, transfers from `others` going towards operators, sub-operators or auxiliary accounts is restricted and the transaction will revert:

```
            // Restricted:
            //   8. others -> operator
            //   9. others -> auxiliary
            //   10. others -> sub-operator
```

## Impact Details

Any tokens deposited through the bridge to such accounts will not succeed and tokens will stay locked in L1 bridge.

## References

<https://github.com/tradeparadex/audit-competition-may-2025/blob/main/paraclear/src/paraclear/paraclear.cairo#L1248>

## Proof of Concept

## Proof of Concept

1. User deposits to L1 bridge with destination an operator, sub-operator or auxiliary account
2. L2 bridge fails to complete the deposit
3. Funds are stuck in L1 bridge
