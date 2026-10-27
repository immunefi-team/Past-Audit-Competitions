# Attackathon \_ Fuel Network 32459 - \[Websites and Applications - Low] URGENT WEB funds drained using

## URGENT: WEB3 funds drained using URL path based manipulation and injection, an attacker can spoof domains on any important web3 dapp API call as legitimate domains.

Submitted on Sat Jun 22 2024 12:25:40 GMT-0400 (Atlantic Standard Time) by @UGWST\_COM for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32459

Report type: Websites and Applications

Report severity: Low

Target: <https://github.com/fuellabs/fuels-wallet/tree/4a20b3d8684a07e40bd2f7559032941b5adbe022>

Impacts:

* Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters, Substituting contract addresses, Submitting malicious transactions
* Injecting/modifying the static content on the target application without JavaScript (persistent), such as: HTML injection without JavaScript, Replacing existing text with arbitrary text, Arbitrary file uploads, etc
* Injection of malicious HTML or XSS through metadata
* Hijack ANY web3 wallet API calls from legitimate trusted dapps

### Description

## Introduction and Executive Summary

During a security audit of `Fuel Wallet`, we were able to find critical vulnerability that leads DIRECTLY to web3 `Fuel Wallet` assets impacting their user base. In this report we will highlight all the security holes we have found, and explain how threat actors would leverage these vulnerabilities for malicious purposes, further proving the impact of the findings.

The recent months have been quite chaotic for Crypto / DEFI / DAO / NFT / projects. We have observed several unique compromises and exploits which caused considerable financial loss to the affected crypto users and projects. Not only smart contract exploits, but also WEB2 vulnerabilities are being actively used to efficiently drain millions from organizations and user funds. A few notable recent examples include the Badger DAO incident(1), where similarly to the vulnerability in this report a Cross Site Scripting injection in a trusted page lead to an over $120,000,000 financial loss. Another incident worth mentioning is EtherDelta incident(2), where a stored Cross Site Scripting injection in token names directly lead to their users being compromised and their funds stolen. We can also see similar TTPs being used against the BitPay's users (3), where the threat actor backdoored a JavaScript library in order to inject malicious JavaScript and exploit BitPay's applications. As we can see, these Web2 to Web3 attacks are no longer theoretical, but extremely practical and actively being exploited by threat actors. One of the root issues is that no matter how secure smart contracts and blockchains may be, web applications are still wildly hard to secure and defend. Additionally, web2 vulnerabilities such as Cross Site Scripting / HTML, and other injections are much easier to exploit and find, which increases the likelihood of it being exploited by threat actors.

The vulnerability discussed and shown in this report are a prime example of that. An attacker can leverage trusted domains like opensea.io / reddit.com or any else, where they allow embedded content from non-trusted different origins. Fuel Wallet would display spoofed trusted domain origins to an unsuspecting user.

We have been awarded critical bounty of $120,000 from Metamask security team for similar finds:

<https://medium.com/metamask/metamask-awards-bug-bounty-for-clickjacking-vulnerability-9f53618e3c3a>

<https://github.com/brave/brave-browser/issues/21904>

And from others wallets like Brave, Phantom the list goes on.

Our research team was able to find multiple vulnerabilities across crypto projects, where an attack chain to spoof trusted domains in real-world would be devastating.

It is worth to mention that Brave Crypto Wallet, Glow\.app Crypto Wallet and Metamask Crypto Wallet were also vulnerable to same spoof origin attack as `Fuel wallet`, as metioned, we have gotten critical bounties from these crypto wallets.

## Vulnerability Technical Details:

The discovered vulnerability is in how `Fuel Wallet` handles and displays the origin from where the web3 api call is executed; instead of displaying the correct true origin, the attacker can spoof the origin domain to any he wants.

Many websites like `reddit.com` or `opensea.io`, are using intended feature to allow embedded RAW HTML from providers like `embed.ly` or `testing-playground.com` etc. we were able to spoof the origin so the executed web3 call looks like comming from `reddit.com`, `opensea.io`, `looksrare` or any else!

## Vulnerability Technical Details:

The discovered vulnerability is in how `Fuel Wallet` handles and displays the origin from where the web3 api call is executed; instead of displaying the correct true origin, the attacker can spoof the origin domain to any he wants.

Many websites like `reddit.com` or `opensea.io`, are using intended feature to allow embedded RAW HTML from providers like `embed.ly` or `testing-playground.com` etc.

## Exploitation and Proof Of Concept:

`Fuel Wallet` displays the domain origin from where transaction is executed in it's wallet UI pop ups - it displays the most important domain name for users to see where they are authorizing transaction from.

The vulnerability here is that a subdomain can be up to 255 characters long, but if you have multiple levels in your subdomain, each level can be 63 characters long, `Fuel Wallet` does not display the full origin to the user, which allows FULL manipulation of shown origin in the displayed wallet UI.

We have carefully prepared Proof Of Concept for the team to see and visit how the `Fuel Wallet` handles these crafted malicious origins and that the attacker can poison/spoof them using URL based manipulation/injections - there are endless possibilities to spoof the domain name:

(when visiting make sure to have your `Fuel Wallet` fully setup and ready to be used.) - (**In the video and PoC connecting FUEL wallet, and executing the WEB3 API transaction is done manually, but via simple js can be triggered automatically**)

`https://fuel.network________________________________fuel.network.__________________________________________________.xdv.cz.srv21.endora.cz/metapoc/block.php`

`https://opensea.io________________________________opensea.io.__________________________________________________.xdv.cz.srv21.endora.cz/metapoc/block.php`

