# #37196 \[W\&A-Insight] DOS due to Misleading 'CircularProgressBar' Display Due to Rounding of 'supplyU

**Submitted on Nov 28th 2024 at 12:37:25 UTC by @Brainiac5 for** [**IOP | SwayLend Frontend**](https://immunefi.com/audit-competition/swaylend-frontend-iop)

* **Report ID:** #37196
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/Swaylend/swaylend-monorepo/tree/develop/apps/frontend>
* **Impacts:**
  * Temporarily disabling user to access target site, such as:
* Locking up the victim from login
* Cookie bombing, etc.

## Description

## Brief/Intro

The `CircularProgressBar` component in `CollateralTable.tsx` rounds the percentage of supply used for a collateral asset. This rounding error, specifically rounding 99.95% to 100%, creates a **significant** psychological denial-of-service (DoS) condition. Users are falsely led to believe a collateral asset is fully supplied and are thus unable to deposit, even when a small amount of the supply cap remains available. **Critically, while the rounding error might seem small (0.05%), the impact is magnified considerably given that individual collateral supply caps often run into tens of millions of dollars. This means even a seemingly negligible rounding error can result in substantial portions of the supply cap potentially remaining perpetually unused.** If unchecked on mainnet, this could severely hinder liquidity and negatively impact the overall functionality and profitability of the lending platform.

## Vulnerability Details

The vulnerability lies within the calculation and display of the supply utilization percentage in both `CollateralTableRow` and `CollateralCard` components. The percentage is calculated as:

```typescript:
supplyUsed = collateralAmount.div(formatUnits(BigNumber(collateralConfiguration.supply_cap.toString()), decimals)).times(100);
```

This `supplyUsed` value is then passed to the `CircularProgressBar` component, which visually represents the percentage. The problem arises from the implicit rounding behavior of the `div` and potentially other operations within the `formatUnits` function. When the `supplyUsed` value is very close to 100%, such as 99.95%, the rounding results in 100%, leading to the visual representation of a completely full progress bar with the value reading '100%'

This is demonstrated in the following code snippet from `CollateralTableRow`:

```typescript
const supplyUsed = collateralAmount
      .div(
        formatUnits(
          BigNumber(collateralConfiguration.supply_cap.toString()),
          decimals
        )
      )
      .times(100);

// ... later in the component ...
<CircularProgressBar percent={supplyUsed.div(100)} />
```

The `percent` prop passed to `CircularProgressBar` is derived from `supplyUsed`, which, as explained above, can be incorrectly rounded to 100. This misrepresentation prevents users from supplying additional collateral, even if a small amount of the supply cap remains.

## Impact Details

The impact of this vulnerability is a psychological denial of service condition affecting users attempting to supply collateral. While the rounding error of 0.05% might seem insignificant, it represents a substantial amount of unused supply cap when considering large supply caps in millions of dollars. For example, a supply cap of $20,000,000 would have $10,000 remaining largely unused due to this rounding error. This lost liquidity in some ways impacts the platform's ability to facilitate lending and borrowing to it's fullest capabilities, potentially reducing its overall efficiency and profitability. Furthermore, the misleading display could damage user trust and negatively affect the platform's reputation.

## Mitigation

Instead of relying on implicit rounding within the BigNumber library's division or other operations, explicitly round the percentage to a sufficient number of decimal places before passing it to the CircularProgressBar component. This ensures that values very close to 100% are not rounded up prematurely. Consider using toFixed() or similar methods to control the precision.

## References

* swaylend-monorepo/apps/frontend/src/components/DashboardView/AssetsTable/CollateralTable.tsx (`The relevant code file`)

## Proof of Concept

## Proof of Concept

```typescript

const CollateralTableRow = ({
  account,
  assetId,
  symbol,
  decimals,
  protocolBalance,
  handleAssetClick,
  collateralConfiguration,
  collateralAmount,
  price,
}: TableRowProps) => {
  // ... other code ...

  const testerNumber = new BigNumber(99.95); // Added for POC: Simulates a near-full supply scenario

  // ... other code ...

 
  // let supplyUsed = BigNumber(0);

  // if (collateralAmount.gt(0)) {
  //   supplyUsed = collateralAmount
  //     .div(
  //       formatUnits(
  //         BigNumber(collateralConfiguration.supply_cap.toString()),
  //         decimals
  //       )
  //     )
  //     .times(100);
  // }

  // ... other code ...

  return (
    <TableRow>
      {/* ... other code ... */}
      <TableCell>
        <div className="w-[48px] h-[48px]">
          <CircularProgressBar percent={testerNumber.div(100)} /> {/* Using testerNumber for POC */}
        </div>
      </TableCell>
      {/* ... other code ... */}
    </TableRow>
  );
};

```

This will render the `CircularProgressBar` with 100% displayed on the frontend.
