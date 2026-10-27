# Boost \_ Shardeum\_ Ancillaries 33809 - \[Websites and Applications - Insight] Blocking the user from i

Submitted on Mon Jul 29 2024 18:41:54 GMT-0400 (Atlantic Standard Time) by @anton\_quantish for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #33809

Report type: Websites and Applications

Report severity: Insight

Target: <https://github.com/shardeum/validator-gui/tree/dev>

Impacts:

* Temporarily disabling user to access target site, such as: Locking up the victim from login, Cookie bombing, etc.

## Description

Hey,

THIS IS THE RESUBMISSION

The original issue was incorrectly close by the triager. Please note that the CORS has not to be set because the PoC uses `no-cors` mode.

## Brief/Intro

Attacker can temporarily block a victim from interacting with GUI via rate-limiting abuse.

## Vulnerability Details

Since an exceeding of the Rate Limiting blocks any requests from the affected IP address to the Validator GUI API, an attacker can easily abuse this and temporary block user from interaction with the app.

Attacker can craft the malicious page sending at least 1500 request to the Validator API and lure a victim into visiting it. After just a few seconds, the victim's IP will be blacklisted and thus he won't be able to interact with the Validator GUI in the next 10 minutes.

There's a chance that this attack could also be persisted with the browser service workers or something, but I didn't research this deeper.

## Impact Details

After the successful attack, the user is temporary not able to interact with the Validator GUI at all.

## Mitigation

1. I think you can disable the rate-limiting at all for the signed-in users (having the valid token set in cookies)
2. Also, you can implement the IP whitelist feature to be sure that the Validator admin will never be blocked by rate-limiting.

## Proof of concept

## Proof of Concept

1. Place the following HTML file somewhere (replacing the API URL with your Validator GUI one):

```html
<html>
	<script>
		async function exploit() {
			for (var i = 0; i < 1550; i++) {
				fetch('http://172.16.205.128:8081/auth/login', {mode: "no-cors"})
			}
		}
		exploit()
	</script>
</html>
```

2. Visit this page and wait for a few (\~10-15) seconds (you can look up the console logs and if there're 429 errors there then it's done)
3. Visit your Validator GUI and make sure you are temporary blocked from interacting with it.
