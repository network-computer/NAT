## Background
This guide will show you how to turn an OpenBSD system into a router. First, we'll define what this router `(also called a "gateway")` will actually do, since your requirements may vary.
- Performing network address translation (NAT)
- Giving a laptop and server static IPs based on their MAC address
- Handing out IP addresses to other clients via DHCP
- Allowing incoming connections to a local web server
- Doing DNS caching for the LAN
- Providing wireless connectivity (requires a supported card)
This example will use two em(4) NICs and an athn(4) wireless card. Replace the em0, em1 and athn0 interface names as appropriate. Sample configuration files are provided, but you're encouraged to read the man pages to understand their full capability.

## Networking
We'll begin with some initial network configuration, using a 192.168.1.0/24 subnet for the wired clients and 192.168.2.0/24 for the wireless.
```
# echo 'net.inet.ip.forwarding=1' >> /etc/sysctl.conf
# echo 'dhcp' > /etc/hostname.em0 # if you have a static IP, use that instead
# echo 'inet 192.168.1.1 255.255.255.0 192.168.1.255' > /etc/hostname.em1
# vi /etc/hostname.athn0
```
Add the following, changing the mode/channel if needed:
media autoselect mode 11n mediaopt hostap chan 1
nwid AccessPointName wpakey VeryLongPassword
inet 192.168.2.1 255.255.255.0
OpenBSD defaults to allowing only WPA2-CCMP connections in HostAP mode. If you need support for older (insecure) protocols, they must be explicitly enabled.

## DHCP
Clients need IP addresses, so we'll set dhcpd(8) to start on boot. Configuration is done via the dhcpd.conf(5) file.
```
# rcctl enable dhcpd
# rcctl set dhcpd flags em1 athn0
# vi /etc/dhcpd.conf
```
```
Take this example and adjust to fit your needs:
subnet 192.168.1.0 netmask 255.255.255.0 {
	option routers 192.168.1.1;
	option domain-name-servers 192.168.1.1;
	range 192.168.1.4 192.168.1.254;
	host myserver {
		fixed-address 192.168.1.2;
		hardware ethernet 00:00:00:00:00:00;
	}
	host mylaptop {
		fixed-address 192.168.1.3;
		hardware ethernet 11:11:11:11:11:11;
	}
}
```
You can specify another RFC 1918 address space here if you prefer, or even a public IP block if you have one. Using this example, clients will query a local DNS server, detailed in a later section. If you don't plan on using a local DNS server, replace the IPs in the domain-name-servers lines with the address of your preferred upstream resolver.

## Firewall

The centerpiece of this guide is the pf.conf(5) file. It's highly recommended to familiarize yourself with it, and PF in general, before copying this example. Each section will be explained in more detail.
```
# vi /etc/pf.conf
```

```
A configuration for a gateway system might look like this:
wired = "em1"
wifi  = "athn0"
table <martians> { 0.0.0.0/8 10.0.0.0/8 127.0.0.0/8 169.254.0.0/16     \
	 	   172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 224.0.0.0/3 \
	 	   192.168.0.0/16 198.18.0.0/15 198.51.100.0/24        \
	 	   203.0.113.0/24 }
set block-policy drop
set loginterface egress
set skip on lo0
match in all scrub (no-df random-id max-mss 1440)
match out on egress inet from !(egress:network) to any nat-to (egress:0)
antispoof quick for { egress $wired $wifi }
block in quick on egress from <martians> to any
block return out quick on egress from any to <martians>
block all
pass out quick inet
pass in on { $wired $wifi } inet
pass in on egress inet proto tcp from any to (egress) port { 80 443 } rdr-to 192.168.1.2
```
Now we'll break this ruleset down and explain what each line does.

```
wired = "em1"
wifi  = "athn0"
```
These are macros, used to make overall maintenance easier. Macros can be referenced throughout the ruleset after being defined.

