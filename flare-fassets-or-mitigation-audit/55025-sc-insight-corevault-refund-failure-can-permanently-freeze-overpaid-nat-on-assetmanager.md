# #55025 \[SC-Insight] corevault refund failure can permanently freeze overpaid nat on assetmanager

* Report ID: #55025
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/flare-foundation/fassets/commit/7aa02b62285cd5313032103710c2e083b166bf60>
* Impacts: Permanent freezing of funds

## Description

When an agent transfers underlying to the Core Vault via `transferToCoreVault` and overpays the required fee, the contract attempts to refund the difference using a 100k gas native transfer. If that refund fails (for example, the caller is a contract that reverts in `receive()` or requires more than 100k gas), the return value is ignored and the overpaid NAT remains stuck on the Asset Manager (diamond) contract with no recovery path.

## Vulnerability Details

Relevant snippet from CoreVault:

```solidity
 // pay the transfer fee and return overpaid transfer fee when the difference is larger than gas use
        // (all transfers are guarded by nonReentrant in the facet)
        if (msg.value > transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE * tx.gasprice) {
            Transfers.transferNAT(state.nativeAddress, transferFeeWei);
            Transfers.transferNATAllowFailure(payable(msg.sender), msg.value - transferFeeWei);
        } else {
            Transfers.transferNAT(state.nativeAddress, msg.value);
        }
```

`Transfers.transferNATAllowFailure(address payable, uint256)` returns `bool` to indicate success/failure of the 100k-gas NAT transfer, but the result is ignored here. If the recipient cannot accept the transfer (reverts in `receive()` or needs more than the fixed `TRANSFER_GAS_ALLOWANCE = 100_000` gas), the function returns `false`. Because the return is ignored, the overpaid amount remains held by the asset manager contract and is never forwarded or burned.

For context: the threshold `transferFeeWei + Transfers.TRANSFER_GAS_ALLOWANCE * tx.gasprice` is correct and ensures wei-to-wei comparison. The bug is independent of that threshold — it’s about ignoring the refund failure.

A similar pattern is handled correctly in `CollateralReservations.sol`:

```solidity
bool success = Transfers.transferNATAllowFailure(minter, returnFee);
// if failed, burn the whole fee, otherwise burn the difference
if (!success) {
    Agents.burnDirectNAT(totalFee);
}
```

Preconditions for the issue:

* Caller is the authorized agent vault owner but is a contract that reverts in `receive` or requires >100k gas to accept ETH.
* Caller overpays `msg.value` enough to enter the refund branch.

## Impact

Overpaid NAT remains in the Asset Manager’s balance indefinitely, with no generic sweep or recovery function exposed to return it.

## Recommended mitigation steps

Update `CoreVault.transferToCoreVault` to handle failed refunds the same way as `CollateralReservations` — check the return value of `Transfers.transferNATAllowFailure(...)` and, on failure, execute an alternative such as burning the fee or forwarding it to a recovery/burn address so funds are not permanently stuck.

