#  Linux-router

Set Linux as router in one command. Able to Provide Internet, or create Wifi hotspot. Support transparent proxy (redsocks). Also useful for routing VM/containers.

It wraps `iptables`, `dnsmasq` etc. stuff. Use in one command, restore in one command or by `control-c`.

 
##  Features

Basic features:

- Create a NATed sub-network
- Provide Internet
- DHCP server and RA
- DNS server 
- IPv6 (NAT only for now)
- Creating Wifi hotspot:
  - Channel selecting
  - Choose encryptions: WPA2/WPA, WPA2, WPA, No encryption
  - Hidden SSID
  - Create AP on the same interface you are getting Internet (require same channel)
- Transparent proxy (redsocks)
- DNS proxy
- Compatible with NetworkManager (automatically set interface as unmanaged)

**For many other features, see below [CLI usage](#cli-usage-and-other-features)**

### Useful in these situations
```
Internet----(eth0/wlan0)-Linux-(wlanX)AP
                                       |--client
                                       |--client
```

```
                                    Internet
Wifi AP(no DHCP)                        |
    |----(wlan1)-Linux-(eth0/wlan0)------
    |           (DHCP)
    |--client
    |--client
```

```
                                    Internet
 Switch                                 |
    |---(eth1)-Linux-(eth0/wlan0)--------
    |--client
    |--client
```

```
Internet----(eth0/wlan0)-Linux-(eth1)------Another PC
```

```
Internet----(eth0/wlan0)-Linux-(virtual interface)-----VM/container
```
 
## Usage

### Share Internet to an interface

```
# lnxrouter -i eth1
```

### Create Wifi hotspot

```
# lnxrouter --ap wlan0 MyAccessPoint --password MyPassPhrase
```

### LAN without Internet

```
# lnxrouter -i eth1 -n
# lnxrouter --ap wlan0 MyAccessPoint --password MyPassPhrase -n
```

### Transparent proxy with tor

```
# lnxrouter -i eth1 --tp 9040 --dns-proxy 9053
```

In `torrc`

```
TransPort 0.0.0.0:9040 
DNSPort 0.0.0.0:9053
TransPort [::]:9040 
DNSPort [::]:9053
```
### Internet for LXC
Create a bridge
```
# brctl addbr lxcbr5
```
In LXC container `config`
```
lxc.network.type = veth
lxc.network.flags = up
lxc.network.link = lxcbr5
lxc.network.hwaddr = xx:xx:xx:xx:xx:xx
```
```
# lnxrouter -i lxcbr5
```

### Use as transparent proxy for LXD
Create a bridge
```
# brctl addbr lxdbr5
```
Create and add LXD profile
```
$ lxc profile create profile5
$ lxc profile edit profile5

### profile content ###
config: {}
description: ""
devices:
  eth0:
    name: eth0
    nictype: bridged
    parent: lxdbr5
    type: nic
name: profile5

$ lxc profile add <container> profile5
```
That should make one container have 2 profiles. `profile5` will override `eth0`.
```
# lnxrouter -i lxdbr5 --tp 9040 --dns-proxy 9053
```
To remove that new profile from container
```
$ lxc profile remove <container> profile5
```

#### To not use profile
Add device `eth0` to container overriding default `eth0`
```
$ lxc config device add <container> eth0 nic name=eth0 nictype=bridged parent=lxdbr5
```
To remove the customized `eth0` to restore default `eth0`
```
$ lxc config device remove <container> eth0
```

### Use as transparent proxy for VirtualBox
On VirtualBox's global settings, create a host-only network `vboxnet5` with DHCP disabled.
```
# lnxrouter -i vboxnet5 --tp 9040 --dns-proxy 9053
```
### CLI usage and other features

```
Usage: lnxrouter [options] 

Options:
    -h, --help              Show this help
    --version               Print version number

    -i <interface>          Interface to share Internet to.
                            An NATed subnet is made upon it.
                            To create Wifi hotspot use '--ap' instead
    -n                      Disable Internet sharing
    --tp <port>             Transparent proxy.
                            redirect non-LAN tcp and udp traffic to port.
                            Usually used with '--dns-proxy'
    
    -g <gateway>            Set gateway IPv4 address, netmask is /24 .
                            (default: 192.168.18.1)
    -6                      Enable IPv6 (NAT)
    --p6 <prefix>           Set IPv6 prefix (length 64)
                            (default: fd00:1:1:1:: )
    --dns-proxy <port>      DNS server redirect queries to port
    --no-serve-dns          Disable DNS server
    --no-dnsmasq            Disable dnsmasq server completely (DHCP, DNS, RA)
    --log-dns               Show DNS server query log
    --dhcp-dns <IP1[,IP2]>|no
                            Set IPv4 DNS offered by DHCP 
                            (default: gateway as DNS)
    --dhcp-dns6 <IP1[,IP2]>|no
                            Set IPv6 DNS offered by DHCP(RA) 
                            (default: gateway as DNS)
                            Note IPv6 addresses need '[]' around
    -d                      DNS server will take into account /etc/hosts
    -e <hosts_file>         DNS server will take into account additional 
                            hosts file
    
    --mac <MAC>             Set MAC address
 
  Wifi hotspot options:
    --ap <wifi interface> <SSID>
                            Create Wifi access point
    --password <password>   Wifi password
    
    --hidden                Hide access point (not broadcast SSID)
    --no-virt               Do not create virtual interface
                            Using this you can't use same wlan interface
                            for both Internet and AP
    -c <channel>            Channel number (default: 1)
    --country <code>        Set two-letter country code for regularity
                            (example: US)
    --freq-band <GHz>       Set frequency band: 2.4 or 5 (default: 2.4)
    --driver                Choose your WiFi adapter driver (default: nl80211)
    -w <WPA version>        Use 1 for WPA, use 2 for WPA2, use 1+2 for both
                            (default: 1+2)
    --psk                   Use 64 hex digits pre-shared-key instead of
                            passphrase
    --mac-filter            Enable Wifi hotspot MAC address filtering
    --mac-filter-accept     Location of Wifi hotspot MAC address filter list
                            (defaults to /etc/hostapd/hostapd.accept)
    --hostapd-debug <level> 1 or 2. Passes -d or -dd to hostapd
    --isolate-clients       Disable wifi communication between clients
    --ieee80211n            Enable IEEE 802.11n (HT)
    --ieee80211ac           Enable IEEE 802.11ac (VHT)
    --ht_capab <HT>         HT capabilities (default: [HT40+])
    --vht_capab <VHT>       VHT capabilities
    --no-haveged            Do not run haveged automatically when needed

  Instance managing:
    --daemon                Run in background
    --list-running          Show running instances
    --list-clients <id>     List clients of an instance
    --stop <id>             Stop a running instance
        For <id> you can use PID or subnet interface name.
        You can get them with '--list-running'
```
> On exiting it restores changes done to system, except `/proc/sys/net/ipv4/ip_forward` and `/proc/sys/net/ipv6/conf/all/forwarding` set by NAT mode.

## Dependencies
- bash
- procps or procps-ng
- iproute2
- dnsmasq
- iptables

Wifi hotspot:

- hostapd
- iw
- iwconfig (you only need this if 'iw' can not recognize your adapter)
- haveged (optional)

## TODO

- Option to ban private network access
- Option to randomize MAC, IP, SSID, password
- Option to redirect all DNS traffic

## Thanks

Many thanks to project [create_ap](https://github.com/oblique/create_ap).
