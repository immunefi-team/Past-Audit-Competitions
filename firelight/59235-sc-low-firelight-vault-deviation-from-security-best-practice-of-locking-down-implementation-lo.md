# 59235 sc low firelight vault deviation from security best practice of locking down implementation logic&#x20;

**Submitted on Nov 10th 2025 at 08:35:26 UTC by @blackgrease for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59235
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

**Affected Files:** `FirelightVault.sol`

The Firelight Vault is an upgrade-able ERC-4626 vault. Even though it is upgrade-able, it does not lock down the implementation logic allowing it to be directing initialized.

The best practice from Openzeppelin for upgradeable contracts - like the Firelight Vault - is to lock down the implementation logic.

"An uninitialized contract can be taken over by an attacker. This applies to both a proxy and its implementation contract, which may impact the proxy. To prevent the implementation contract from being used, you should invoke the \_disableInitializers function in the constructor to automatically lock it when it is deployed:" - <https://docs.openzeppelin.com/contracts/5.x/api/proxy#UUPSUpgradeable>

The Firelight Vault does not follow this security best practice allowing for the Firelight Vault Implementation logic to be taken over by anyone.

This will result in undesired consequences if users mistakenly interact with the Raffle implementation logic. The proxy logic remains unaffected because the storage is different

## Impact

The impact for this issue is an Insight, under "Security Best Practices" and "Code Optimizations and Enhancements". Locking down the implementation logic prevents anyone from claiming ownership.

The current contract logic is a deviation from Security Best Practices.

## Mitigation

The recommended mitigation is to lock down the implementation logic thus preventing direct initialization by adding the line `constructor(){ _disableInitializers(); }`

Adding the above fix, will cause the PoC to fail with the revert `InvalidInitialization()`.

## Link to Proof of Concept

<https://gist.github.com/blackgrease/ff37911d49ac94c94d80a1056ae4af0d>

## Proof of Concept

The provided Foundry PoC - gist link - shows the implementation logic can be taken over. It deploys the Vault behind a proxy - following normal protocol deployment - and then takes over the implementation logic as an attacker address. The proxy remains unaffected as even though deposits are made to the implementation logic and proxy, the proxy records its `totalAssets` correctly.

(Implementing the provided fix will cause the test to revert with `InvalidInitialization()`)

* Run with: `forge test --mt testFirelightVaultImplementationTakeover -vvv`

### Logs

```
[PASS] testFirelightVaultImplementationTakeover() (gas: 1644099)
Logs:
  Vault Assets in protocol owned Firelight Vault:  10000000000000000000
  Vault Assets in attacker owned Firelight Vault:  10000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.03ms (1.83ms CPU time)

```
