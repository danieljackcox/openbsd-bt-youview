# BT IPTV (YouView) on OpenBSD

This is a writeup on using OpenBSD for home networking (routing, firewall) on a BT broadband link with Youview IPTV. It took me some days of tinkering to get this to work.

## Hardware

The connection is a standard 80/20 mbit/s VDSL2 link, with a BT OpenReach modem (Huawei Echolife HG612). OpenReach ECI modems also work as well as other standard VDSL2 modems however I've had the best success and most stable connection with the Huawei (and this modem can be unlocked).

The server running OpenBSD is a [PCEngines APU2](https://pcengines.ch/apu2.htm) with three ethernet ports, `em0`, `em1` and `em2`. I'm using `em0` as the WAN interface connected directly to the modem and `em1` as the LAN interface, connected to a network switch and the clients.

## Setup

### Overview
A broadband connection is made using PPPoE on vlan 101. IPTV mutlicast is injected from the cabinet outside of the PPP link and arrives on the bare ethernet.

### Broadband data connection (PPPoE)

The standard broadband data connection uses PPPoE, users on non-openreach modems should take care as ALL data from the modem ethernet is tagged on VLAN 101, Openreach modems automatically listen on vlan 101 and detags before it reaches you.

This is where BT does things in a fairly standard way as just a regular PPPoE connection is made, BT also supports RFC 4638, so the PPPoE connection can have an MTU of 1500. I have done it this way to avoid fragmentation issues.

With `em0` as the ethernet connection to the modem my `/etc/hostname.em0` looks like this

```
mtu 1508 up
```
MTU is set to 1508 as this includes the MTU of the PPPoE connection (1500) plus 8 bits encapsulation.

For the PPPoE connection my `/etc/hostname.pppoe0` is as follows
```
inet 0.0.0.0 255.255.255.255 NONE mtu 1500 \
pppoedev em0 authproto chap \
authname bthomehub@btbroadband.com up
dest 0.0.0.1
!/sbin/route add default -ifp pppoe0 0.0.0.1
```

Change the `pppoedev em0` term to `pppoedev vlan101` and create the `/etc/hostname.vlan101` file if you are handling the vlan detagging yourself.

Once a connection is made `pppoe0` will be the default route, you can then build tunnels on top of this if you wish.

Note: I had some issues both the ECI and huawei openreach modems where the lifetime of the PPPoE connection would not be longer than 6 minutes. This was apparently caused by the modems doing an aggressive handshake with the cabinet despite a poor SNR. I solved this by unlocking the huawei modem (there are instructions online for this), telnetting in and issuing a command to renegotiate with higher SNR.
```
username: admin
password: admin
> sh
> xdslcmd configure --snr 120
```
You can find more details on this [here](https://www.increasebroadbandspeed.co.uk/SNR-tweak).

### YouView IPTV
This was a little tricky to get working and it took days of playing around, reading online and looking through packets with wireshark.

The crux of the issue is that the IPTV multicast stream required for youview is injected from the cabinet *outside* of the PPP tunnel.
You will require `igmpproxy` to register the streams and provide correct multicast routing.
Enable `igmpproxy` and multicast routing

In `/etc/rc.conf.local`
```
multicast=YES
multicast_router=YES
pkg_scripts=igmpproxy
```
In `/etc/sysctl.conf`
```
net.inet.ip.mforwarding=1
```

Some of these options might not be needed, I haven't tested removing any yet.

 In order to access and route the multicast stream you need to assign an address to the WAN ethernet, update the `/etc/hostname.em0` file to reflect this
```
mtu 1508 up
inet 10.22.22.10 255.255.255.0 10.22.22.255
```
You can choose any IP address for this, BT doesn't care but it's wise to choose one from the private block that you won't use.

For `igmpproxy` you'll want to update `/etc/igmpproxy.conf` to include the interfaces and multicast address blocks
```
quickleave
phyint em0 upstream # WAN interface
        altnet 224.0.0.0/4 # youviw IPTV address blocks
        altnet 109.159.247.0/24

phyint em1 downstream # LAN interface
```

The final part of this are the appropriate `pf` rules
```
# $iptv_if is the external WAN ethernet, not the pppoe interface
# $int_if is the LAN interface

pass in on $int_if proto igmp allow-opts #allow-opts is required here
pass in on $iptv_if proto udp from any to 224.0.0.0/4 port 5802 #allow the UDP multicast streams in

pass in on $iptv_if proto igmp allow-opts


pass out quick on $int_if proto igmp allow-opts
pass out quick on $int_if proto udp from any to 224.0.0.0/4 port 5802 #allow multicast streams on local network
pass out on $iptv_if proto igmp from ($iptv_if) to any allow-opts
pass out on $iptv_if proto igmp from ($iptv_if) to any allow-opts
```
