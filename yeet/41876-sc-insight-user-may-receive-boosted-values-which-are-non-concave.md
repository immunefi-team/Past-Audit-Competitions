# #41876 \[SC-Insight] User may receive boosted values which are non-concave

**Submitted on Mar 19th 2025 at 03:58:15 UTC by @h2134 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41876
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Yeet.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The boosted values user receives by specifying nft tokens when yeets are non-concave, this lead to user receive unexpected amount of reward tokens, and the the power of big NFT holders is not properly limited.

## Vulnerability Details

The [Official Document](https://docs.yeetit.xyz/yeet/yeet-game/nftboostweight) states that a concave curve is applied to the calculation of boosted value, holding greater numbers of NFTs applies increased boost but the marginal benefit should be non-increasing.

> A concave curve is applied to this boost in order to limit the power of big holders

To achieve this, `nftBoostLookup` is defined as the NFT boost lookup table.

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L187-L214>

```
    uint256[26] public nftBoostLookup = [
        0,
        345,
        540,
        675,
        765,
        840,
        900,
        960,
        1005,
        1050,
        1080,
        1100,
        1155,
        1185,
        1215,
        1245,
        1275,
        1305,
        1335,
        1365,
        1380,
        1400,
        1440,
        1455,
        1470,
        1500
    ];
```

However, when we draw the curve, we can find the curve is not always concave and there are infection points where the curve becomes convex.

For example, from `10` to `11`, the increased weight ratio is `1.018` (1100 / 1080), however from `11` to `12`, the increased ratio `1.05` (1155 / 1100), therefore there is an infection point at `11` makes the curve convex.

Please find the curve image in the attachment.

## Impact Details

The concave curve is applied to limit the power of big holders, fail to do so results in user receives unexpected amount of reward tokens, when the curve becomes convex, a big holder can receive more reward tokens than expected as they gain more boosted values by providing more NFTs, whereas the small holders receive less rewards.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Yeet.sol#L187-L214>

## Proof of Concept

## Proof of Concept

Take the example of points \[10, 11, 12], because the curve is expected to be concave, then the boosted values gained from `10` and `12` should be no larger than the 2x value of the boosted value gained from `11`, however the POC show otherwise:

Please run `forge test --mt testAudit_NFTBoost_LookUp -vv` in `Yeet.Test.sol`:

```
    function testAudit_NFTBoost_LookUp() public {
        address alice = makeAddr("Alice");
        nft.mintAmount(alice, 25);

        uint256[] memory aliceTokenIds1 = new uint256[](10);
        for (uint i = 0; i < aliceTokenIds1.length; ++i) {
            aliceTokenIds1[i] = i;
        }
        uint256 aliceBoostedValue1 = yeet.getBoostedValue(alice, 0.1 ether, aliceTokenIds1);
        console.log("Alice boostedValue1:", aliceBoostedValue1);

        uint256[] memory aliceTokenIds2 = new uint256[](12);
        for (uint i = 0; i < aliceTokenIds2.length; ++i) {
            aliceTokenIds2[i] = i;
        }
        uint256 aliceBoostedValue2 = yeet.getBoostedValue(alice, 0.1 ether, aliceTokenIds2);
        console.log("Alice boostedValue2:", aliceBoostedValue2);

        uint256 aliceTotalBoostedValue = aliceBoostedValue1 + aliceBoostedValue2;
        console.log("Alice total boostedValue:", aliceTotalBoostedValue);

        console.log();

        address bob = makeAddr("Bob");
        nft.mintAmount(bob, 25);

        uint256[] memory bobTokenIds1 = new uint256[](11);
        for (uint i = 0; i < bobTokenIds1.length; ++i) {
            bobTokenIds1[i] = i;
        }
        uint256 bobBoostedValue1 = yeet.getBoostedValue(bob, 0.1 ether, bobTokenIds1);
        console.log("Bob boostedValue1:", bobBoostedValue1);

        uint256[] memory bobTokenIds2 = new uint256[](11);
        for (uint i = 0; i < bobTokenIds2.length; ++i) {
            bobTokenIds2[i] = i;
        }
        uint256 bobBoostedValue2 = yeet.getBoostedValue(bob, 0.1 ether, bobTokenIds2);
        console.log("Bob boostedValue2:", bobBoostedValue2);

        uint256 bobTotalBoostedValue = bobBoostedValue1 + bobBoostedValue2;
        console.log("Bob total boostedValue:", bobTotalBoostedValue);

        // The cruve is non-concave at points [10, 11, 12]
        assertLt(bobTotalBoostedValue, aliceTotalBoostedValue);
    }
```
