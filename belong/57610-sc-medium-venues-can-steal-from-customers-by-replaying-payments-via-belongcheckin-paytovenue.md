# 57610 sc medium venues can steal from customers by replaying payments via belongcheckin paytovenue&#x20;

**Submitted on Oct 27th 2025 at 15:39:22 UTC by @blackgrease for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57610
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

**Affected Files:** `BelongCheckIn.sol`

The `BelongCheckIn` contract coordinates venue deposits, customer check-ins and promoter settlements. The `BelongCheckIn::payToVenue` function processes a customer's payment to a venue.

The payment logic is gated: a `Signer` address must sign message input parameters before a payment proceeds — specifically the fields in the `CustomerInfo` struct: `paymentInUSDC, visitBountyAmount, spendBountyPercentage, customer, venueToPayFor, promoter, amount, block.chainid`.

However, once signed, `payToVenue` does not enforce logic to:

* prevent repeating a signed payment to the same venue,
* control who can call/execute `payToVenue`.

As a result, if a user has approved the `BelongCheckIn` contract for USDC or LONG, a venue can replay the signed message and drain more funds than the intended single payment.

Below are two exploitation scenarios (USDC used as the payment method).

Scenario 1:

{% stepper %}
{% step %}
A victim has a balance of 5000 USDC and has 3 venue payments to make.
{% endstep %}

{% step %}
Venue\_A requires 500 USDC, Venue\_B requires 2500 USDC, Venue\_C requires 2000 USDC. All inputs are signed by the `Signer`.
{% endstep %}

{% step %}
The victim approves the `BelongCheckIn` contract for 5000 USDC.
{% endstep %}

{% step %}
The victim makes the initial payment to Venue\_A.
{% endstep %}

{% step %}
Because `payToVenue` does not:

* check the caller, and
* enforce a nonce preventing replay, Venue\_A can call/frontrun the victim's following `payToVenue` calls repeatedly until the user's balance is depleted.
  {% endstep %}

{% step %}
Venue exits with the user's funds causing user loss.
{% endstep %}
{% endstepper %}

Scenario 2:

{% stepper %}
{% step %}
A victim has 500 USDC and grants infinite approval to the `BelongCheckIn` contract for convenience.
{% endstep %}

{% step %}
Venue\_A requires a payment of 500 USDC.
{% endstep %}

{% step %}
The victim approves the `BelongCheckIn` for the required USDC amount and completes the payment to Venue\_A.
{% endstep %}

{% step %}
At that moment the victim's balance is 0, so immediate replays would fail.
{% endstep %}

{% step %}
The victim later receives 10,000 USDC to their address for other uses.
{% endstep %}

{% step %}
Because `payToVenue` does not:

* check the caller, and
* enforce a nonce preventing replay, Venue\_A can call `payToVenue` again for the previously-signed payment (500 USDC) and drain the newly deposited funds.
  {% endstep %}

{% step %}
Venue exits with the user's funds causing user loss.
{% endstep %}
{% endstepper %}

### The Problematic code

The vulnerable code path allows this behavior because there is no nonce, deadline, nor caller restriction on execution:

