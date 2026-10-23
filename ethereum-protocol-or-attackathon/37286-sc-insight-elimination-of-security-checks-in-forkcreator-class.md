# #37286 \[SC-Insight] Elimination of Security Checks in ForkCreator Class

**Submitted on Dec 1st 2024 at 16:44:46 UTC by \[redacted] for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37286
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/ethereum/execution-specs>
* **Impacts:**
  * (Compiler) Elimination of security checks

## Description

\#Summary:

The ForkCreator class and its associated methods (find\_replace and duplicate\_fork) fail to validate user input (from\_fork and to\_fork) before constructing file paths or performing file operations. This lack of input validation allows attackers to bypass intended security controls, such as directory traversal prevention or file overwriting safeguards.

Attackers can exploit this vulnerability to:

1- Perform directory traversal attacks to access sensitive system files.\
2- Overwrite arbitrary files or directories, potentially causing data loss or privilege escalation.

Vuln Code Snippet:

<https://github.com/ethereum/execution-specs/blob/master/src/ethereum\\_spec\\_tools/new\\_fork.py#L162>

Proof of Concept (PoC):

The following PoC demonstrates how an attacker could exploit the vulnerability to copy sensitive system files from /etc to a new location /tmp/malicious\_fork, exposing critical information.

Link POC - <https://gist.githubusercontent.com/ShellInjector/f1869cd2456d01285a02828b5aec582b/raw/db5381a2946caa6e7ca15053ed6f263ff8ae589f/POC>

\#Usage:

`python poc.py --from_fork="../../../etc" --to_fork="../../../../tmp/malicious_fork"`

\#Output of the POC :

If the exploit succeeds, it will copy the contents of /etc into /tmp/malicious\_fork. The output will look like this:

\[INFO] Copying from src/ethereum/../../../etc to src/ethereum/../../../../tmp/some\_fork\
\[INFO] Successfully copied to src/ethereum/../../../../tmp/some\_fork

\#Impact:

```
Data Exposure: Allows attackers to access sensitive files (..).
Data Loss: Enables overwriting of existing directories or files without validation.
Privilege Escalation: Overwriting critical system files could lead to a compromise of system integrity.
```

## Link to Proof of Concept

<https://gist.link>

## Proof of Concept

## Proof of Concept

<https://gist.githubusercontent.com/ShellInjector/f1869cd2456d01285a02828b5aec582b/raw/db5381a2946caa6e7ca15053ed6f263ff8ae589f/POC>
