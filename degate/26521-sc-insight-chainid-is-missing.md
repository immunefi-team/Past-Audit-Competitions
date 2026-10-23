# 26521 - \[SC - Insight] ChainId is missing

Submitted on Dec 4th 2023 at 12:44:43 UTC by @sachinZmishra for [Boost | DeGate](https://immunefi.com/bounty/boosteddegatebugbounty/)

Report ID: #26521

Report type: Smart Contract

Report severity: Insight

Target: <https://etherscan.io/address/0x54D7aE423Edb07282645e740C046B9373970a168#code>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

## Description

In FastWithdrawalLiquidityProvider.sol contract, Domain\_Separator is initialized in the constructor in which chainId parameter is absent.

DOMAIN\_SEPARATOR = EIP712.hash(EIP712.Domain("FastWithdrawalLiquidityProvider", "1.0", address(this)));

The chainId is often included in the EIP-712 domain separator to prevent replay attacks across different Ethereum networks.

The absence of chainId in the DOMAIN\_SEPARATOR construction can potentially lead to a vulnerability in certain scenarios. Including chainId in the DOMAIN\_SEPARATOR is a security best practice to prevent replay attacks across different Ethereum networks.

Without the chainId in the DOMAIN\_SEPARATOR, an attacker could potentially craft a valid signature on one Ethereum network (e.g., the mainnet) and replay it on another Ethereum network (e.g., a testnet or a different mainnet). Including chainId in the domain separator ensures that the signature is specific to a particular Ethereum network, mitigating this type of cross-chain replay attack.

To address this potential vulnerability, you should update the DOMAIN\_SEPARATOR construction to include chainId.

DOMAIN\_SEPARATOR = EIP712.hash( EIP712.Domain( "FastWithdrawalLiquidityProvider", "1.0", address(this), chainId // Include the chainId in the domain separator ) );

## Proof of concept

Scenario:

Alice is a liquidity provider who has funds stored in the FastWithdrawalLiquidityProvider contract. Bob is an attacker who wants to exploit the absence of chainId in the contract.

Normal Interaction:

Alice regularly interacts with the contract to perform fast withdrawals by submitting valid approvals. These approvals are signed using her private key and include the current chainId in the DOMAIN\_SEPARATOR.

Attack Steps by Bob:

Bob observes Alice's valid transactions on the Ethereum mainnet. He notices that the FastWithdrawalLiquidityProvider contract uses EIP-712 signatures but does not include chainId in the DOMAIN\_SEPARATOR.

Crafting the Attack:

Bob crafts a fake approval mimicking Alice's valid approvals but with a different chainId. He crafts this on a testnet or a different Ethereum network where the contract may have a different chainId. The fake approval includes Alice's account, addresses, and other details but is signed with Bob's private key.

Executing the Attack:

Bob deploys a malicious contract that calls the execute function of the FastWithdrawalLiquidityProvider contract with his crafted fake approval.

Exploitation:

Since the FastWithdrawalLiquidityProvider contract doesn't include chainId in the DOMAIN\_SEPARATOR, it treats the fake approval as if it were signed on the mainnet. The contract executes the fast withdrawal based on the fake approval, allowing Bob to drain funds from the contract.

Outcome:

Bob successfully exploits the vulnerability, causing unintended withdrawals from the contract. Alice becomes a victim as her funds are drained without her consent.
