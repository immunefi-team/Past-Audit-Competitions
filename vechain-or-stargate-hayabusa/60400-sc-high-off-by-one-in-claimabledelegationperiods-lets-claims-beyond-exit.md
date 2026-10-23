# 60400 sc high off by one in claimabledelegationperiods lets claims beyond exit

**Submitted on Nov 22nd 2025 at 07:40:31 UTC by @AgentJacker for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #60400
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/Stargate.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Protocol insolvency

## Description

## Brief/Intro

The Stargate.sol contract has an off by one error in the `claimableDelegationPeriods` function that allows users to claimn rewards for periods after their delegation has ended.

## Vulnerability Details

The issue is in the if statement logic, it doesnt check for a specific edge case of endPeriod = nextClaimablePeriod

```
        if (
            endPeriod != type(uint32).max &&
            endPeriod < currentValidatorPeriod &&
            endPeriod > nextClaimablePeriod
        ) {
            return (nextClaimablePeriod, endPeriod);
        }
```

it uses strict inequality when comparing endPeriod with nextClaimablePeriod. When `endPeriod == nextClaimablePeriod` the condition evaluates to false, causing the function to

* skip the ended delegation logic entirely
* fall through to the active delegation branch
* return (nextClaimablePeriod, completedPeriods) as if the delegation is still active
* allow claiming rewards for periods where the NFT has no active delegation

this causes ended delegations as active delegations, enabling post-exit reward theft

an example flow is

* user delegates nft and later requests exit so endperiod is set to a specific period
* at the boundary condition where nextClaimablePeriod == endPeriod `endPeriod > nextClaimablePeriod` which is 2 > 2 = false
* the ended delegation check fails, code continues to

````if
      return (nextClaimablePeriod, completedPeriods);  
  }```
- Returns periods extending to completedPeriods
- _claimableRewardsForPeriod doesn't verify period membership so it calculates and pays rewards for non-delegated periods

## Impact Details
- Protocol insolvency

## Link to Proof of Concept

https://gist.github.com/agentjacker/ef4c9f61d5a4f5fbcf83ef4e51638fee

## Proof of Concept

## Proof of Concept

- move the gist test file https://gist.github.com/agentjacker/ef4c9f61d5a4f5fbcf83ef4e51638fee into packages/contracts/test/integration
- cd into packages/contracts
- run VITE_APP_ENV=local MNEMONIC="denial kitchen pet squirrel other broom bar gas better priority spoil cross" npx hardhat test --network vechain_solo test/integration/PostExitLeakageVulnerability.test.ts
- Ensure Docker is installed and running

here is an example 
````

* Users effective stake = 1,000,000 VET
* Total validator delegated stake = 10,000,000 VET
* Users share = 10 percent of rewards
* VTHO rewards per period = 10,000 VTHO
* Users legitimate rewards (0 periods) = 0 VTHO
* Validator operates for 100 periods after user's exit

Stolen Rewards Calculation:

* User's share per period = (1,000,000 / 10,000,000) × 10,000 = 1,000 VTHO
* Periods stolen = 100 (periods 3-102)
* Total stolen = 1,000 VTHO × 100 = 100,000 VTHO

At $0.002 per VTHO: $200 stolen per attack At $0.01 per VTHO: $1,000 stolen per attack

```
```

PoC: Post-Exit Leakage Vulnerability - Claims Beyond Exit

\=== PoC: Post-Exit Leakage Vulnerability ===

✓ Staked and matured NFT ✓ Delegated NFT to validator Delegation start period: 2 ✓ Fast-forwarded to period 6

Delegation Details Before Exit: Start Period: 2 End Period: 4294967295 (finite) ✓ Requested delegation exit ✓ Fast-forwarded to period 7 (exit period)

Delegation Details After Exit: End Period: 2

Before Claiming: Last Claimed Period (inferred): 1 First Claimable Period: 2 Last Claimable Period: 7 End Period: 2 Checking if rewards are claimable through endPeriod...

Claimable Rewards Before Claim: 46027397260273972602960 wei Claiming: periods 2 to 7 (endPeriod: 2) Warning: lastClaimable (7) > endPeriod (2)

After Claiming: Last Claimed Period: 0 End Period: 2 lastClaimedAfter: 0, endPeriodAfter: 2 Note: lastClaimed (0) != endPeriod (2) This may indicate the delegation exited before becoming active, or rewards weren't fully claimed

✓ Fast-forwarded 5 more periods to period 18

After Advancing Periods: Last Claimed Period: 0 First Claimable Period: 8 Last Claimable Period: 18 End Period: 2 Current Validator Period: 18

✓ VULNERABILITY CONFIRMED:

* End Period: 2 (delegation ended at this period)
* First Claimable: 8 (BEYOND EXIT!)
* Last Claimable: 18 (extends 16 periods beyond exit) ✔ Demonstrates post-exit leakage allowing claims beyond exit period (25217ms)

1 passing (27s)

```
```
