# 57336 sc insight explicit precise and consistent use of application and address types and variable naming prevents bugs

**Submitted on Oct 25th 2025 at 10:26:34 UTC by @uhudo for** [**Audit Comp | Folks Finance: Wormhole NTT on Algorand**](https://immunefi.com/audit-competition/audit-comp--folks-finance-wormhole-ntt-on-algorand)

* Report ID: #57336
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/ntt\\_contracts/transceiver/WormholeTransceiver.py>
* Impacts: (none explicitly listed)

## Description

### Intro

Throughout the codebase, variables use generic types (e.g., UInt64) rather than more specific ones (e.g., Application). This forces frequent, unnecessary, and complex type casting. Variable naming is also generic. Both choices increase the likelihood of mistakes — including the one behind the vulnerability reported in submission #57333.

### Details

Example: in `Transceiver` (source): <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/912c92d5219efe94ae707389da85c93e17e7e36b/ntt\\_contracts/transceiver/Transceiver.py#L17>

The global state `transceiver_manager` is defined as a generic `UInt64` (see L26), but it represents an `Application` (its ID). Because of this, code uses explicit lookups and casts, for example in `send_message()`:

```
transceiver_manager_address, exists = op.AppParamsGet.app_address(self.transceiver_manager.value)
assert exists, err.TRANSCEIVER_MANAGER_ADDRESS_UNKNOWN
assert transceiver_manager_address == Txn.sender, err.TRANSCEIVER_MANAGER_CALLER
```

If the variable were defined with an `Application` type and a clearer name, the same logic could be expressed more directly and readably:

```
assert self.transceiver_manager_app.address == Txn.sender, err.TRANSCEIVER_MANAGER_CALLER
```

This improves readability and enables stronger type hints and compiler/static checks, reducing the chance of bugs such as the one reported in #57333.

## Proof of Concept

The PoC is included in the main description (see code examples above).