```
table <martians> { 0.0.0.0/8 10.0.0.0/8 127.0.0.0/8 169.254.0.0/16     \
	 	   172.16.0.0/12 192.0.0.0/24 192.0.2.0/24 224.0.0.0/3 \
	 	   192.168.0.0/16 198.18.0.0/15 198.51.100.0/24        \
	 	   203.0.113.0/24 }
```
This is a table of non-routable private addresses that will be used later.

```
set block-policy drop
set loginterface egress
set skip on lo0
```
PF allows certain options to be set at runtime. The block-policy decides whether rejected packets should return a TCP RST or be silently dropped. The loginterface is exactly what it sounds like: which interface should have packet and byte statistics collection enabled. These statistics can be viewed with the pfctl -si command. In this case, we're using the egress group rather than a specific interface. The egress keyword automatically chooses the interface that holds the default route, or the em0 WAN interface in our example. Finally, skip allows you to completely omit a given interface from packet processing. The firewall will ignore loopback traffic on the lo(4) interface.

```
match in all scrub (no-df random-id max-mss 1440)
match out on egress inet from !(egress:network) to any nat-to (egress:0)
```
The match rules used here accomplish two things: normalizing incoming packets and performing network address translation, with the egress interface between the LAN and the public internet. For a more detailed explanation of match rules and their different options, refer to the pf.conf(5) manual.

```
antispoof quick for { egress $wired $wifi }
block in quick on egress from <martians> to any
block return out quick on egress from any to <martians>
```
The antispoof keyword provides some protection from packets with spoofed source addresses. Packets coming in on the egress interface should be dropped if they appear to be from the list of unroutable addresses we defined. Such packets were likely sent due to misconfiguration, or possibly as part of a spoofing attack. Similarly, our clients should not attempt to connect to such addresses. We'll specify the "return" action to prevent annoying timeouts for users. Note that this can cause problems if you're doing double NAT.

```
block all
```
The firewall will set a "default deny" policy for all traffic. This means we will only allow incoming and outgoing connections that we explicitly put in our ruleset.

```
pass out quick inet
```
Allow outgoing IPv4 traffic from both the gateway itself and the LAN clients.

```
pass in on { $wired $wifi } inet
```
Allow internal LAN traffic.

```
pass in on egress inet proto tcp from any to (egress) port { 80 443 } rdr-to 192.168.1.2
```
Forward incoming connections (on TCP ports 80 and 443, for a web server) to our machine at 192.168.1.2. This is merely an example of port forwarding.

## DNS
At this point, clients should be assigned IP addresses and granted access to the internet, while being protected by the firewall. If that's all you need, you're done and can reboot now. That being said, a DNS cache is common addition to a gateway system.
When clients issue a DNS query, they'll first hit the unbound(8) cache. If it doesn't have the answer, it goes out to the upstream resolver that you've configured. Results are then fed to the client and cached for a period of time, making future lookups of the same address quicker.
```
# rcctl enable unbound
# vi /var/unbound/etc/unbound.conf
```
Something like this should work for most setups:
```
server:
	interface: 192.168.1.1
	interface: 192.168.2.1
	interface: 127.0.0.1
	access-control: 192.168.1.0/24 allow
	access-control: 192.168.2.0/24 allow
	do-not-query-localhost: no
	hide-identity: yes
	hide-version: yes

forward-zone:
        name: "."
        forward-addr: 1.2.3.4  # IP of the upstream resolver
```
Further configuration options can be found in unbound.conf(5). Outgoing DNS lookups may also be encrypted with the dnscrypt-proxy package -- see its included README file for details.
If you want the gateway to use the caching resolver for lookups too, don't forget to change its /etc/resolv.conf file to point to 127.0.0.1. Since the majority of routers won't be doing many DNS queries, this is likely not needed. Also note that it will need to be changed back to an actual resolver when using bsd.rd for network-based upgrades if you do so.