`https://looksrare.org________________________________looksrare.org.__________________________________________________.xdv.cz.srv21.endora.cz/metapoc/block.php`

(The position of the spoofed domain is up to the attacker; it can be at the beginning of URL as well.)

WATCH: VIDEO SPOOF POC

<https://ugwst.com/pocs/fuell.mp4>

(**In the video and PoC connecting FUEL wallet, and executing the WEB3 API transaction is done manually, but via simple js can be triggered automatically**)

Furthermore the attacker can aswell craft clever pop-up exploit which opens small window 1px by 1px and instantly executes the spoofed web3 calls, then the attacker can include javascript to close the window instantly.

This exploit was possible to be used on many popular platforms which allows web3 wallet interactions via their playgrounds to display NFT's via SVG.

Code example:

```
<a href="#"  onclick="window.open('https://opensea.io________________________________opensea.io.__________________________________________________.xdv.cz.srv21.endora.cz/metapoc/block.php','mywindow','menubar=1,resizable=1,width=1,height=1'); return false;">
    Authorize transaction
</a>

```

The victim clicks authorize transaction, from fake website however the wallet pop up matches the legitimate top-level domain.

The trick here is that each character in the URL bar has slightly different length / width; it allows an attacker to craft URL which will perfectly fit for spoofed shown origin.

As described previously the attacker would optimally want to use this vulnerability from embeds or using pop-ups - like `reddit.com` or `opensea.io`, `looksrare.org` since the spoofed domain would match the top-level origin the victim is on, there is zero trust issues to authorize the malicious spoofed origin and quickly execute two calls - connect wallet then sign which would lead to complete compromise of crypto wallet.

## Recommendations:

As shown in examples and real-world reports, the ability to spoof origins needs to be fixed. Users rely on the `Fuel Wallet` to show them the true origin of web3 api execution for critical functions like signatures etc.

The domain name should be fully shown in the `Fuel Wallet` UI interface, its even spoofed in "connected sites UI".

Brave and Metamask deployed a fix in beta-version (nightly) which fits the entire origin inside the shown prompt in the UI.

## Impact:

Complete overtake of `Fuel Wallet` user wallet which leads to full crypto and nft drained. were able to spoof the origin so the executed web3 call looks like comming from `reddit.com`, `opensea.io` or any else!

### Proof of concept

## Exploitation and Proof Of Concept:

`Fuel Wallet` displays the domain origin from where transaction is executed in it's wallet UI pop ups - it displays the most important domain name for users to see where they are authorizing transaction from.

The vulnerability here is that a subdomain can be up to 255 characters long, but if you have multiple levels in your subdomain, each level can be 63 characters long, `Fuel Wallet` does not display the full origin to the user, which allows FULL manipulation of shown origin in the displayed wallet UI.

We have carefully prepared Proof Of Concept for the team to see and visit how the `Fuel Wallet` handles these crafted malicious origins and that the attacker can poison/spoof them using URL based manipulation/injections - there are endless possibilities to spoof the domain name:

(when visiting make sure to have your `Fuel Wallet` fully setup and ready to be used.) - (**In the video and PoC connecting FUEL wallet, and executing the WEB3 API transaction is done manually, but via simple js can be triggered automatically**)

`https://fuel.network________________________________fuel.network.__________________________________________________.xdv.cz.srv21.endora.cz/metapoc/block.php`

`https://opensea.io________________________________opensea.io.__________________________________________________.xdv.cz.srv21.endora.cz/metapoc/block.php`

`https://looksrare.org________________________________looksrare.org.__________________________________________________.xdv.cz.srv21.endora.cz/metapoc/block.php`

(The position of the spoofed domain is up to the attacker; it can be at the beginning of URL as well.)

WATCH: VIDEO SPOOF POC

<https://ugwst.com/pocs/fuell.mp4>

(**In the video and PoC connecting FUEL wallet, and executing the WEB3 API transaction is done manually, but via simple js can be triggered automatically**)

Furthermore the attacker can aswell craft clever pop-up exploit which opens small window 1px by 1px and instantly executes the spoofed web3 calls, then the attacker can include javascript to close the window instantly.

This exploit was possible to be used on many popular platforms which allows web3 wallet interactions via their playgrounds to display NFT's via SVG.

Code example:

```
<a href="#"  onclick="window.open('https://opensea.io________________________________opensea.io.__________________________________________________.xdv.cz.srv21.endora.cz/metapoc/block.php','mywindow','menubar=1,resizable=1,width=1,height=1'); return false;">
    Authorize transaction
</a>

```

The victim clicks authorize transaction, from fake website however the wallet pop up matches the legitimate top-level domain.

The trick here is that each character in the URL bar has slightly different length / width; it allows an attacker to craft URL which will perfectly fit for spoofed shown origin.

As described previously the attacker would optimally want to use this vulnerability from embeds or using pop-ups - like `reddit.com` or `opensea.io`, `looksrare.org` since the spoofed domain would match the top-level origin the victim is on, there is zero trust issues to authorize the malicious spoofed origin and quickly execute two calls - connect wallet then sign which would lead to complete compromise of crypto wallet.

## Recommendations:

As shown in examples and real-world reports, the ability to spoof origins needs to be fixed. Users rely on the `Fuel Wallet` to show them the true origin of web3 api execution for critical functions like signatures etc.

The domain name should be fully shown in the `Fuel Wallet` UI interface, its even spoofed in "connected sites UI".

Brave and Metamask deployed a fix in beta-version (nightly) which fits the entire origin inside the shown prompt in the UI.
