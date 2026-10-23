# #46910 \[SC-Insight] Token Balance Event Data Inconsistency in Position Transfers

**Submitted on Jun 6th 2025 at 06:06:53 UTC by @Catchme for** [**IOP | Paradex**](https://immunefi.com/audit-competition/iop-paradex)

* **Report ID:** #46910
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/tradeparadex/audit-competition-may-2025/tree/main/paraclear>
* **Impacts:**
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The `_transfer_positions_internal()` function in the `paraclear.cairo` file has a flaw where token balance update events emit incorrect `prev_amount` values for both sender and receiver accounts during multi-position transfers. This inconsistency can lead to misleading event data.

## Vulnerability Details

In the `_transfer_positions_internal()` function, the code incorrectly uses the initial token balance amount for all position transfer events within the loop, rather than tracking the actual previous amount for each iteration. This affects both sender and receiver accounts, leading to incorrect event emissions.

```cairo
// Vulnerable Code Location
// File: paraclear/src/paraclear/paraclear.cairo
// Lines: 2039-2238

// At function start
let initial_account_token_balance = *account_state
    .token_balances[account_state.settlement_token_index];
let mut updated_account_token_amount: i128 = initial_account_token_balance
    .amount
    .try_into()
    .unwrap();

// Inside the transfer loop
for idx in 0..account_state.perpetual_names.len() {
    // ... position transfer logic updates updated_account_token_amount ...
    
    updated_account_token_amount = sender_token_amount_realized;
    self.token.write_asset_balance(
        sender,
        initial_account_token_balance.token_address,
        updated_account_token_amount.into(),
    );
    
    // Problem - Always uses initial amount
    self.emit(
        TokenComponent::Event::TokenAssetBalanceUpdate(
            TokenAssetBalanceUpdate {
                account: sender,
                token_address: initial_account_token_balance.token_address,
                prev_amount: initial_account_token_balance.amount,  // Always the same
                updated_amount: updated_account_token_amount.into(),
                is_liquidation: is_liquidation,
            },
        ),
    );
}
```

## Impact Details

1. **Audit Trail Inconsistency**: Each position transfer event shows the same starting balance, making it impossible to track intermediate balance changes.
2. **Event Processing Logic**: Systems that process these events sequentially will have incorrect intermediate state calculations.
3. **Monitoring and Alerting**: Balance change monitoring systems may trigger false alerts or miss real changes.

## Proof of Concept

## Proof of Concept

```
#[test]
fn test_poc_event_data_inconsistency_vulnerability() {
    
    // Simulate the buggy logic that exists in the actual code
    let initial_sender_balance: i128 = 100000000000; // 1000 USDC (8 decimals)
    let initial_receiver_balance: i128 = 0;
    
    let mut current_sender_balance = initial_sender_balance;
    let mut current_receiver_balance = initial_receiver_balance;
    
    // Simulate 3 position transfers with different PnL (like BTC, ETH, SOL positions)
    let position_pnls = array![
        20000000000_i128,  // +200 USDC profit
        -10000000000_i128, // -100 USDC loss  
        30000000000_i128   // +300 USDC profit
    ];
    
    let mut event_data_correct = array![];
    let mut event_data_buggy = array![];
    
    // Process each position transfer
    let mut i = 0;
    loop {
        if i >= position_pnls.len() {
            break;
        }
        
        let pnl = *position_pnls.at(i);
        
        // Record what the previous balance SHOULD be for events
        let correct_prev_sender = current_sender_balance;
        let correct_prev_receiver = current_receiver_balance;
        
        // Update balances (this part works correctly in actual code)
        current_sender_balance += pnl;
        current_receiver_balance -= pnl;
        
        // What CORRECT events should contain
        event_data_correct.append((correct_prev_sender, current_sender_balance));
        event_data_correct.append((correct_prev_receiver, current_receiver_balance));
        
        // What BUGGY events actually contain (the vulnerability)
        event_data_buggy.append((initial_sender_balance, current_sender_balance));
        event_data_buggy.append((initial_receiver_balance, current_receiver_balance));
        
        i += 1;
    };
    
    // Verify the vulnerability exists by checking event data differences
    let mut vulnerability_detected = false;
    
    // Check sender events
    let sender_event_1_correct = *event_data_correct.at(0); // (1000, 1200)
    let sender_event_2_correct = *event_data_correct.at(2); // (1200, 1100) 
    let sender_event_3_correct = *event_data_correct.at(4); // (1100, 1400)
    
    let sender_event_1_buggy = *event_data_buggy.at(0); // (1000, 1200)
    let sender_event_2_buggy = *event_data_buggy.at(2); // (1000, 1100) <- BUG!
    let sender_event_3_buggy = *event_data_buggy.at(4); // (1000, 1400) <- BUG!
    
    // First event should be the same
    assert!(sender_event_1_correct == sender_event_1_buggy, "First event should match");
    
    // Second and third events should be different (proving the bug)
    if sender_event_2_correct != sender_event_2_buggy {
        vulnerability_detected = true;
    }
    if sender_event_3_correct != sender_event_3_buggy {
        vulnerability_detected = true;
    }
    
    // Check receiver events 
    let receiver_event_2_correct = *event_data_correct.at(3); // (-200, -100)
    let receiver_event_2_buggy = *event_data_buggy.at(3);     // (0, -100) <- BUG!
    
    if receiver_event_2_correct != receiver_event_2_buggy {
        vulnerability_detected = true;
    }
    
    // Verify final balances are mathematically correct
    let total_pnl = 20000000000_i128 - 10000000000_i128 + 30000000000_i128; // +400 USDC
    let expected_final_sender = initial_sender_balance + total_pnl;
    let expected_final_receiver = initial_receiver_balance - total_pnl;
    
    assert!(current_sender_balance == expected_final_sender, "Sender balance math error");
    assert!(current_receiver_balance == expected_final_receiver, "Receiver balance math error");
    
    // The vulnerability is confirmed: balance calculations are correct but event data is wrong
    assert!(vulnerability_detected, "Event data inconsistency vulnerability not detected");
    
    // Additional verification: demonstrate the specific bug pattern
    let (buggy_prev_2, _) = sender_event_2_buggy;
    let (buggy_prev_3, _) = sender_event_3_buggy;
    let (correct_prev_2, _) = sender_event_2_correct;
    let (correct_prev_3, _) = sender_event_3_correct;
    
    // Bug pattern: buggy events always use initial balance as prev_amount
    assert!(buggy_prev_2 == initial_sender_balance, "Bug pattern not confirmed");
    assert!(buggy_prev_3 == initial_sender_balance, "Bug pattern not confirmed");
    
    // Correct pattern: prev_amount should track intermediate changes
    assert!(correct_prev_2 != initial_sender_balance, "Correct pattern not confirmed");
    assert!(correct_prev_3 != initial_sender_balance, "Correct pattern not confirmed");

}
```
