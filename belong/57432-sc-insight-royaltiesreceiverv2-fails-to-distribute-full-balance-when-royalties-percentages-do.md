# 57432 sc insight royaltiesreceiverv2 fails to distribute full balance when royalties percentages do not sum to 10000

**Submitted on Oct 26th 2025 at 07:56:36 UTC by @RoadRunner26383 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57432
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/periphery/RoyaltiesReceiverV2.sol>

Impacts:

* Permanent freezing of unclaimed royalties

***

## Description

### Brief/Intro

RoyaltiesReceiverV2 uses a hardcoded 10,000 BPS denominator for royalty payout calculations, while Factory permits creator+platform shares to sum to any value ≤ 10,000 BPS. When the configured sum is below 10,000, the difference between total shares and the fixed denominator results in a residual fraction of every royalty payment that cannot be released to any payee, causing permanent freezing of unclaimed royalties for both native currency and ERC20 tokens. This matches the program's in-scope High-severity impact "Permanent freezing of unclaimed royalties."

***

## Vulnerability Details

Affected Assets

* Primary affected contract:
  * contracts/v2/periphery/RoyaltiesReceiverV2.sol — Fixed `TOTAL_SHARES = 10_000` used in all `_pendingPayment` calculations for both native and ERC20 releases.
* Supporting / contributing component:
  * contracts/v2/platform/Factory.sol — Royalties parameter validation permits `creatorBps + platformBps ≤ 10_000` without requiring equality.

Description

The issue arises due to a mismatch between the royalty denominator fixed at 10,000 in RoyaltiesReceiverV2 and the flexible BPS configuration allowed by Factory.sol (creatorBps + platformBps ≤ 10,000). When the total share is less than 10,000, residual ETH/ERC20 funds become permanently stuck within the RoyaltiesReceiverV2 contract, as the payout function cannot release the remainder to any recipient.

In production, Factory deploys RoyaltiesReceiverV2 via OpenZeppelin's Clones library, and each clone reads royalty shares from the Factory instance during payout calculations.

Root Cause

RoyaltiesReceiverV2 computes each payee's pending payment using the formula:

```
payment = (totalReceived * shares(account)) / TOTAL_SHARES - released[account]
```

where `TOTAL_SHARES` is a constant set to `10_000`:

```solidity
uint256 private constant TOTAL_SHARES = 10_000;
```

However, Factory's `setRoyaltiesParameters` function only enforces that the sum of `amountToCreator` and `amountToPlatform` is **less than or equal to** 10,000 BPS:

```solidity
if (uint256(_royalties.amountToCreator) + uint256(_royalties.amountToPlatform) > 10_000) {
    revert InvalidRoyaltiesParameters();
}
```

Residual arises whenever total configured shares are below the fixed denominator: with totalShares = 1,000 BPS and denominator 10,000, the distributed fraction is 1000/10000 = 10% and the residual is 1 - 1000/10000 = 90%, which is never attributable to any payee.

This allows valid configurations where `creatorBps + platformBps < 10_000`, such as 500 + 500 = 1,000 BPS (10% total). When such a configuration is deployed, the royalty receiver divides incoming funds by 10,000 and allocates only the configured percentage to payees, leaving the remainder permanently stranded in the contract.

Attack/Exploitation Path

No active exploitation is required; the vulnerability is triggered by normal contract usage:

1. Admin configures Factory with royalties where `amountToCreator + amountToPlatform < 10_000` (e.g., creator=500, platform=500).
2. Factory deploys an AccessToken collection via ERC1967Proxy and creates a RoyaltiesReceiverV2 instance via Clones.cloneDeterministic with these parameters.
3. Royalties (native ETH or ERC20) are sent to the RoyaltiesReceiverV2 by marketplaces or direct transfers.
4. When `releaseAll()` is called, only `(creatorBps + platformBps) / 10_000` of the total balance is distributed; the remainder stays in the contract.
5. Repeated calls to `releaseAll()` never distribute the residual because no payee has shares covering the unallocated fraction, resulting in permanent freezing.

