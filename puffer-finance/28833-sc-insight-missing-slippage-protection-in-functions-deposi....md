# 28833 - \[SC - Insight] Missing slippage protection in functions deposi...

Submitted on Feb 28th 2024 at 12:35:02 UTC by @MrPotatoMagic for [Boost | Puffer Finance](https://immunefi.com/bounty/pufferfinance-boost/)

Report ID: #28833

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x7276925e42f9c4054afa2fad80fa79520c453d6a>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The functions `depositWstETH()` and `depositStETH()` in PufferDepositor.sol and the functions `deposit()` and `mint()` in PufferVault.sol allow users to obtain pufETH tokens. But these functions do not allow the user to pass in a `minAmountOut` parameter to specify the minimum amount of shares they want to receive. Due to this, users will be prone to intentional MEV attacks and unintentional slippage from other users., which will cause them to receive less than expected shares.

**Note: This issue is different from that reported in the Blocksec report, which talks about slippage in the swap functions only (which are OOS).**

## References

Blocksec report (see details of 2.3.1 for proof) - <https://github.com/blocksecteam/audit-reports/blob/main/solidity/blocksec\\_puffer\\_v1.0-signed.pdf>

Additionally, the ERC4626 EIP mentions that: `If implementors intend to support EOA account access directly, they should consider adding an additional function call for deposit/mint/withdraw/redeem with the means to accommodate slippage loss or unexpected deposit/withdrawal limits, since they have no other means to revert the transaction if the exact output amount is not achieved.`

EIP4626 - <https://eips.ethereum.org/EIPS/eip-4626#security-considerations>

## Vulnerability Details

1. First, let's look at the functions `depositWstETH()` and `depositStETH()`. Both functions currently only take in WstETH or StETH and provide users with pufETH, without verifying if the return value of pufETH amount received is atleast of a certain minimum amount. This is due to a lack of `minAmountOut` parameter, which checks if the `pufETHAmount` returned is greater than equal to it. If true, we should finish execution but if not then the execution should be reverted.

```solidity
File: PufferDepositor.sol
158:     /**
159:      * @inheritdoc IPufferDepositor
160:      */
161:     function depositWstETH(Permit calldata permitData) external restricted returns (uint256 pufETHAmount) {
162:         try ERC20Permit(address(_WST_ETH)).permit({
163:             owner: msg.sender,
164:             spender: address(this),
165:             value: permitData.amount,
166:             deadline: permitData.deadline,
167:             v: permitData.v,
168:             s: permitData.s,
169:             r: permitData.r
170:         }) { } catch { }
171: 
172:         SafeERC20.safeTransferFrom(IERC20(address(_WST_ETH)), msg.sender, address(this), permitData.amount);
173:         uint256 stETHAmount = _WST_ETH.unwrap(permitData.amount);
174: 
175:         return PUFFER_VAULT.deposit(stETHAmount, msg.sender);
176:     }
177: 
178:     /**
179:      * @inheritdoc IPufferDepositor
180:      */
181:     
182:     function depositStETH(Permit calldata permitData) external restricted returns (uint256 pufETHAmount) {
183:         try ERC20Permit(address(_ST_ETH)).permit({
184:             owner: msg.sender, 
185:             spender: address(this),
186:             value: permitData.amount,
187:             deadline: permitData.deadline,
188:             v: permitData.v,
189:             s: permitData.s,
190:             r: permitData.r
191:         }) { } catch { }
192: 
193:         SafeERC20.safeTransferFrom(IERC20(address(_ST_ETH)), msg.sender, address(this), permitData.amount);
194:         return PUFFER_VAULT.deposit(permitData.amount, msg.sender);
195:     }
```

2. Second. let's take a look at the function `deposit()` and `mint()` in PufferVault.sol. These functions are publicly accessible and allow users (who have already approved the vault with StETH) to obtain pufETH tokens. The same issue mentioned above applies to these functions as well. In case of mint(), the call would revert since the slippage requires the user to approve more assets to the vault contract.

```solidity
File: PufferVault.sol
085:     /**
086:      * @inheritdoc ERC4626Upgradeable
087:      * @dev Restricted in this context is like `whenNotPaused` modifier from Pausable.sol
088:      */
089:     function deposit(
090:         uint256 assets,
091:         address receiver
092:     ) public virtual override restricted returns (uint256) {
093:         return super.deposit(assets, receiver);
094:     }
095: 
096:     /**
097:      * @inheritdoc ERC4626Upgradeable
098:      * @dev Restricted in this context is like `whenNotPaused` modifier from Pausable.sol
099:      */
100:     function mint(
101:         uint256 shares,
102:         address receiver
103:     ) public override restricted returns (uint256) {
104:         return super.mint(shares, receiver);
105:     }
```

## Impact Details

The issue has been marked as Medium-severity since:

1. When a user uses functions `depositWstETH()`, `depositStETH()` and `deposit`, they receive lesser shares than expected due to the slippage.
2. When a user uses function `mint()`, the slippage could cause the user to receive lesser shares or entirely revert due to enough assets not being approved.
3. This is mentioned as a security consideration in EIP4626.

## Proof of Concept

How to use this POC:

* Add the POC to PufferVaultMainnet.fork.t.sol
* At the top of the file, add { Test , console} to the namespace. The same is needed to be done in the TestHelper.sol file.
* Add the RPC url for mainnet in setUp() in TestHelper.sol
* Run the POC using `forge test --mt testMissingSlippageIssue -vvv`

```solidity
File: PufferVaultMainnet.fork.t.sol
087:     // Actors
088:     address public jack = makeAddr("jack");
089:     address public jill = makeAddr("jill");
090: 
091:     function testMissingSlippageIssue() public giveToken(BLAST_DEPOSIT, address(stETH), jack, 50 ether) giveToken(BLAST_DEPOSIT, address(stETH), jill, 50 ether) {
092: 
093:         vm.prank(jack);
094:         uint256 jackExpectedShares = pufferVault.previewDeposit(50 ether);
095:         console.log("Jack's expected shares:", jackExpectedShares);
096: 
097:         vm.prank(jill);
098:         stETH.approve(address(pufferVault), type(uint256).max);
099:         vm.prank(jill);
100:         uint256 jillReceivedShares = pufferVault.depositStETH(50 ether, jill);
101:         console.log("Jill's received shares:", jillReceivedShares);
102: 
103:         // Let's say the vault receives some tokens from claiming
104:         vm.deal(address(pufferVault), 50 ether);
105: 
106:         vm.prank(jack);
107:         stETH.approve(address(pufferVault), type(uint256).max);
108:         vm.prank(jack);
109:         uint256 jackReceivedShares = pufferVault.depositStETH(50 ether, jack);
110:         console.log("Jack's received shares:", jackReceivedShares);
111: 
112:         console.log("Jack's slippage:", jackExpectedShares - jackReceivedShares);
113:     }
```

```solidity
[PASS] testMissingSlippageIssue() (gas: 391707)
Logs:
  Jack's expected shares: 49905530654562557002
  Jill's received shares: 49905530654562557002
  Jack's received shares: 49898438881924232864
  Jack's slippage: 7091772638324138
```
