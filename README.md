# NAT
Basic NAT config with FreeBSD




## Test computer (Border NAT)
```
$ dmesg | less: 
FreeBSD 8.0-STABLE-201004 #0: Mon Apr 5 15:59:06 UTC 2010 
CPU: Intel(R) Pentium(R) 4 CPU 3.20GHz (3200.01-MHz K8-class CPU) 
real memory = 536870912 (512 MB) 
age0: mem 0xfeac0000-0xfeafffff irq 17 at device 0.0 on pci2 
rl0: port 0xe800-0xe8ff mem 0xfebffc00-0xfebffcff irq 19 at device 0.0 on pci4 
```
## Router 
Up to date with the latest security advisories
```
# freebsd-update fetch
# freebsd-update install
```
Update packages
```
# pkg upgrade
```
Install dependencies
```
# pkg install iperf netstat
```

## Scheme
`laptop(192.168.0.188)`-->`age0(192.168.0.1)`-->`rl0(192.168.133.142)`-->`internet`

IP Pool for nat is `192.168.133.0/24`
DNS Server: `192.168.133.1`
`age0` - internal interface 
`rl0` - external interface 

## Config NAT
### Enable gateway function
To enable gateway function on FreeBSD, add the following line to `/etc/rc.conf`:
```
gateway_enable="YES"
```
This line enables IP forwarding (i.e., sysctl net.inet.ip.forwarding=1).

### Enable NAT with  pf
The NAT function is in pf (packet filter). To use pf, add the following lines to `/etc/rc.conf`:
```
pf_enable="YES"
pf_rules="/etc/pf.rules"
pflog_enable="YES"
pflog_logfile="/var/log/pflog"
```
The first line enables pf on boot, and the second one specifies the configuration of rules for pf. The third line enables logging and the last one specifies the path to log file.

Then create  `/etc/pf.rules`  as follows.
```
ls -l /etc/pf.rules
-rw-r--r--  1 root  wheel     0 Mar  3 18:30 /etc/pf.rules
```
The content of the file is like following:
```
### Options ###
set limit states 100000

### Macros ###
ext_if = "em0"               # External network interface for IPv4
ext_if6 = "em0"              # External network interface for IPv6
ext_addr = "192.0.2.1"       # External IPv4 address (i.e., global)
int_if = "em1"               # Internal network interface for IPv4
int_if6 = "em1"              # Internal network interface for IPv6
int_addr = "10.0.0.1"        # Internal IPv4 address (i.e., gateway for private network)
int_network = "10.0.0.0/24"  # Internal IPv4 network

### Tables ###
# Host local address
table <local> const { 127.0.0.1 }
# IPv4 private address ranges
table <private> const { 10/8, 172.16/12, 192.168/16 }
# Special-use IPv4 addresses defined in RFC3330
table <special> const { 0/8, 14/8, 24/8, 39/8, 127/8, 128.0/16, 169.254/16, 192.0.0/24, 192.0.2/24, 192.88.99/24, 198.18/15, 240/4 }

### Scrub: Packet normalization ###
# Scrub for all incoming packets
scrub in all
# Randomize the ID field for all outgoing packets
scrub out all random-id
# If you have MTU problem or something like that
#scrub out all random-id  max-mss 1400

### NAT ###
nat on $ext_if from $int_network to ! <private> -> $ext_addr

### Filters ###
# Permit keep-state packets for UDP and TCP on external interfaces
pass out quick on $ext_if proto udp all keep state
pass out quick on $ext_if6 proto udp all keep state
pass out quick on $ext_if proto tcp all modulate state flags S/SA
pass out quick on $ext_if6 proto tcp all modulate state flags S/SA

# Permit any packets from internal network to this host
pass in quick on $int_if inet from $int_network to $int_addr

# Permit established sessions from internal network to any (incl. the Internet)
pass in quick on $int_if inet from $int_network to any keep state
# If you want to limit the number of sessions per NAT, nodes per NAT (simultaneously), and sessions per source IP
# Please refer to <http://www.openbsd.org/faq/pf/filter.html> for greater detailed information
#pass in quick on $int_if inet from $int_network to any keep state (max 30000, source-track rule, max-src-nodes 100, max-src-states 500 )

# Permit and log all packets from clients in private network through NAT
pass in quick log on $int_if all

# Pass any other packets
pass in all
pass out all
```
Note that this file also defines filter rules which is useful or mandatory to provide the network to users. If you do not provide this network to others, you can eliminate filter section just by writing following:
```
pass in all
pass out all
```

Here, we note that file /var/log/pflog would be automatically created as follows.
```
# ls -l /var/log/pflog
-rw-------  1 root  wheel  3832 Jul 13 17:45 /var/log/pflog
```

## Test

1. with utility "ping": ping -c 500 -f 192.168.1.112 
2. with utility "iperf": iperf -c 192.168.1.112 -n 1M -i 1 -t 180 

There is one rule for NAT in `/etc/pf.conf.ports`: 

`nat pass on $ext_if from to any -> 10.1.6.0/24 source-hash test static-port`

a). Ping `192.168.1.112`:
`ping -c 500 -f 192.168.1.112`: 
```
PING 192.168.1.112 (192.168.1.112) 56(84) bytes of data. 
--- 192.168.1.112 ping statistics --- 
500 packets transmitted, 398 received, 20% packet loss, time 1658ms 
rtt min/avg/max/mdev = 0.239/0.339/5.425/0.262 ms, ipg/ewma 3.323/0.328 ms 
```
b) On the server `192.168.1.112`: 
`iperf -s 80`

c) On the laptop: 
`iperf -c 192.168.1.112 -p 80 -n 1M -i 1 -t 180 `

## Netstat check
#### age0
```
# netstat -w1d -I age0: 
                    input (age0) output 
packets     errs     idrops    bytes      packets        errs    bytes    colls 
5247          0         0       7332276   1600          0     83700      0 
5286          0         0       7331330   1578          0     82296      0 
5278          0         0       7339278   1589          0     83754      0 
5312          0         0       7380344   1570          0     82728      0 
5328          0         0       7337764   1567          0     83160      0 
```
#### rl0
```
# netstat -w1d -I rl0: 
                 input (rl0) output 
packets     errs    idrops     bytes     packets      errs       bytes     colls 
1556          0       0            93508    5133        0       7275788   0 
1547          0       0            92832    5169        0       7337174   0 
1551          0       0            93072    5161        0       7321088   0 
1539          0       0            92352    5199        0       7381268   0 
1520          0       0            91212    5195        0       7367642   0 
```

## Router Status
```
# top –S: 
last pid: 6320; load averages: 0.07, 0.02, 0.00 up 1+18:19:20 10:08:26 
70 processes: 3 running, 55 sleeping, 12 waiting 
CPU: 0.0% user, 0.0% nice, 1.2% system, 4.7% interrupt, 94.2% idle 
Mem: 21M Active, 136M Inact, 89M Wired, 44K Cache, 59M Buf, 237M Free 
Swap: 2048M Total, 2048M Free 
```