```solidity
//@audit: no nonce or logic to prevent this payment from happening again.
function payToVenue(CustomerInfo calldata customerInfo) external {
        BelongCheckInStorage memory _storage = belongCheckInStorage;
        VenueRules memory rules = generalVenueInfo[customerInfo.venueToPayFor].rules;

        _storage.contracts.factory.nftFactoryParameters().signerAddress.checkCustomerInfo(customerInfo, rules);

        uint256 venueId = customerInfo.venueToPayFor.getVenueId();

        if (customerInfo.promoter != address(0)) {
            uint256 rewardsToPromoter = customerInfo.paymentInUSDC
                ? customerInfo.visitBountyAmount + customerInfo.spendBountyPercentage.calculateRate(customerInfo.amount)
                : _storage.paymentsInfo.usdc
                    .unstandardize(
                        // standardization
                        _storage.paymentsInfo.usdc.standardize(customerInfo.visitBountyAmount)
                            + customerInfo.spendBountyPercentage
                                .calculateRate(
                                    _storage.paymentsInfo.long
                                        .getStandardizedPrice(
                                            _storage.contracts.longPF,
                                            customerInfo.amount,
                                            _storage.paymentsInfo.maxPriceFeedDelay
                                        )
                                )
                    );
            uint256 venueBalance = _storage.contracts.venueToken.balanceOf(customerInfo.venueToPayFor, venueId);
            require(venueBalance >= rewardsToPromoter, NotEnoughBalance(rewardsToPromoter, venueBalance));

            _storage.contracts.venueToken.burn(customerInfo.venueToPayFor, venueId, rewardsToPromoter);
            _storage.contracts.promoterToken
                .mint(customerInfo.promoter, venueId, rewardsToPromoter, _storage.contracts.venueToken.uri(venueId));
        }

        if (customerInfo.paymentInUSDC) {
            _storage.paymentsInfo.usdc
                .safeTransferFrom(customerInfo.customer, customerInfo.venueToPayFor, customerInfo.amount); //@audit-issue: e.g if USDC has been approved for other venues (or infinite approvals), then a venue can abuse the approval. Can be called by anyone to execute the payment...including venue
        } else {
            // platform subsidy - processing fee
            uint256 subsidyMinusFees =
                _storage.fees.platformSubsidyPercentage.calculateRate(customerInfo.amount)
                - _storage.fees.processingFeePercentage.calculateRate(customerInfo.amount);
            _storage.contracts.escrow
                .distributeLONGDiscount(customerInfo.venueToPayFor, address(this), subsidyMinusFees);

            // customer paid amount - longCustomerDiscountPercentage (3%)
            uint256 longFromCustomer =
                customerInfo.amount - _storage.fees.longCustomerDiscountPercentage.calculateRate(customerInfo.amount);
            _storage.paymentsInfo.long.safeTransferFrom(customerInfo.customer, address(this), longFromCustomer); //@audit-issue: e.g if LONG has been approved for other venues (or infinite approvals), then a venue can abuse the approval.  Can be called by anyone to execute the payment...including venue

            uint256 longAmount = subsidyMinusFees + longFromCustomer;

            if (rules.longPaymentType == LongPaymentTypes.AutoStake) {
                // Approve only what is needed, then clear allowance after deposit.
                _storage.paymentsInfo.long.safeApproveWithRetry(address(_storage.contracts.staking), longAmount);
                _storage.contracts.staking.deposit(longAmount, customerInfo.venueToPayFor);
                _storage.paymentsInfo.long.safeApprove(address(_storage.contracts.staking), 0);
            } else if (rules.longPaymentType == LongPaymentTypes.AutoConvert) {
                _swapLONGtoUSDC(customerInfo.venueToPayFor, longAmount);
            } else {
                _storage.paymentsInfo.long.safeTransfer(customerInfo.venueToPayFor, longAmount);
            }
        }

        emit CustomerPaid(customerInfo.customer, customerInfo.venueToPayFor, customerInfo.promoter, customerInfo.amount, customerInfo.visitBountyAmount, customerInfo.spendBountyPercentage);
    }
```

## Impact

A user’s USDC/LONG funds can be stolen. Venues are not controlled by the protocol and must be treated as untrusted actors. Even if venues are initially vetted, they could turn malicious later and exploit this replay behavior.

## Mitigation

Two possible mitigations are proposed:

{% hint style="info" %}

1. Add a nonce (and optionally a deadline) to the signed `CustomerInfo` payload and store used nonces to prevent replays.
2. Restrict execution of `payToVenue` so that only the customer (or an authorized executor) can call it. Note: restricting only to the customer may reduce flexibility (e.g., paying from another address), so the nonce + deadline approach is recommended.
   {% endhint %}

## Link to Proof of Concept

<https://gist.github.com/blackgrease/cb90d9d6706dfc83b27ed97507d87aaa>

## Proof of Concept

A runnable Foundry PoC is provided in the gist above. The PoC demonstrates Scenario 1.

Run with:

```bash
forge test --mt testVenuesCanStealAdditionalPaymentsFromCustomers -vvv --via-ir
```

Test Console logs (from the PoC):

```
[PASS] testVenuesCanStealAdditionalPaymentsFromCustomers() (gas: 1237678)
Logs:
  Venue starting USDC Balance:  895000000
  Customer starting balance (10k USDC):  10000000000
  Venue payment amount in USDC:  1000000000
  Customer USDC balance after payments:  9000000000
  Venue final balance (USDC):  10895000000
  Customer final balance (USDC):  0
  Additional Amount stolen (9k USDC):  9000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 15.46s (2.97s CPU time)
```

Explaining the test setup:

{% stepper %}
{% step %}
The PoC forks mainnet and uses on-chain Uniswap Router/Factory/Quoter addresses to create a USDC/LONG pool so venue deposits and payments function.
{% endstep %}

{% step %}
Deploys necessary contracts and creates the Uniswap Pool so the Venue Deposit and customer payment flows can be exercised.
{% endstep %}

{% step %}
Because the original LONG contract did not fit the flow in testing, a Mock LONG is used in the PoC (the issue demonstrated is unchanged).
{% endstep %}
{% endstepper %}

Due to multiple interactions, a stack trace was not provided in the report; the PoC and test logs above show the exploit behavior.

## Foundry Setup

{% stepper %}
{% step %}
Clone the repo:

```bash
git clone https://github.com/belongnet/checkin-contracts.git
```

{% endstep %}

{% step %}
Install Foundry dependencies:

```bash
forge install OpenZeppelin/openzeppelin-contracts-upgradeable@v5.4.0 --no-commit
forge install OpenZeppelin/openzeppelin-contracts@v5.4.0 --no-commit
npm install solady --force
```

{% endstep %}

{% step %}
Update remappings in `foundry.toml` (replace previous commented remappings with the following):

```toml
## Foundry Remappings
remappings = ["@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts","@openzeppelin/contracts-upgradeable/=lib/openzeppelin-contracts-upgradeable/contracts"]
```

{% endstep %}
{% endstepper %}