(Reference to the function: <https://github.com/flare-foundation/fassets/blob/59373cee12e6d2a9fa0a9cc8735bb486faa51b36/contracts/assetManager/library/CoreVault.sol#L45>)

## Proof of Concept

The PoC demonstrates the issue by calling `transferToCoreVault` from a contract whose `receive()` reverts, ensuring the 100k gas refund fails and the overpaid NAT stays in the AssetManager.

PoC helper contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.23;

import "../../userInterfaces/IAssetManager.sol";

/**
 * RefundRejector is a helper contract for PoC.
 * - It can be set as the agent's work address so that AssetManager recognizes it as the agent owner.
 * - It calls AssetManager.transferToCoreVault from within the contract (so msg.sender is this contract).
 * - Its receive() reverts, which makes Transfers.transferNATAllowFailure(...) return false when trying to refund.
 * - It has a payable deposit() method to allow funding it despite the reverting receive().
 */
contract RefundRejector {
    // allow funding via explicit function call
    function deposit() external payable {}

    // revert on plain ETH transfers (as used by Transfers.transferNATAllowFailure)
    receive() external payable {
        revert("RefundRejector: reject receive");
    }

    // call transferToCoreVault on AssetManager diamond, forwarding the provided value
    function callTransferToCoreVault(IAssetManager assetManager, address agentVault, uint256 amountUBA) external payable {
        assetManager.transferToCoreVault{ value: msg.value }(agentVault, amountUBA);
    }
}
```

Integration test (create file `test/integration/fasset-simulation/15-CoreVault-RefundPoC.ts`):

```typescript
import { expectRevert, time } from "@openzeppelin/test-helpers";
import { filterEvents } from "../../../lib/utils/events/truffle";
import { BNish, toBN, toWei } from "../../../lib/utils/helpers";
import { getTestFile, loadFixtureCopyVars } from "../../utils/test-helpers";
import { Agent } from "../utils/Agent";
import { AssetContext } from "../utils/AssetContext";
import { CommonContext } from "../utils/CommonContext";
import { Minter } from "../utils/Minter";
import { testChainInfo } from "../utils/TestChainInfo";
import { assertWeb3Equal } from "../../utils/web3assertions";

const RefundRejector = artifacts.require("RefundRejector");

contract(`AssetManagerSimulation.sol; ${getTestFile(__filename)}; CoreVault refund failure PoC`, async accounts => {
    const governance = accounts[10];
    const agentOwner1 = accounts[20];
    const minterAddress1 = accounts[30];
    const underlyingAgent1 = "Agent1";
    const underlyingMinter1 = "Minter1";

    let commonContext: CommonContext;
    let context: AssetContext;

    async function initialize() {
        commonContext = await CommonContext.createTest(governance);
        context = await AssetContext.createTest(commonContext, testChainInfo.xrp);
        // Enable Core Vault so transferToCoreVault passes onlyEnabled checks
        await context.assignCoreVaultManager();
        return { commonContext, context };
    }

    beforeEach(async () => {
        ({ commonContext, context } = await loadFixtureCopyVars(initialize));
    });

    it("PoC: refund failure traps overpaid NAT in AssetManager", async () => {
        // 1) Create agent and make available
        const agent = await Agent.createTest(context, agentOwner1, underlyingAgent1);
        await agent.depositCollateralLotsAndMakeAvailable(100);
        // 2) Mint so that we have backing to transfer
        const minter = await Minter.createTest(context, minterAddress1, underlyingMinter1, context.underlyingAmount(1_000_000));
        const [minted] = await minter.performMinting(agent.vaultAddress, 10);
        await context.updateUnderlyingBlock();

        // 3) Determine transfer amount (all backing)
        const info = await agent.getAgentInfo();
        const transferAmount = info.mintedUBA; // includes pool fee component
        const cbTransferFee = await context.assetManager.transferToCoreVaultFee(transferAmount);
        const overpay = toWei(10); // exceeds TRANSFER_GAS_ALLOWANCE * gasprice in HH env
        const payTransferFee = cbTransferFee.add(overpay);

        // 4) Deploy refund-rejecting caller and set it as agent work address
        const rr = await RefundRejector.new();
        await context.agentOwnerRegistry.setWorkAddress(rr.address, { from: agent.ownerManagementAddress });

        // 5) Observe balances before
        const amBefore = toBN(await web3.eth.getBalance(context.assetManager.address));
        const rrBefore = toBN(await web3.eth.getBalance(rr.address));

        // 6) Initiate transfer to core vault via RefundRejector (msg.sender = rr)
        await rr.callTransferToCoreVault(context.assetManager.address, agent.vaultAddress, transferAmount, { value: payTransferFee });

        // 7) Balances after: refund fails, so overpayment remains in AssetManager
        const amAfter = toBN(await web3.eth.getBalance(context.assetManager.address));
        const rrAfter = toBN(await web3.eth.getBalance(rr.address));

        // AssetManager balance increased by overpayment only (transfer fee forwarded out)
        assertWeb3Equal(amAfter.sub(amBefore), overpay);
        // RefundRejector didn't receive refund (receive() reverts)
        assertWeb3Equal(rrAfter.sub(rrBefore), 0);
    });
});
```

Run the test with:

yarn hardhat test test/integration/fasset-simulation/15-CoreVault-RefundPoC.ts

Logs from the PoC run:

```
  Contract: AssetManagerSimulation.sol; test/integration/fasset-simulation/15-CoreVault-RefundPoC.ts; CoreVault refund failure PoC
    ✔ PoC: refund failure traps overpaid NAT in AssetManager (78ms)


  1 passing (1s)

✨  Done in 4.18s.
```

## Steps to reproduce (high-level)

{% stepper %}
{% step %}

### Prepare environment

Deploy the AssetManager/fassets stack and ensure Core Vault is enabled (so `transferToCoreVault` is allowed).
{% endstep %}

{% step %}

### Create agent and mint

* Create an agent and make it available.
* Perform minting to create backing for the agent so there is underlying to transfer.
  {% endstep %}

{% step %}

### Deploy RefundRejector and set as agent work address

* Deploy the `RefundRejector` helper contract (its `receive()` reverts).
* Set `RefundRejector` as the agent's work address so that it is recognized as the agent owner.
  {% endstep %}

{% step %}

### Trigger transfer with overpayment

* Call `transferToCoreVault` from `RefundRejector`, forwarding a `msg.value` that overpays the transfer fee enough to trigger the refund branch.
* The contract will attempt the 100k-gas refund to `RefundRejector`, which will fail due to its reverting `receive()`.
  {% endstep %}

{% step %}

### Observe balances

* AssetManager balance increases by the overpaid amount.
* RefundRejector does not receive the refund — funds remain stuck on AssetManager.
  {% endstep %}
  {% endstepper %}
