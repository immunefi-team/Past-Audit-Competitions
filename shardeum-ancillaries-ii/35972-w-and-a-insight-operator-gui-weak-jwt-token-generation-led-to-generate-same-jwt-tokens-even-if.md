# #35972 \[W\&A-Insight] Operator-GUI Weak JWT Token Generation Led To Generate same JWT Tokens Even if

**Submitted on Oct 14th 2024 at 17:46:35 UTC by @Ouabala for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35972
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/validator-gui/tree/dev>
* **Impacts:**
  * Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.
  * Temporarily disabling user to access target site, such as: Locking up the victim from login, Cookie bombing, etc.
  * Improperly disclosing confidential user information, such as: Email address, Phone number, Physical address, etc.
  * Stealing sensitive data related to node operations
  * Disrupting the node's operations
  * Gain Access to Other Dashboards

## Description

\#Summary:

A critical vulnerability exists in the operator-gui login system of Shardeum, where the generated JWT tokens can be easily forged or guessed by attackers, potentially compromising node authentication and sensitive operations.

\#Description:

The vulnerability arises from how the JWT tokens are generated in the loginHandler function of the operator-gui:

\`\`\` if (cliResponse.login !== 'authorized') { unautorizedResponse(req, res); return; } const accessToken = jwt.sign({ nodeId: '' /\*\* add unique node id \*/ }, jwtSecret, { expiresIn: '8h' });

\`\`\`

The accessToken is signed using a secret (jwtSecret) and includes the nodeId. However, since the nodeId is not unique and the current logic does not properly include specific node information, an attacker can easily generate a valid JWT token by:

1- Guessing the nodeId: If the nodeId is left as an empty string or not sufficiently unique, attackers can guess it or forge tokens for other nodes. 2- Weak jwtSecret: If the jwtSecret is poorly generated or not stored securely, it becomes trivial for an attacker to exploit this to sign arbitrary tokens.

Link --> <https://github.com/shardeum/validator-gui/blob/dev/api/auth.ts#L39C4-L43C108>

Please refer to the attached image {JWT\_GENERATING\_TO\_ATO.png} to observe how the accessToken is generated once a user logs in with their password. The token is produced using vulnerable code, allowing me to generate other valid JWT tokens. These tokens will remain valid even if a user sets up the nodeId parameter, which can easily be predicted or may become exposed soon after testing. In any case, the use of nodeId alone is insufficient to secure the authentication process.

\#Steps to Reproduce:

```
1- Run the operator-gui with the existing login system.
2- Extract or guess the weak nodeId used in the JWT payload.
```

3- Forge a JWT token with a guessed or stolen jwtSecret. 4- Use the forged token to authenticate as a valid node without having proper credentials. Also note that the generated JWT can be used in all other API endpoints which can also be used to stake or unstake . So a malicious Attacker can generate the valid JWT to affect other nodes and even make other sensitive actions .

\#Proof of Concept (PoC):

The PoC demonstrates how an attacker could forge a JWT token:

```
Generate a token using the known nodeId and jwtSecret:
```

\`\`\` jwt.sign({ nodeId: 'guessed-node-id' }, 'exposed-jwt-secret', { expiresIn: '8h' })

\`\`\`

## Link to Proof of Concept

<https://gist.githubusercontent.com/ShellInjector/cd3aa9a8bc31a625ca73482c48d228fd/raw/1445cd9619e58fb195f9aa0447e4e71122808ce6/gistfile1.txt>

## Proof of Concept

## Proof of Concept

use this to generate new JWTs then go takeover other nodes .

jwt.sign({ nodeId: 'guessed-node-id' }, 'exposed-jwt-secret', { expiresIn: '8h' })
