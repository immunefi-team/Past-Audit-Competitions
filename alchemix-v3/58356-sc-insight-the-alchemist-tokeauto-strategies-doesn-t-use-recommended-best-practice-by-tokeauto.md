# 58356 sc insight the alchemist tokeauto strategies doesn t use recommended best practice by tokeauto&#x20;

**Submitted on Nov 1st 2025 at 14:55:25 UTC by @Davuka for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58356
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Violating recommended best practice

## Description

## Brief/Intro

Alchemist allocates collateral into Auto Finance without validating whether the destination vault’s debt data is stale before performing read/write operations against the AutoPool. As a result, strategies may receive wrong share amounts due to the use of stale debt report data.

## Vulnerability Details

> It is our recommendation to check for stale debt reporting before performing a read or write operation against the Autopool. While our systems strive to ensure that the reporting information is always up to date, outages or other network issues have the possibility to prevent this. Should the debt reporting data be stale, users shares and or value can be misrepresented.

> To check for this state the Autopool exposes a oldestDebtReporting() function. If this returned timestamp is older than 1 day, you should prevent your operation from executing.

From the Auto finance documentation, the above recommendation was stated for all integrating protocols.

It is important to note that Tokemak implements an internal function totalAssetsTimeChecked to biasly handle scenarios whereby the debt report is stale. This function does not revert when stale data is detected instead, it applies extreme pricing rules (ceiling for deposits, floor for withdrawals) to protect Tokemak’s Autopool from suffering losses. However, this fallback mechanism only protects Tokemak itself. It does not guarantee that integrators will receive accurate valuations or correct share assignments. As a result, integrators such as Alchemist must still enforce a stale-data cutoff check themselves.

```solidity
     function totalAssetsTimeChecked(
        StructuredLinkedList.List storage debtReportQueue,
        mapping(address => AutopoolDebt.DestinationInfo) storage destinationInfo,
        IAutopool.TotalAssetPurpose purpose
    ) external returns (uint256) {
        IDestinationVault destVault = IDestinationVault(debtReportQueue.peekHead());
        uint256 recalculatedTotalAssets = IAutopool(address(this)).totalAssets(purpose);

        while (address(destVault) != address(0)) {
            uint256 lastReport = destinationInfo[address(destVault)].lastReport;

            if (lastReport + MAX_DEBT_REPORT_AGE_SECONDS > block.timestamp) {
                // Its not stale

                // This report is OK, we don't need to recalculate anything
                break;
            } else {
                // It is stale, recalculate

                //slither-disable-next-line unused-return
                uint256 currentShares = destVault.balanceOf(address(this));
                uint256 staleDebt;
                uint256 extremePrice;

                // Figure out exactly which price to use based on its purpose
                if (purpose == IAutopool.TotalAssetPurpose.Deposit) {
                    // We use max value so that anything deposited is worth less
                    extremePrice = destVault.getUnderlyerCeilingPrice();

                    // Round down. We are subtracting this value out of the total so some left
                    // behind just increases the value which is what we want
                    staleDebt = destinationInfo[address(destVault)].cachedMaxDebtValue.mulDiv(
                        currentShares, destinationInfo[address(destVault)].ownedShares, Math.Rounding.Down
                    );
                } else if (purpose == IAutopool.TotalAssetPurpose.Withdraw) {
                    // We use min value so that we value the shares as worth less
                    extremePrice = destVault.getUnderlyerFloorPrice();
                    // Round up. We are subtracting this value out of the total so if we take a little
                    // extra it just decreases the value which is what we want
                    staleDebt = destinationInfo[address(destVault)].cachedMinDebtValue.mulDiv(
                        currentShares, destinationInfo[address(destVault)].ownedShares, Math.Rounding.Up
                    );
                } else {
                    revert InvalidTotalAssetPurpose();
                }

                // Back out our stale debt, add in its new value
                // Our goal is to find the most conservative value in each situation. If the current
                // value we have represents that, then use it. Otherwise, use the new one.
                uint256 newValue = (currentShares * extremePrice) / destVault.ONE();

                if (purpose == IAutopool.TotalAssetPurpose.Deposit && staleDebt > newValue) {
                    newValue = staleDebt;
                } else if (purpose == IAutopool.TotalAssetPurpose.Withdraw && staleDebt < newValue) {
                    newValue = staleDebt;
                }

                recalculatedTotalAssets = recalculatedTotalAssets + newValue - staleDebt;
            }

            destVault = IDestinationVault(debtReportQueue.getAdjacent(address(destVault), true));
        }

        return recalculatedTotalAssets;
    }
```

## Impact Details

Alchemist Toke strategies do not enforce checks for stale debt reporting before depositing or withdrawing assets from Toke autopools. In situations where the Autopool data is outdated, Alchemist may receive fewer shares minted than they should.

## References

Link to the totalAssetsTimeChecked: <https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/vault/libs/AutopoolDebt.sol#L349>

Link to the doc recommendation: <https://docs.auto.finance/developer-docs/integrating/checking-for-stale-data>

## Proof of Concept

## Proof of Concept

a) Clone the auto finance repo: <https://github.com/Tokemak/v2-core-pub> b) Create StaleDebt.t.sol file at test/unit/vault/StaleDebt.t.sol

c) run this command: forge test --match-path test/unit/vault/StaleDebt.t.sol --match-test test\_ExistingPriceUsedWhenStaleDestinationRepriceIsLower -vvv

```solidity
// SPDX-License-Identifier: UNLICENSED
// Copyright (c) 2023 Tokemak Foundation. All rights reserved.
pragma solidity >=0.8.7;
import {AutopoolETHTests, DestinationVaultFake} from "./Autopool.t.sol";
import { IAutopool } from "src/interfaces/vault/IAutopool.sol";
import { Test, console } from "forge-std/Test.sol";

contract StaleDebt is  AutopoolETHTests {

      function test_ExistingPriceUsedWhenStaleDestinationRepriceIsLower() public {
        emit log_named_uint("Oldest debt reporting", vault.oldestDebtReporting());

        address userA = makeAddr("user1");
        address userB = makeAddr("user2");

      uint shareA =  _depositFor(userA, 10e18);

        // Mimic a deployment
        DestinationVaultFake destVault = _setupDestinationVault(
            DVSetup({
                autoPool: vault,
                dvSharesToAutopool: 10e18,
                valuePerShare: 1e9,
                minDebtValue: 9e9,
                maxDebtValue: 11e9,
                lastDebtReportTimestamp: block.timestamp - 2 days // Make the data stale
             }),
            18
        ); 
  

        emit log_named_uint("Oldest debt reporting", block.timestamp - vault.oldestDebtReporting());

        // We had a valuePerShare of 1e18 when we deployed, lets value each LP at 0.5e18
        // This is the idea that when a pool is attacked and skewed to one side we will take the highest priced
        // Token and value all of the reserves at that price, giving the user the worst execution but still letting
        // it go through and relying on their slippage settings. However, when our existing price is higher,
        // keep using it
        _mockDestVaultCeilingPrice(address(destVault), 0.5e18);

        uint256 actualShares = _depositFor(userB, 10e18);

       //  console.log("Calculated share balance", calculatedShares);

       //  console.log("Calculated userB shares", shareA);
         emit log_named_uint("Calculated userA balance e%", shareA);
         emit log_named_uint("Calculated userB balance e%", actualShares);

       // assertEq(calculatedShares, actualShares, "shares");
    }
}
```
