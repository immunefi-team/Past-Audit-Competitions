# #46271 \[SC-Medium] Rewards claiming functionality is broken.

**Submitted on May 27th 2025 at 16:25:16 UTC by @farman1094 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46271
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/AgentVault.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

There is functionality in `AgentVault.sol` to claim rewards but that is broken. Rewards can never be claimed.

## Vulnerability Details

The two function of `AgentVault:: claimDelegationRewards` and `AgentVault:: claimAirdropDistribution` is used to claim rewards.

Both function underside call the `claim` function from rewards manager and distribution

```solidity
// claimAirdropDistribution
 _distribution.claim(address(this), _recipient, _month, false);

// claimDelegationRewards
_rewardManager.claim(address(this), _recipient, _lastRewardEpoch, false, _proofs);
```

Here we sending the hard coded value false. This false value means the rewards (native eth) should be sent instead of the wNative token. Can be confirmed here. <https://github.com/flare-foundation/flare-smart-contracts-v2/blob/8dc7b8c936c960aa7338e0bb903f1966c3bf57ed/contracts/userInterfaces/LTS/RewardsV2Interface.sol#L36C4-L50C48>

```
@param _wrap Indicates if the reward should be wrapped (deposited) to the WNAT contract.
```

But in `AgentVault.sol` we have check we don't allow the native eth without `internalWithdrawal` .

```
    receive() external payable {
        require(internalWithdrawal , "internal use only");
    }
```

So the whole call would revert and the agent owner never we able to claim the rewards.

## Impact Details

There is rewards available which could come from the FTSO (Flare Time Series Oracle) system. By delegating voting power (of wNAT or similar tokens) to signal providers in the FTSO, the vault could earns rewards.

The rewards can also come from airdrop distribution. But cannot be claimed due to the broken functionality as explained above.

So the yield generated can never be claimed. Which is directly financial loss to the agent.

## Proof of Concept

## Proof of Concept

1. Initially owner delegate the collateral For FTSO or other factor.
2. The rewards would come as a FTSO reward because of delegation or in form of airdrop form distribution.
3. Then user will use one of the function `AgentVault:: claimDelegationRewards`or `AgentVault:: claimAirdropDistribution` to claim the rewards.
4. As we are sending the `false` value

```solidity
// claimAirdropDistribution
 _distribution.claim(address(this), _recipient, _month, false);

// claimDelegationRewards
_rewardManager.claim(address(this), _recipient, _lastRewardEpoch, false, _proofs);
```

5. The false means the native eth should be send back instead of reward in form of `wNative` token.
6. But as there is an check in the `AgentVault.sol`

* check: <https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/AgentVault.sol#L54>

```
    receive() external payable {
        require(internalWithdrawal, "internal use only");
    }
```

It would revert, so the rewards can never be claimed
