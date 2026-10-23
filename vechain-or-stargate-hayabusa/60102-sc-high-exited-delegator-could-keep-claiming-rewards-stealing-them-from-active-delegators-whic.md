# 60102 sc high exited delegator could keep claiming rewards stealing them from active delegators which would then lead to freeze of funds

**Submitted on Nov 18th 2025 at 17:55:17 UTC by @rzizah for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60102
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Smart contract unable to operate due to lack of token funds
  * Temporary freezing of funds for at least 24 hour

## Description / Brief

For a delegator to exit delegation from an Active validator they call `requestDelegationExit`, which directly removes this delegator's stakes from effective stakes via `_updatePeriodEffectiveStake`.

If, after calling `requestDelegationExit`, the delegator does nothing further (doesn't unstake/redelegate) while the validator keeps completing periods, the delegator can repeatedly call `claimRewards`. The first claim is capped to the delegation end period (the requested exit time), but subsequent calls can continue to claim rewards that should belong to currently active delegators — effectively stealing rewards and potentially depleting the contract token balance, which may later block other delegators from claiming or unstaking.

## Vulnerability Details

{% stepper %}
{% step %}

### Scenario (high-level)

* A delegator delegates to an active validator.
* Their delegation becomes active and they can claim rewards.
* The delegator calls `requestDelegationExit` while the validator is still active.
* `requestDelegationExit` calls `_updatePeriodEffectiveStake`, which immediately decreases effective stake for future periods, but does not delete the delegation id while the validator is active (the delegation remains in status ACTIVE with a requested exit flag).
* Time passes: the delegator does nothing further (doesn't unstake).
* The validator continues completing periods. When the exited delegator later calls `claimRewards`, they can claim beyond the end period:
  * The first claim is capped at the end period (the requested exit).
  * Subsequent claims can claim rewards from the current active effective stake because the condition that would cap them no longer applies, allowing the exited delegator to keep claiming rewards intended for active delegators.
* This results in theft of rewards and can leave the contract with insufficient token balance, causing other functions that rely on `claimRewards` (like `unstake`) to revert or block user funds.
  {% endstep %}

{% step %}

### Relevant code paths

* `requestDelegationExit` decreases effective stake via:

  * <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L567-L569>

  ```solidity
  // decrease the effective stake
  _updatePeriodEffectiveStake($, delegation.validator, _tokenId, completedPeriods + 2, false);
  ```
* `claimRewards` calls `_claimRewards`, which obtains claimable periods:

  * <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L790-L791>

  ```solidity
  (uint32 firstClaimablePeriod, uint32 lastClaimablePeriod) = _claimableDelegationPeriods($, _tokenId);
  ```
* In `_claimableDelegationPeriods`, delegations that ended are capped by `endPeriod`, but the condition can be bypassed for subsequent calls:

  * <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L963-L973>

  ```solidity
  if (
      endPeriod != type(uint32).max &&
      endPeriod < currentValidatorPeriod &&
      endPeriod > nextClaimablePeriod
  ) {
      return (nextClaimablePeriod, endPeriod);
  }
  ```

  Because the state can be such that `endPeriod > nextClaimablePeriod` will never be true on later calls, an exited delegator may continue to be considered eligible to claim rewards beyond their intended end period. This allows an exited delegator to claim rewards that should be computed using current effective stakes (belonging to active delegators), draining the contract balance.
  {% endstep %}
  {% endstepper %}

## Impact Details

* Stealing of rewards that should belong to active delegators.
* Contract token balance depletion, which may cause later calls (e.g., other delegators' `claimRewards` or `unstake`) to revert due to insufficient token balance.
* Temporary freezing of funds for other delegators until additional tokens are transferred into the contract to cover the shortfall.

The freeze can be remedied by transferring the missing token balance to the contract.

## References

* claimable delegation periods: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L963-L973>
* \_claimRewards: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L790-L791>
* changing lastclaimedperiod: <https://github.com/vechain/stargate-contracts/blob/877f294a132bf3fd9b51821c5f58b9f9e91c60c1/packages/contracts/contracts/Stargate.sol#L815-L816>

## Proof of Concept (POC)

This POC demonstrates how an exited delegator can repeatedly call `claimRewards` and extract rewards meant for active delegators. It includes:

* A local config snippet to use a local environment.
* A Hardhat integration test `DelegationExitRewardsClaimBug.test.ts` showing the attack flow and the resulting balances and reverts.

Files and commands:

* Add `local.ts` into `packages/config/` with this content (unchanged):

```ts
import { AppConfig } from ".";

const config: AppConfig = {
  environment: "local",
  basePath: "http://localhost:3000",
  ipfsPinningService: "https://api.gateway-proxy.vechain.org/api/v1/pinning/pinFileToIPFS",
  ipfsFetchingService: "https://api.gateway-proxy.vechain.org/ipfs",
  legacyNodesContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  stargateNFTContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  stargateDelegationContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  nodeManagementContractAddress: "0x45d5CA3f295ad8BCa291cC4ecd33382DE40E4FAc",
  stargateContractAddress: "0x00000000000000000000000000005374616B6572",
  protocolStakerContractAddress: "0x00000000000000000000000000005374616B6572",
  protocolParamsContractAddress: "0x0000000000000000000000000000506172616d73",
  indexerUrl: "http://localhost:8080/api/v1",
  nodeUrl: "http://localhost:8669",
  network: {
    id: "solo",
    name: "solo",
    type: "solo",
    defaultNet: true,
    urls: [
      "http://localhost:8669"
    ],
    explorerUrl: "https://explore-testnet.vechain.org",
    blockTime: 10000,
    genesis: {
      id: "0x0000000089970f535c92d8f2151346f002755b4cf6f7fb4b731317fc6df8ee51"
    }
  },
  cyclePeriods: [
    { value: 18, label: "3 minutes" },
    { value: 180, label: "30 minutes" },
    { value: 8640, label: "1 day" },
  ],
}
export default config;
```

* Create `test/integration/DelegationExitRewardsClaimBug.test.ts` with the provided test code (keeps original content unchanged). Then run:

```sh
VITE_APP_ENV=local npx hardhat test test/integration/DelegationExitRewardsClaimBug.test.ts
```

The test script (excerpt) reproduces:

* Two delegators delegate to the same active validator.
* Delegator1 requests exit (state remains ACTIVE but requested to exit).
* Validator completes multiple periods.
* Delegator1 (exited) calls `claimRewards` multiple times:
  * The first claim yields rewards for periods they were active.
  * Subsequent claims extract additional rewards (stealing), increasing delegator1's balance and draining the contract.
* Delegator2 (still active) then cannot claim due to insufficient contract balance and their `claimRewards` reverts with an insufficient balance error.

Sample logs from the run (kept verbatim):

<details>

<summary>Click to expand logs</summary>

```sh
  Delegation Exit Rewards Claim Bug POC

=== Step 1: delegator1 delegates to validator ===
Validator effective stake after both delegations: 3000000000000000000 (period 2)
Validator effective stake after delegator1 exit request: 1500000000000000000 (period 4)
delegator1 claimable periods: 2 to 3
delegator1 VTHO balance before claim: 0
delegator1 VTHO balance after claim: 100000000000000000
delegator1 rewards claimed: 100000000000000000
delegator1 VTHO balance after claim for periods he wsn't active in: 400000000000000000
delegator1 rewards claimed(stolen): 300000000000000000
delegator2 VTHO balance before claim check: 0
delegator2 claimable periods: 2 to 6
delegator2 claimableRewards amount: 400000000000000000
delegator2 claim reverted with error: VM Exception while processing transaction: reverted with custom error 'ERC20InsufficientBalance("0x59b670e9fA9D0A427751Af201D676719a970857b", 0, 400000000000000000)'
    ✔ POC: Exited delegator can keep claiming rewards for active delegation periods (158ms)
```

</details>

## Notes / Mitigation Ideas (not exhaustive, for triage)

* Ensure that when `requestDelegationExit` is called and the effective stake is decreased for future periods, the delegation's claimable periods and/or last claimed period state are updated in a way that prevents subsequent claims beyond the actual exit period.
* Make sure `_claimableDelegationPeriods` and related state transitions cannot be manipulated by leaving a delegation in an ACTIVE-with-exit-request state that still permits claiming from current effective stake.
* Consider revoking or adjusting rights to claim rewards immediately upon an exit request, or introducing a robust guard ensuring a delegation that has requested exit cannot continue to claim from future periods once their effective stake is adjusted.
* Add unit/integration tests (like this POC) to validate no further claims are possible by exited delegators across validator period transitions.

If you want, I can:

* Convert the POC test into a reduced reproducible script or a minimal unit test,
* Suggest exact contract patches (code-level) to prevent this behavior, based on the referenced functions.
