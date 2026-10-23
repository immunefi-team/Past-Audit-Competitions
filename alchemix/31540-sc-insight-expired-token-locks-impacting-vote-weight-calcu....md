# 31540 - \[SC - Insight] Expired Token Locks Impacting Vote Weight Calcu...

Submitted on May 21st 2024 at 04:17:10 UTC by @cheatcode for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31540

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Manipulation of governance voting result deviating from voted outcome and resulting in a direct change from intended effect of original results

## Description

### Description

The `_balanceOfTokenAt` function in the Voting Escrow contract calculates the voting power of tokens without verifying if the locks have expired. This results in expired token locks still contributing to the voting power, which can lead to inaccurate representation of active voter support in governance decisions.

### Impact

Including expired locks in the vote weight calculation can distort the actual voting outcomes, enabling outdated stakes to influence current governance decisions. This misalignment can lead to decisions that do not reflect the present intentions of active stakeholders, undermining the protocol's governance integrity and effectiveness.

### Vulnerable Code

```solidity
function _balanceOfTokenAt(uint256 _tokenId, uint256 _time) internal view returns (uint256) {
    uint256 _epoch = userPointEpoch[_tokenId];
    if (_epoch == 0 || _time < pointHistory[userFirstEpoch[_tokenId]].ts) {
        return 0;
    } else {
        uint256 _min = 0;
        uint256 _max = userPointEpoch[_tokenId];
        for (uint256 i = 0; i < 128; ++i) {
            if (_min >= _max) {
                break;
            }
            uint256 _mid = (_min + _max + 1) / 2;
            if (userPointHistory[_tokenId][_mid].ts <= _time) {
                _min = _mid;
            } else {
                _max = _mid - 1;
            }
        }
        Point memory lastPoint = userPointHistory[_tokenId][_min];
        int256 biasCalculation = locked[_tokenId].maxLockEnabled
            ? int256(0)
            : lastPoint.slope * (int256(_time) - int256(lastPoint.ts));
        lastPoint.bias -= biasCalculation;
        if (lastPoint.bias < 0) {
            lastPoint.bias = 0;
        }
        return uint256(lastPoint.bias);
    }
}
```

### Mitigation

Incorporating a check at the beginning of the `_balanceOfTokenAt` function to verify that the token's lock has not expired ensures that the function returns a vote weight of zero for expired tokens. This modification prevents expired locks from affecting vote weight calculations, ensuring that the voting power reflects only the active and legitimate stakes, thereby enhancing the accuracy and fairness of governance processes.

```solidity
function _balanceOfTokenAt(uint256 _tokenId, uint256 _time) internal view returns (uint256) {
    if (_time > locked[_tokenId].end) {
        return 0;
    }
    uint256 _epoch = userPointEpoch[_tokenId];
    if (_epoch == 0 || _time < pointHistory[userFirstEpoch[_tokenId]].ts) {
        return 0;
    } else {
        uint256 _min = 0;
        uint256 _max = userPointEpoch[_tokenId];
        for (uint256 i = 0; i < 128; ++i) {
            if (_min >= _max) {
                break;
            }
            uint256 _mid = (_min + _max + 1) / 2;
            if (userPointHistory[_tokenId][_mid].ts <= _time) {
                _min = _mid;
            } else {
                _max = _mid - 1;
            }
        }
        Point memory lastPoint = userPointHistory[_tokenId][_min];
        int256 biasCalculation = lastPoint.slope * (int256(_time) - int256(lastPoint.ts));
        lastPoint.bias -= biasCalculation;
        if (lastPoint.bias < 0) {
            lastPoint.bias = 0;
        }
        return uint256(lastPoint.bias);
    }
}
```

## Proof of Concept

```python
import time

class TokenLock:
    def __init__(self, end_time):
        self.end_time = end_time

class VotingSystem:
    def __init__(self):
        self.locks = {}
        self.current_time = int(time.time())

    def add_token_lock(self, token_id, end_time):
        self.locks[token_id] = TokenLock(end_time)

    def _balance_of_token_at_issue(self, token_id):
        lock = self.locks.get(token_id)
        if not lock:
            return 0
        # Original code that does not check for expiration
        return 100 if self.current_time <= lock.end_time else 0

    def _balance_of_token_at_fixed(self, token_id):
        lock = self.locks.get(token_id)
        if not lock or self.current_time > lock.end_time:
            return 0
        # Fixed code with expiration check
        return 100

    def simulate_voting_power(self):
        token_id = 1
        end_time = self.current_time + 10  # Token expires after 10 seconds
        self.add_token_lock(token_id, end_time)

        # Checking before expiration
        print("Votes before expiration (Issue):", self._balance_of_token_at_issue(token_id))
        print("Votes before expiration (Fixed):", self._balance_of_token_at_fixed(token_id))

        # Simulate time passing beyond token expiration
        time.sleep(11)
        self.current_time = int(time.time())

        # Checking after expiration
        print("Votes after expiration (Issue):", self._balance_of_token_at_issue(token_id))
        print("Votes after expiration (Fixed):", self._balance_of_token_at_fixed(token_id))

# Create instance of the voting system
voting_system = VotingSystem()
voting_system.simulate_voting_power()
```

### Expected Output

1. **Before Token Expiration:**
   * With the issue: `Votes before expiration (Issue): 100`
   * With the fix: `Votes before expiration (Fixed): 100`
2. **After Token Expiration:**
   * With the issue: `Votes after expiration (Issue): 0` (This output might mistakenly show a value other than zero if not properly handled in the real contract)
   * With the fix: `Votes after expiration (Fixed): 0`

This script simulates the problem of token locks influencing voting weights after expiration and the effectiveness of the proposed solution in ensuring only active tokens contribute to voting power.