Vulnerable Calculation (excerpt from RoyaltiesReceiverV2.sol):

```solidity
function _pendingPayment(
    address account,
    uint256 totalReceived,
    uint256 alreadyReleased
) internal view returns (uint256) {
    return (totalReceived * shares(account)) / TOTAL_SHARES - alreadyReleased;
}
```

The function uses `TOTAL_SHARES = 10_000` regardless of the actual sum of configured shares, creating a mismatch when total configured shares are below this denominator.

***

## Impact Details

Severity Classification: High – Permanent freezing of unclaimed royalties

Financial Impact

Every royalty payment sent to a misconfigured RoyaltiesReceiverV2 results in a proportional loss:

* For a configuration with total shares = 1,000 BPS (10%), 90% of all royalties are permanently frozen.
* For total shares = 5,000 BPS (50%), 50% of all royalties are permanently frozen.

Example Scenario

1. A collection is configured with creator=500 BPS, platform=500 BPS (total 1,000 BPS).
2. The collection generates 100 ETH in secondary-market royalties.
3. After `releaseAll()` calls, only 10 ETH (5 ETH to creator, 5 ETH to platform) is distributed; 90 ETH remains permanently stuck in the RoyaltiesReceiverV2 with no recovery path.
4. The same loss applies to any ERC20 royalties (e.g., USDC, WETH).
5. If a collection accrues 1,000,000 USDC in royalties under a 1,000 BPS configuration, approximately 900,000 USDC becomes permanently frozen.

Protocol-Wide Risk

* Any collection deployed with misconfigured royalties experiences cumulative loss.
* Creators and platform lose revenue, undermining trust in the royalty infrastructure.
* The issue affects all RoyaltiesReceiverV2 instances deployed via Factory with non-10,000 share totals.

Why This Qualifies as In-Scope High Impact

The program explicitly lists "Permanent freezing of unclaimed royalties" as an in-scope High-severity impact. This vulnerability deterministically causes this outcome when Factory's allowed configuration permits total shares < 10,000.

***

## References

Affected In-Scope Contract Files:

* `contracts/v2/periphery/RoyaltiesReceiverV2.sol` — TOTAL\_SHARES constant, \_pendingPayment(), releaseAll();
* `contracts/v2/platform/Factory.sol` — Royalties parameter validation.

Program Scope:

* Immunefi Audit Comp | Belong – Information page: <https://immunefi.com/audit-competition/audit-comp-belong/information/>

External Reference:

* OpenZeppelin PaymentSplitter implementation (correct use of dynamic share denominator): <https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/finance/PaymentSplitter.sol>

Belong Contract Documentation & Build Guides:

* <https://github.com/belongnet/checkin-contracts/tree/main/docs/guides>

Testing Environment:

* Foundry (forge), Solidity 0.8.27, local VM (no fork required)

Proof-of-Concept link:

* <https://gist.github.com/DEX0029/658fc088033ad9926a4db1dee968c18d>

***

## Proof of Concept

Overview

This proof of concept demonstrates that RoyaltiesReceiverV2 permanently freezes royalty payments when the sum of creator and platform basis points is less than 10,000. The PoC uses the actual Belong contracts deployed via their production patterns:

* Factory deployed and initialized via ERC1967Proxy (upgradeable proxy pattern).
* RoyaltiesReceiverV2 deployed via OpenZeppelin Clones (EIP-1167).
* Tests run against real contract logic without mocks.

The PoC includes three test cases: ERC20 residual freeze, native currency residual freeze, and a control test showing zero residual when shares total exactly 10,000 BPS.

Prerequisites

* Git
* Foundry (forge)
* Basic familiarity with Solidity 0.8.27
* Cloned Belong audit competition repository from Immunefi

Setup and PoC Steps

{% stepper %}
{% step %}

### Clone repository

```bash
git clone https://github.com/immunefi-team/audit-comp-belong.git
cd audit-comp-belong
```

{% endstep %}

