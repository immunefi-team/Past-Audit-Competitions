# #43287 \[BC-Low] Certain fees are unaccounted for causing failed transactions

**Submitted on Apr 4th 2025 at 10:24:09 UTC by @okmxuse for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43287
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Modification of transaction fees outside of design parameters

## Description

### Description

As per the Celestia Docs <https://docs.celestia.org/how-to-guides/submit-data#estimating-pfb-gas>:

> The first transaction sent by an account (sequence number == 0) has an additional one time gas cost of 10,000 gas. If this is the case, this should be accounted for.

This gas cost must be accounted for each time a user submits their first transaction. However, Movement does not enforce this requirement anywhere in the codebase.

### Impact

As a result, transactions may fail, and since Celestia does not refund fees, the associated gas costs will be lost. Moreover, the user's transaction will not be processed.

We believe this issue warrants a **LOW** severity rating, as it leads to unintended fee modifications outside the protocol’s design scope. The protocol has not been designed to handle this scenario.

Even if this behaviour would somehow be intentional, users should be explicitly informed either through documentation or in-protocol messaging.

### Recommendation

Ensure this scenario is properly addressed—either by enforcing it within the codebase or by clearly notifying users to prevent failed transactions.

## Proof of Concept

### POC

* Bob submits his first transaction using the Movement environment.
* the necessary gas is specified to ensure the transaction is processed smoothly.
* However, the Movement environment does not account for the additional 10,000 gas required for first-time transactions.
* As a result, Bob’s transaction fails, and the gas fee is lost.
