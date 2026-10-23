# 51946 sc high commission claims fail for removed reward tokens

**Submitted on Aug 6th 2025 at 19:23:49 UTC by @aksoy for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51946
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

Validators are unable to request their accrued commissions if a reward token is removed from the contract. Although stakers can still claim their pending rewards for removed tokens, validators are blocked by a missing logic path, leading to unrecoverable commissions.

### Vulnerability Details

The `requestCommissionClaim` function enforces a `_validateIsToken` modifier that reverts if the provided token is not marked as an active reward token in `isRewardToken`. Once a token is removed via `removeRewardToken`, its `isRewardToken[token]` mapping is set to `false`, causing any future commission claims to revert with `TokenDoesNotExist(token)`. Validators cannot request commission claims because of the strict token validation enforced.

```solidity
    modifier _validateIsToken(
        address token
    ) {
        if (!PlumeStakingStorage.layout().isRewardToken[token]) {
            revert TokenDoesNotExist(token);
        }
        _;
    }

    function requestCommissionClaim(
        uint16 validatorId,
        address token
    )
        external
        onlyValidatorAdmin(validatorId)
        nonReentrant
        _validateValidatorExists(validatorId)
        _validateIsToken(token)
    {
        ...
    }
```

### Impact Details

Validators may permanently lose access to their accrued commissions for any token that gets removed.

While stakers can still claim removed tokens’ rewards due to historical checkpointing, validators lack such an alternative path.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L508>

## Proof of Concept

Add test to file :: `plume/test/PlumeStakingDiamond.t.sol`

```solidity
    function testRemovedTokenRequestCommmission() public {
        uint16 validatorId = DEFAULT_VALIDATOR_ID;
        address token = address(pUSD);
        // address recipient = validatorAdmin; // Not used by name

        // Set up commission
        vm.startPrank(validatorAdmin);
        ValidatorFacet(address(diamondProxy)).setValidatorCommission(
            validatorId,
            10e16
        ); // 10%
        vm.stopPrank();

        // Set reward rate and fund treasury
        vm.startPrank(admin);
        address[] memory tokensToSet = new address[](1); // Renamed
        tokensToSet[0] = token;
        uint256[] memory ratesToSet = new uint256[](1); // Renamed
        ratesToSet[0] = 1e18; // 1 PUSD per second
        RewardsFacet(address(diamondProxy)).setRewardRates(
            tokensToSet,
            ratesToSet
        );
        // pUSD.transfer(address(treasury), 2000 ether); // Ensure enough funds
        pUSD.transfer(address(treasury), 10e24); // Increased funding
        vm.stopPrank();

        // Stake to accrue commission
        vm.startPrank(user1);
        StakingFacet(address(diamondProxy)).stake{value: 10 ether}(validatorId);
        vm.stopPrank();

        // Advance time to accrue commission
        vm.warp(block.timestamp + 1 days);

        vm.startPrank(admin);
        RewardsFacet(address(diamondProxy)).removeRewardToken(token);
        vm.stopPrank();

        // Request commission claim
        vm.startPrank(validatorAdmin);
        uint256 tsBeforeRequest = block.timestamp; // Capture timestamp BEFORE request
        vm.expectRevert(
            abi.encodeWithSelector(TokenDoesNotExist.selector, token)
        );
        ValidatorFacet(address(diamondProxy)).requestCommissionClaim(
            validatorId,
            token
        );
    }
```