{% step %}

### Install dependencies

```bash
forge install smartcontractkit/chainlink --no-commit
forge install OpenZeppelin/openzeppelin-contracts-upgradeable --no-commit
forge install OpenZeppelin/openzeppelin-contracts --no-commit
```

{% endstep %}

{% step %}

### Create the test file

```bash
mkdir -p test
touch test/RoyaltiesResidualFreeze_Valid.t.sol
```

{% endstep %}

{% step %}

### Configure foundry.toml (if needed)

Example settings (optional):

```toml
[profile.default]
src = "contracts"
test = "test"
libs = ["lib"]
solc_version = "0.8.27"
optimizer = true
optimizer_runs = 200

remappings = [
"forge-std/=lib/forge-std/src/",
"solady/=lib/solady/",
"@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/",
"@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts/",
"@chainlink/contracts/=lib/chainlink/",
]

exclude = [
"contracts/v1",
"contracts/mocks",
"contracts/v2/tokens/LONG.sol",
"contracts/v2/utils/Helper.sol",
"contracts/v2/platform/BelongCheckIn.sol"
]
```

If compilation issues occur, you can create a separate directory with only the two relevant contracts (RoyaltiesReceiverV2 and Factory) and the test file.
{% endstep %}

{% step %}

### Save the PoC test implementation

Save the provided test file contents to `test/RoyaltiesResidualFreeze_Valid.t.sol` (full implementation included below).
{% endstep %}

{% step %}

### Run tests

Run the full test suite with verbose tracing:

```bash
forge test -vvvv
```

Or run individual tests:

* ERC20 Residual Freeze Test:

```bash
forge test --match-test test_ERC20_RoyaltiesResidualFreeze -vvvv
```

* Native Currency Residual Freeze Test:

```bash
forge test --match-test test_Native_RoyaltiesResidualFreeze -vvvv
```

* Control Test (No Residual):

```bash
forge test --match-test test_Control_NoResidual_WhenTotalShares_10000 -vvvv
```

Optional Gas Report:

```bash
forge test -vvvv --gas-report
```

{% endstep %}
{% endstepper %}

Complete PoC Implementation (test/RoyaltiesResidualFreeze\_Valid.t.sol)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.27;

import "forge-std/Test.sol";
import {Factory} from "contracts/v2/platform/Factory.sol";
import {RoyaltiesReceiverV2} from "contracts/v2/periphery/RoyaltiesReceiverV2.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {Clones} from "@openzeppelin/contracts/proxy/Clones.sol";

// Minimal ERC20 for testing
contract TestERC20 {
    string public name = "TestToken";
    string public symbol = "TT";
    uint8 public decimals = 18;
    mapping(address => uint256) public balanceOf;
    event Transfer(address indexed from, address indexed to, uint256 value);
    function mint(address to, uint256 amt) external { balanceOf[to] += amt; emit Transfer(address(0), to, amt); }
    function transfer(address to, uint256 amt) external returns (bool) {
        uint256 bal = balanceOf[msg.sender]; require(bal >= amt, "bal");
        unchecked { balanceOf[msg.sender] = bal - amt; balanceOf[to] += amt; }
        emit Transfer(msg.sender, to, amt); return true;
    }
}

