# #35697 \[BC-Insight] \[Informational] Code logic contains potential risk of full network shutdown

**Submitted on Oct 3rd 2024 at 19:27:36 UTC by @Merkle\_Bonsai for** [**Audit Comp | Shardeum: Core II**](https://immunefi.com/audit-competition/shardeum-core-ii-boost)

* **Report ID:** #35697
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/shardus-core/tree/dev>
* **Impacts:**
  * informational: potential risk

## Description

Immunefi guys, please, just pass it to the project. I'm OK if it will be lowered to informational with no reward. We already discussed it in #35415, and Mehdi personally told to report such things anyway for that project. <https://discord.com/channels/787092485969150012/1256211020482084987/1286726883530117230>

## Vulnerability Details

Object handlers contain prototype values that have various risks, including non-standard properties like \`**defineGetter**\` and \`**defineSetter**\`. Properties that are not validated explicitly should not be processed with array access.

For example, one of most important handlers of shardus-core is simply lucky to be written in specific way: \`\`\` const { route, payload } = data ... const handler = this.internalRoutes\[route] if (!payload) { await handler(null, respond, header, sign) return } await handler(payload, respond, header, sign) \`\`\`

If it would be written like this (which looks legitimate and totally same), it would be presenting the vector for total network shutdown: \`\`\` const { route, payload } = data ... await this.internalRoutes\[route]\(payload ?? null, respond, header, sign) if (!payload) { return } \`\`\`

However, that would allow attacker to pass following data: \`\`\` { route: "**defineGetter**", payload: "binary/gossip" } \`\`\`

that will be handled like this: \`\`\` await this.internalRoutes\["**defineGetter**"]\("binary/gossip", respond, header, sign) \`\`\`

causing the value to be overwritten

## Impact Details

Impact may vary, depending on location of that issue. It is better to either check if property is own (not from proto) or use ES6 Map structures for that cases.

## Proof of Concept

## Proof of Concept

\`\`\` routes = {gossip: () => 'hello'} // => {gossip: ƒ} routes.**defineGetter**('gossip', () => 'mocked') routes.gossip // => 'mocked' \`\`\`
