# #36675 \[SC-Insight] Missing revoke instruction leads to Old delegate accounts have unlimited number

**Submitted on Nov 10th 2024 at 19:00:49 UTC by @shanb1605 for** [**Audit Comp | Jito Restaking**](https://immunefi.com/audit-competition/jito-restaking-audit-competition)

* **Report ID:** #36675
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/jito-foundation/restaking/tree/master/restaking\\_program>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

***Exact Affected Files:***

<https://github.com/jito-foundation/restaking/blob/master/restaking\\_program/src/ncn\\_delegate\\_token\\_account.rs>

<https://github.com/jito-foundation/restaking/blob/master/restaking\\_program/src/operator\\_delegate\\_token\\_account.rs>

<https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/delegate\\_token\\_account.rs>

## Brief/Intro

The Restaking program has delegating token process for both NCN and Operator. Also, Vault program has a process for token delegation. These three process delegates the token to delegate account, which is authorized by the delegate admin.

When the `approve` instruction from SPL token 2022 program is called inside these process, it approves unlimited token allowance to delegate account. This allowance stays, even delegate admin changes the delegate account.

Since, there is no specific instruction or inbuilt instruction to revoke the old approvals, the past delegates have still some allowance left and has a chance to steal the tokens from the owner by past malicious delegates.

## Vulnerability Details

Unlimited token approval take place in the three process. ***ncn\_delegate\_token\_account.rs*** <https://github.com/jito-foundation/restaking/blob/master/restaking\\_program/src/ncn\\_delegate\\_token\\_account.rs#L62-L69>

***Operator\_delegate\_token\_account.rs*** <https://github.com/jito-foundation/restaking/blob/master/restaking\\_program/src/operator\\_delegate\\_token\\_account.rs#L62-L69>

***delegate\_token\_account.rs (Vault program)*** <https://github.com/jito-foundation/restaking/blob/master/vault\\_program/src/delegate\\_token\\_account.rs#L65-L72>

The issue is not unlimited approvals, rather than missing instruction for revoking approvals inside those files or separate process function to revoke the approval from Old delegate account.

To describe the vulnerability, here is the necessary assumption:

1. The NCN/Operator/Vault delegate admin has approved Alice as token delegate account for some time.
2. On some occurrence, the delegate admin wishes to change the delegate account.
3. Now, the NCN/Operator/Vault delegate admin has approved Bob as token delegate account.
4. Alice notices, the initial call for allowance is set to U64 Max allowance, and there is still allowance amount is left.
5. Alice, the Old delegate turns out be malicious and spends all remaining token allowance.

## Impact Details

The impact labelled as ***direct theft of user funds***, as the report clearly states that past delegates have still some allowance amount left, and they are able to spend/steal the amount.

I searched for any revoke instruction written as separate process, it is not viable on in-scope repo. Since, there is missing revoke instruction, the allowance for old delegates stays forever on-chain leading to possible unauthorized spend.

## References

The Solana staking documentation mentions deactivating the stake delegations must be one of the operations by the authority. <https://solana.com/docs/economics/staking/stake-accounts#understanding-account-authorities>

## Recommendation(s)

Consider adding revoke functionality from SPL token 2022 program: <https://docs.rs/spl-token-2022/latest/spl\\_token\\_2022/instruction/fn.revoke.html>

It could be added in affected files or kept as separate process function.

## Link to Proof of Concept

<https://gist.github.com/shanb1605/f11146e17f72880e0a5e30b53549b4ec>

## Proof of Concept

## Proof of Concept

POC demonstrates the assumptions stated in ***Vulnerability Description***

Make sure everything is set up through cloning the repo: <https://github.com/jito-foundation/restaking/tree/master>

There is no specific instruction for viewing allowance, so the code logs the allowance amount from state variable after executing each process for Alice and Bob.

Add this snippet to /integration\_tests/tests/restaking/ncn\_delegate\_token\_account.rs <https://gist.github.com/shanb1605/f11146e17f72880e0a5e30b53549b4ec#file-ncn\\_delegate\\_token\\_account-rs>

Add this snippet to /integration\_tests/tests/restaking/Operator\_delegate\_token\_account.rs <https://gist.github.com/shanb1605/f11146e17f72880e0a5e30b53549b4ec#file-operator\\_delegate\\_token\\_account-rs>

Add this snippet to /integration\_tests/tests/vault/delegate\_token\_account.rs <https://gist.github.com/shanb1605/f11146e17f72880e0a5e30b53549b4ec#file-delegate\\_token\\_account-rs>

Each step possess different commands to run, ***follow the snippet's comment on Line no: 1*** to run the POC