contract RoyaltiesResidualFreeze_Valid is Test {
    address public creator  = address(0xC0FFEE);
    address public platform = address(0xFEEFEE);

    Factory factory;
    RoyaltiesReceiverV2 receiver;
    TestERC20 token;

    receive() external payable {}

    function setUp() public {
        // Deploy Factory via proxy with sub-10,000 BPS royalties
        Factory factoryImpl = new Factory();

        bytes memory initData = abi.encodeWithSelector(
            Factory.initialize.selector,
            Factory.FactoryParameters({
                platformAddress: platform,
                signerAddress: address(this),
                defaultPaymentCurrency: address(0),
                platformCommission: 0,
                maxArraySize: 1000,
                transferValidator: address(0)
            }),
            Factory.RoyaltiesParameters({
                amountToCreator: 500,
                amountToPlatform: 500
            }),
            Factory.Implementations({
                accessToken: address(0),
                creditToken: address(0),
                royaltiesReceiver: address(0),
                vestingWallet: address(0)
            }),
            [uint16(0), 0, 0, 0, 0]
        );

        ERC1967Proxy proxy = new ERC1967Proxy(address(factoryImpl), initData);
        factory = Factory(address(proxy));

        // Deploy RoyaltiesReceiverV2 implementation
        RoyaltiesReceiverV2 receiverImpl = new RoyaltiesReceiverV2();

        // Create a clone of the receiver implementation (mimics Factory's deployment pattern)
        receiver = RoyaltiesReceiverV2(payable(Clones.clone(address(receiverImpl))));

        // Initialize the cloned receiver
        receiver.initialize(
            RoyaltiesReceiverV2.RoyaltiesReceivers({
                creator:  creator,
                platform: platform,
                referral: address(0)
            }),
            factory,
            bytes32(0)
        );

        token = new TestERC20();
    }

    function test_Native_RoyaltiesResidualFreeze() public {
        vm.deal(address(this), 100 ether);
        (bool ok, ) = address(receiver).call{value: 100 ether}("");
        require(ok, "fund receiver failed");

        receiver.releaseAll(receiver.NATIVE_CURRENCY_ADDRESS());
        assertEq(creator.balance, 5 ether, "creator 5 ETH");
        assertEq(platform.balance, 5 ether, "platform 5 ETH");
        assertEq(address(receiver).balance, 90 ether, "90 ETH residual");

        receiver.releaseAll(receiver.NATIVE_CURRENCY_ADDRESS());
        assertEq(address(receiver).balance, 90 ether, "residual unchanged");
    }

    function test_ERC20_RoyaltiesResidualFreeze() public {
        token.mint(address(this), 100e18);
        require(token.transfer(address(receiver), 100e18), "erc20 fund failed");

        receiver.releaseAll(address(token));
        assertEq(token.balanceOf(creator),  5e18, "creator 5e18");
        assertEq(token.balanceOf(platform), 5e18, "platform 5e18");
        assertEq(token.balanceOf(address(receiver)), 90e18, "90e18 residual");

        receiver.releaseAll(address(token));
        assertEq(token.balanceOf(address(receiver)), 90e18, "residual unchanged");
    }

    function test_Control_NoResidual_WhenTotalShares_10000() public {
        // Second Factory with 10,000 BPS total
        Factory factoryImpl2 = new Factory();
        bytes memory initData2 = abi.encodeWithSelector(
            Factory.initialize.selector,
            Factory.FactoryParameters({
                platformAddress: platform,
                signerAddress: address(this),
                defaultPaymentCurrency: address(0),
                platformCommission: 0,
                maxArraySize: 1000,
                transferValidator: address(0)
            }),
            Factory.RoyaltiesParameters({
                amountToCreator: 6000,
                amountToPlatform: 4000
            }),
            Factory.Implementations({
                accessToken: address(0),
                creditToken: address(0),
                royaltiesReceiver: address(0),
                vestingWallet: address(0)
            }),
            [uint16(0), 0, 0, 0, 0]
        );
        ERC1967Proxy proxy2 = new ERC1967Proxy(address(factoryImpl2), initData2);
        Factory factory2 = Factory(address(proxy2));

        // Clone receiver for control test
        RoyaltiesReceiverV2 receiverImpl2 = new RoyaltiesReceiverV2();
        RoyaltiesReceiverV2 receiver2 = RoyaltiesReceiverV2(payable(Clones.clone(address(receiverImpl2))));
        receiver2.initialize(
            RoyaltiesReceiverV2.RoyaltiesReceivers({
                creator:  creator,
                platform: platform,
                referral: address(0)
            }),
            factory2,
            bytes32(0)
        );

        token.mint(address(this), 100e18);
        require(token.transfer(address(receiver2), 100e18), "erc20 fund failed");
        receiver2.releaseAll(address(token));

        assertEq(token.balanceOf(creator),  60e18, "creator 60e18");
        assertEq(token.balanceOf(platform), 40e18, "platform 40e18");
        assertEq(token.balanceOf(address(receiver2)), 0, "no residual");
    }
}
```

Running the PoC

* The three tests pass and demonstrate:
  * Residual freeze for ERC20 and native currency when total shares < 10,000.
  * No residual when total shares == 10,000.

An example verbose output from `forge test -vvvv` is included in the original report (omitted here for brevity).

***

## Test Breakdown

1. test\_ERC20\_RoyaltiesResidualFreeze
   * Factory configured with creator=500 BPS, platform=500 BPS (10% total).
   * Transfer 100e18 tokens to receiver.
   * `releaseAll(token)` distributes 5e18 to creator, 5e18 to platform; 90e18 remains. Second call distributes nothing further.
2. test\_Native\_RoyaltiesResidualFreeze
   * Same 500/500 BPS configuration.
   * Fund receiver with 100 ETH.
   * `releaseAll(NATIVE_CURRENCY_ADDRESS)` distributes 5 ETH to creator, 5 ETH to platform; 90 ETH remains. Second call distributes nothing further.
3. test\_Control\_NoResidual\_WhenTotalShares\_10000
   * Factory configured with creator=6000 BPS, platform=4000 BPS (100% total).
   * Transfer 100e18 tokens to receiver.
   * `releaseAll(token)` distributes full 60e18/40e18, leaving 0 residual.

***

## Recommended Fixes

1. Primary Fix: Dynamic Share Denominator\
   Replace the fixed `TOTAL_SHARES = 10_000` with dynamic calculation using the actual sum of configured shares. Example adjustment to `_pendingPayment`:

```solidity
function _pendingPayment(
    address account,
    uint256 totalReceived,
    uint256 alreadyReleased
) internal view returns (uint256) {
    uint256 totalShares = shares(_creator) + shares(_platform);
    if (_referral != address(0)) totalShares += shares(_referral);

    if (totalShares == 0) return 0;

    return (totalReceived * shares(account)) / totalShares - alreadyReleased;
}
```

Benefit: Ensures full distributability regardless of configured share percentages.

2. Alternative Fix: Enforce creatorBps + platformBps == 10,000 at configuration\
   Modify Factory validation to require exact 10,000 total:

```solidity
function _setRoyaltiesParameters(RoyaltiesParameters memory _royalties) internal {
    if (uint256(_royalties.amountToCreator) + uint256(_royalties.amountToPlatform) != 10_000) {
        revert InvalidRoyaltiesParameters();
    }
    royaltiesParameters = _royalties;
}
```

Benefit: Keeps fixed denominator invariant intact.

Recommended additional measures:

* Add emergency withdrawal function with timelock for recovering stuck funds after grace period.
* Validate share totals in RoyaltiesReceiverV2 initialization to reject misconfigured deployments.
* Add residual tracking view function to expose unallocated balance for monitoring.
* Extend tests to cover edge cases (single payee, zero shares, referral configurations).

***

## Responsible Disclosure

This vulnerability was discovered during the Immunefi Audit Competition for Belong Network and is disclosed according to the program's Responsible Publication Category 3 policy. The issue affects in-scope smart contracts within the competition's listed assets and aligns with the program's high-severity impact: "Permanent freezing of unclaimed royalties."

***

## Conclusion

The PoC conclusively demonstrates that RoyaltiesReceiverV2 permanently freezes a proportional amount of every royalty payment when configured with share totals below 10,000 BPS. The vulnerability is deterministic, irreversible under current contract interfaces, affects both native currency and ERC20 tokens, and directly maps to the program's listed High-severity impact.

If you want, I can:

* Produce a minimal patch diff for RoyaltiesReceiverV2 showing the dynamic share change.
* Produce a small Factory patch enforcing equality to 10,000 if you prefer the invariant approach.
