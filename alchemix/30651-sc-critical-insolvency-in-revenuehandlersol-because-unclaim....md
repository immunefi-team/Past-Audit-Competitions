# 30651 - \[SC - Critical] Insolvency in RevenueHandlersol because unclaim...

Submitted on May 3rd 2024 at 02:32:14 UTC by @Django for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #30651

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/RevenueHandler.sol>

Impacts:

* Protocol insolvency
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `RevenueHandler.sol` contract accepts the repayments from the Alchemix protocol and splits it to users based on their locked VE positions. However, this contract will eventually reach a state of insolvency because unclaimed revenue is counted as new revenue for each newly-checkpointed epoch. Users will have a cumulative higher claimable balance than token balance in the contract.

## Vulnerability Details

The Revenue Handler contract has a `checkpoint()` function that must be called once at the beginning of each epoch (every 2 weeks). This functions takes the revenue obtained from the previous period and allots it to users based on the contract's token balances.

The issue arises due to the fact that previously-allotted and unclaimed revenue will still count toward these token balances, double-counting them for allotment.

```
    function checkpoint() public {
        // only run checkpoint() once per epoch
        if (block.timestamp >= currentEpoch + WEEK /* && initializer == address(0) */) {
            currentEpoch = (block.timestamp / WEEK) * WEEK;


            ...


                uint256 thisBalance = IERC20(token).balanceOf(address(this));


                // If poolAdapter is set, the revenue token is an alchemic-token
                if (tokenConfig.poolAdapter != address(0)) {
                    // Treasury only receives revenue if the token is an alchemic-token
                    treasuryAmt = (thisBalance * treasuryPct) / BPS;
                    IERC20(token).safeTransfer(treasury, treasuryAmt);


                    // Only melt if there is an alchemic-token to melt to
                    amountReceived = _melt(token);


                    // Update amount of alchemic-token revenue received for this epoch
                    epochRevenues[currentEpoch][tokenConfig.debtToken] += amountReceived;
                } else {
                    // If the revenue token doesn't have a poolAdapter, it is not an alchemic-token
                    amountReceived = thisBalance;


                    // Update amount of non-alchemic-token revenue received for this epoch
                    epochRevenues[currentEpoch][token] += amountReceived;
                }
```

This line double accounts for previously-unclaimed revenue.

`uint256 thisBalance = IERC20(token).balanceOf(address(this));`

Then the users are able to claim their portion of the claimable revenue based on the `_claimable()` function which directly referrenced the `epochRevenues` mapping that has already double-counted.

```
uint256 epochTotalVeSupply = IVotingEscrow(veALCX).totalSupplyAtT(epochTimestamp);
            if (epochTotalVeSupply == 0) continue;
            uint256 epochRevenue = epochRevenues[epochTimestamp][token];
            uint256 epochUserVeBalance = IVotingEscrow(veALCX).balanceOfTokenAt(tokenId, epochTimestamp);
            totalClaimable += (epochRevenue * epochUserVeBalance) / epochTotalVeSupply;
```

## Impact Details

* Insolvency due to users being able to claim more than the contract's token balance
* Early claimers will be able to claim more than the last claimers, who will not be able to claim anything.

## Output from POC

This POC simply sets up a token position, accrues revenue and checkpoints once, waits another epoch period, and checkpoints again. Since the user never claimed, the claimable revenue is doubled even though no new revenue was accrued.

```
[FAIL. Reason: assertion failed] testClaimNonAlchemicRevenueInsolvency() (gas: 1723846)
Logs:
  (1) Claimable:  1000000000000000000000
  (2) Claimable:  2000000000000000000000
  Error: Claim amount should not go up because of unclaimed revenue
  Error: a == b not satisfied [uint]
    Expected: 2000000000000000000000
      Actual: 1000000000000000000000
```

## Proof of Concept

```
    function testClaimNonAlchemicRevenueInsolvency() external {
        uint256 revAmt = 1000e18;
        uint256 tokenId = _setupClaimableNonAlchemicRevenue(revAmt, bal);
        uint256 balBefore = IERC20(bal).balanceOf(address(this));

        assertEq(balBefore, 0, "should have no bal before claiming");

        uint256 claimable1 = revenueHandler.claimable(tokenId, bal);
        console.log("(1) Claimable: ", claimable1);

        hevm.warp(block.timestamp + ONE_EPOCH_TIME);
        revenueHandler.checkpoint();

        uint256 claimable2 = revenueHandler.claimable(tokenId, bal);
        console.log("(2) Claimable: ", claimable2);

        assertEq(claimable1, claimable2, "Claim amount should not go up because of unclaimed revenue");
    }
```
