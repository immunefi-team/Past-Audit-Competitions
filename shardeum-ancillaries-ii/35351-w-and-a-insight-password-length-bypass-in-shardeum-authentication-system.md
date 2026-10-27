# #35351 \[W\&A-Insight] Password Length Bypass in Shardeum Authentication System

**Submitted on Sep 18th 2024 at 01:58:15 UTC by @Ouabala for** [**Audit Comp | Shardeum: Ancillaries II**](https://immunefi.com/audit-competition/shardeum-ancillaries-ii-boost)

* **Report ID:** #35351
* **Report Type:** Websites and Applications
* **Report severity:** Insight
* **Target:** <https://github.com/shardeum/validator-gui/tree/dev>
* **Impacts:**
  * Temporarily disabling user to access target site, such as: Locking up the victim from login, Cookie bombing, etc.
  * Changing non-sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction: Changing the first/last name of user, Enabling/disabling notifications
  * Changing sensitive details of other users (including modifying browser local storage) without already-connected wallet interaction and with up to one click of user interaction, such as: Email Password of the victim etc.

## Description

\#Description:

I have identified a vulnerability in the Shardeum authentication system, specifically related to the password length validation mechanism. According to the standard security policy, the password should be at least 8 characters long. However, I was able to bypass this restriction and successfully authenticate with a password of only 1 character, thereby compromising the system’s password strength requirements.

\#Steps to Reproduce:

```
I intercepted the request made during the login process and altered the payload to include a password that does not meet the 8-character minimum.
Despite sending a password shorter than the required length, the system allowed me to successfully authenticate.
```

Here is an example of the request that was sent:

\`\`\` POST /auth/login HTTP/1.1 Host: localhost:8080 Cookie: csrfToken=640e6c11bec4d106e3c0226873477e039fd908f381d97e7190229771fc1fc112%7C7d4617692f564fa00024ecc0cd985ce2febb749a8543e4e2a6b9a31327ca0e5e User-Agent: Mozilla/5.0 (X11; Linux x86\_64; rv:128.0) Gecko/20100101 Firefox/128.0 Accept: */* Accept-Language: en-US,en;q=0.5 Accept-Encoding: gzip, deflate, br Referer: <https://localhost:8080/> Content-Type: application/json X-Csrf-Token: 640e6c11bec4d106e3c0226873477e039fd908f381d97e7190229771fc1fc112 Content-Length: 79 Origin: <https://localhost:8080> Sec-Fetch-Dest: empty Sec-Fetch-Mode: cors Sec-Fetch-Site: same-origin Priority: u=4 Te: trailers Connection: close

{"password":"ce5ca673d13b36118d54a7cf13aeb0ca012383bf771e713421b4d1fd841f539a"}

\`\`\`

The value ce5ca673d13b36118d54a7cf13aeb0ca012383bf771e713421b4d1fd841f539a is the SHA-256 hash of the password "toor", which is only 4 characters long. But you can also test one character .

Please note that i'm only using a random encrypting site to let me encrypt text to sha-256. and this is also a bug since you are only using a random hashing mechanism.

Despite the hash length, the original password itself does not meet the minimum length requirement. The system should ideally validate the length of the un-hashed password before proceeding with the authentication, but it seems that only the hash is being checked, which allowed me to bypass the 8-character minimum limit.

\#Impact:

```
This vulnerability weakens the security of the password authentication system by allowing users to set or validate passwords that are shorter than the intended minimum length.
Shorter passwords are more vulnerable to brute-force attacks and can be easily cracked, especially if simple or common passwords are used.
```

## Proof of Concept

## Proof of Concept
