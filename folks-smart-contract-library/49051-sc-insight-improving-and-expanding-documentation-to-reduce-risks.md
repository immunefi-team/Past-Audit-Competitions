# #49051 \[SC-Insight] Improving and expanding documentation to reduce risks

**Submitted on Jul 10th 2025 at 23:53:20 UTC by @uhudo for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #49051
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/Upgradeable.py>
* **Impacts:**

## Description

## Intro

As upgrading of smart contract is a high security risk and features a few particularities specific to Algorand, it would be useful to expand and clarify the library's documentation with descriptions of: protecting from external upgrades, pre-allocating maximum program size and storages, clear documentation of storage used, and clarity on when upgrades can be cancelled or overwritten.

## Details

Multiple areas of improving library documentation should be considered to mitigate possible risks for applications that end up using it:

* Comment on L54 can be misleading, as it states `then no account will be able to upgrade the contract.`. As upgrades are a high security risks, this should be expanded by a clarification `by using this library. If the application includes any methods with "allow_actions=["UpdateApplication"]", the application might still be upgradeable.`
* When upgrading applications on Algorand, they cannot increase the maximum size and number of global & local storage slots used. To prevent unexpected surprises for library users, it would be good to put a disclaimer about this.
* Overall, the library itself does not document what storage it uses, which can be ambiguous especially in the case of Upgrade contract as it uses dynamic handling of global storage as well as inherits from other contracts. Providing an explicit documentation of all storage variables used would help with the use of library, similarly as was provided with the audit competition.
* Scheduled contract upgrades can be cancelled or overwritten even after the time from which they should be active has been reached because they must be completed via `complete_contract_upgrade` call. I.e. if an upgrade has not yet been completed, it can be cancelled or overwritten even if its delay to deployment has already passed. This should be expressed more explicitly to prevent possible misunderstandings.

*P.S. There is "t" missing in "existing" at L91.*

## Proof of Concept

## Proof of Concept

Included in main description.
