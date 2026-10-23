# #46121 \[SC-High] Malicious agent can manipulate the totalCollateral to cause damage to the protocol

**Submitted on May 25th 2025 at 07:11:25 UTC by @Cryptor for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46121
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A malicious or compromised agent can exploit the claimDelegationRewards and claimAirdropDistribution functions in the CollateralPool.sol contract to artificially inflate the totalCollateral state variable. This is achieved by providing a malicious reward/distribution contract address that returns a large "claimed" amount without actually transferring the corresponding WNAT to the pool. This manipulation distorts the pool's perceived value, leading to unfair advantages for the agent during payouts (e.g., liquidations where the pool contributes), incorrect collateral ratio calculations, and potential dilution of value for other honest pool participants.

## Vulnerability Details

The CollateralPool.sol contract allows agents to claim delegation rewards and airdrop distributions through the claimDelegationRewards and claimAirdropDistribution functions, respectively. Both functions take an external contract address (\_rewardManager or \_distribution) as a parameter, which is controlled by the agent.

claimDelegationRewards function:

```

    function claimDelegationRewards(//@audit
        IRewardManager _rewardManager,
        uint24 _lastRewardEpoch,
        IRewardManager.RewardClaimWithProof[] calldata _proofs
    )
        external
        onlyAgent
        returns (uint256)
    {
        uint256 claimed = _rewardManager.claim(address(this), payable(address(this)), _lastRewardEpoch, true, _proofs);
        totalCollateral += claimed; // Vulnerable line
        emit ClaimedReward(claimed, 1);
        return claimed;
    }
```

claimAirdropDistribution function:

```
    function claimAirdropDistribution(
        IDistributionToDelegators _distribution,
        uint256 _month
    )
        external
        onlyAgent
        returns(uint256)
    {
        uint256 claimed = _distribution.claim(address(this), payable(address(this)), _month, true);
        totalCollateral += claimed; // Vulnerable line
        emit ClaimedReward(claimed, 0);
        return claimed;
    }
```

The problem is that the CollateralPool contract makes no validation checks on the return values from the claim functions in both the rewardmanager contract and the airdrop distribution contract. It just directly adds the return value to the totalCollateral state variable without verifying that the pool actually received the corresponding amount of WNAT.

A malicious agent can:

Deploy a custom, malicious contract that implements the IRewardManager or IDistributionToDelegators interface. This malicious contract's claim function will be designed to return an arbitrarily large uint256 value, while transferring little or no actual WNAT to the CollateralPool. The agent then calls claimDelegationRewards or claimAirdropDistribution on their CollateralPool, providing the address of their malicious contract. The CollateralPool will call the malicious contract's claim function, receive the inflated claimed amount, and add it to totalCollateral. This results in totalCollateral being significantly higher than the actual WNAT balance held by the pool.

## Impact Details

An artificially inflated totalCollateral has several detrimental impacts on the protocol:

1. Unfair Benefit to Agent During Payouts/Liquidations: The payout function is called by the AssetManager to use pool collateral, often during liquidations. It calculates the number of agent's pool tokens to burn based on \_agentResponsibilityWei and assetData.poolNatBalance (which is derived from totalCollateral).

<https://github.com/flare-labs-ltd/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L661>

```solidity
 poolNatBalance: totalCollateral,
```

The payout function <https://github.com/flare-labs-ltd/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L878-L880>

```
    uint256 toSlashTokenMax = assetData.poolNatBalance > 0 ?
         assetData.poolTokenSupply.mulDiv(_agentResponsibilityWei, assetData.poolNatBalance) : agentTokenBalance;
```

If assetData.poolNatBalance (i.e., totalCollateral) is artificially high, the denominator in the mulDiv is larger. This results in a smaller toSlashTokenMax, meaning fewer of the agent's pool tokens are burned to cover their \_agentResponsibilityWei. The agent unfairly retains a larger share of the pool, effectively reducing their penalty or contribution during liquidation. This can lead to a drain of actual assets from the pool if the agent later exits with these unfairly retained tokens. In alternative scenarios, other pool contributors can execute the same attack, causing damage to the protocol

2. Distorted Collateral Ratios (CR): The Liquidation.sol library uses \_agent.collateralPool.totalCollateral() (via \_getCollateralAmount when \_kind == Collateral.Kind.POOL) to calculate the agent's pool CR.

```
    } else if (_kind == Collateral.Kind.POOL) {
        // Return tracked collateral amount in the pool.
        return _agent.collateralPool.totalCollateral();
    }
```

An inflated totalCollateral will make the agent's pool CR appear healthier than it is. This can prevent or delay necessary liquidations or CCB (Collateral Call Band) triggers, allowing an undercollateralized agent to continue operating and potentially accrue more bad debt, increasing systemic risk.

3. Incorrect Pool Token Pricing for New Entrants: When new users enter the pool, the number of pool tokens they receive for their NAT deposit is influenced by the totalCollateral (via assetData.poolNatBalance in \_collateralToTokenShare). An artificially high totalCollateral makes existing pool tokens seem more valuable, meaning new entrants receive fewer tokens for their NAT, effectively overpaying.
4. Dilution of Honest Pool Participants: If an agent benefits from an inflated totalCollateral (e.g., by having fewer tokens burned during a payout), the "cost" is socialized among other honest pool token holders. Their share of the pool's actual assets is diluted because the agent did not contribute their fair share.

## Proof of Concept

## Proof of Concept

A malicious agent first deploys a custom MaliciousRewardManager contract designed to return a large "claimed" amount without actually transferring WNAT. The agent then calls claimDelegationRewards on their CollateralPool, providing the address of this malicious contract, which artificially inflates the pool's totalCollateral. A new, honest user calls enter on this CollateralPool to deposit their NAT, the \_collateralToTokenShare function uses the inflated totalCollateral (via assetData.poolNatBalance). This makes each existing pool token appear more valuable than it truly is, resulting in the new user receiving significantly fewer pool tokens for their deposited NAT than they should, effectively overpaying.
