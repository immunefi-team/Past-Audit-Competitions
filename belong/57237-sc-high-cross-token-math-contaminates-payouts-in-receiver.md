# 57237 sc high cross token math contaminates payouts in receiver&#x20;

**Submitted on Oct 24th 2025 at 16:14:48 UTC by @OxPrince for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57237
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/receiver/receiver.cairo>
* **Impacts:**
  * Theft of unclaimed royalties

## Description

### Brief/Intro

The `Receiver` contract is intended to split incoming ERC-20 payments among payees according to fixed shares, mirroring OpenZeppelin’s payment splitter pattern. However, the implementation stores cumulative payout state (`total_released` and `released`) *globally*, without differentiating between payment tokens. When more than one ERC-20 token flows through the contract, the payout math for a given token incorporates historical releases from other tokens. This breaks the invariant that each token should be settled independently and leads to systematic mis-accounting.

## Vulnerability Details

* `total_released` is a single `u256` shared across every token processed by the contract, and `released` is keyed only by payee (`src/receiver/receiver.cairo:30-33`).
* `_pending_payment` computes `total_received` as `token.balance_of(contract) + total_released`, then subtracts `released[to]` (`src/receiver/receiver.cairo:131-145`). Because both state variables are global, the computation for token *A* mixes in payouts that happened in token *B*.
* In the happy path where all payees fully settle each token before any other token arrives, the error cancels out. As soon as a token is partially settled (common when payees claim at different times) and a new token is processed, the math drifts: one payee is under-paid on the new token while another is over-paid, and future withdrawals partially “rebalance” using the wrong asset.
* If a payee requests a release for a token whose balance is zero while previous tokens have been settled for other payees, `_pending_payment` can produce a positive `to_release`. The subsequent ERC-20 `transfer` call then reverts because there is no balance in that token, breaking forward progress for honest users.

## Concrete Scenario

{% stepper %}
{% step %}

### Two payees, 60% / 40% shares — step 1

Token B: Payee #1 withdraws first, receiving 60 tokens; 40 tokens remain unclaimed.
{% endstep %}

{% step %}

### Step 2

Token A: Payee #1 calls `release` and receives only 36 tokens (should be 60). Payee #2 then receives 64 tokens (should be 40). Contract balances are now distorted across tokens.
{% endstep %}

{% step %}

### Step 3

When payee #2 later withdraws from token B, they receive only 16 tokens instead of 40; the missing 24 were “repaid” using token A. The net share across all tokens sums correctly (80 tokens each), but each token’s distribution is wrong, violating the functional requirement that payouts stay denominated in their original asset.
{% endstep %}
{% endstepper %}

## Impact Details

* Mispriced payouts: payees end up with the wrong token balances, undermining trust in the revenue split.

## References

* Target source: <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/receiver/receiver.cairo>

## Proof of Concept

test\_receiver.cairo

```rust
#[test]
fn test_cross_token_payout_contamination_misallocates() {
    let (_, _, receiver_address, token_a) = deploy_factory_nft_receiver_erc20(false, true);
    let erc20_class = declare("ERC20Mock").unwrap().contract_class();
    let (token_b, _) = erc20_class.deploy(@array![]).unwrap();

    let receiver = IReceiverDispatcher { contract_address: receiver_address };

    // Step 1: settle token B partially (only creator withdraws).
    IERC20MintableDispatcher { contract_address: token_b }.mint(receiver_address, 1000);
    receiver.release(token_b, constants::CREATOR());

    assert_eq!(receiver.released(constants::CREATOR()), 800);
    assert_eq!(receiver.released(constants::PLATFORM()), 0);
    assert_eq!(receiver.totalReleased(), 800);

    // Step 2: deposit token A and trigger withdrawals.
    IERC20MintableDispatcher { contract_address: token_a }.mint(receiver_address, 1000);
    receiver.release(token_a, constants::CREATOR());

    let creator_token_a = IERC20Dispatcher { contract_address: token_a }.balance_of(constants::CREATOR());
    assert_eq!(creator_token_a, 640); // Expected share would be 800 if accounting were per-token.
    assert_eq!(receiver.released(constants::CREATOR()), 1440);
    assert_eq!(receiver.totalReleased(), 1440);

    receiver.release(token_a, constants::PLATFORM());
    let platform_token_a = IERC20Dispatcher { contract_address: token_a }.balance_of(constants::PLATFORM());
    assert_eq!(platform_token_a, 360);
    assert_eq!(receiver.released(constants::PLATFORM()), 360);

    // The residual token B balance remains locked in the contract.
    let remaining_token_b = IERC20Dispatcher { contract_address: token_b }.balance_of(receiver_address);
    assert_eq!(remaining_token_b, 200);
}
```
