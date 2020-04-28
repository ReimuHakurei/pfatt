# About

I'm assuming you already know what this project is. The original repo disappeared for some reason. The original project has two modes, bridge mode and supplicant mode. I have no interest in the older proxy-based method, but (for now) it remains here.

If you want any of the stuff in the original repo's readme, see README-orig.md.

I assume you already have certificates extracted. If not, head over to eBay or do some Googling to figure it out.

## Install

1. Grab this repo to your local machine.
```
git clone https://github.com/ReimuHakurei/pfatt
```
2. Next, edit all configuration variables in `pfatt.sh`.

3. Upload the pfatt directory to `/conf` on your pfSense box.
```
scp -r pfatt root@pfsense:/conf/
```
4. Upload your extracted certs (from EAP-TLS_8021x_xxxxxx-xxxxxxxxxxxxxxx.tar.gz or otherwise) to `/conf/pfatt/wpa`. You should have three files in the wpa directory as such. You may also need to match the permissions.
```
[2.4.4-RELEASE][root@pfsense.knox.lan]/conf/pfatt/wpa: ls -al
total 19
drwxr-xr-x  2 root  wheel     5 Jan 10 16:32 .
drwxr-xr-x  4 root  wheel     5 Jan 10 16:33 ..
-rw-------  1 root  wheel  5150 Jan 10 16:32 ca.pem
-rw-------  1 root  wheel  1123 Jan 10 16:32 client.pem
-rw-------  1 root  wheel   887 Jan 10 16:32 private.pem
```
5. Edit your `/conf/config.xml` to include `<earlyshellcmd>/conf/pfatt/bin/pfatt.sh</earlyshellcmd>` above `</system>`. 

6. Connect cables
    - `$ONT_IF` to ONT (outside)
    - `LAN NIC` to local switch (as normal)

7. Prepare for console access.
8. Reboot.
9. pfSense will detect new interfaces on bootup. Follow the prompts on the console to configure `ngeth0` as your pfSense WAN. Your LAN interface should not normally change. However, if you moved or re-purposed your LAN interface for this setup, you'll need to re-apply any existing configuration (like your VLANs) to your new LAN interface. pfSense does not need to manage `$EAP_BRIDGE_IF` or `$ONT_IF`. I would advise not enabling those interfaces in pfSense as it can cause problems with the netgraph.
10. In the webConfigurator, configure the  WAN interface (`ngeth0`) to DHCP using the MAC address of your Residential Gateway.

If everything is setup correctly, EAP authentication should complete. Netgraph should be tagging the WAN traffic with VLAN0, and your WAN interface is configured with a public IPv4 address via DHCP.

#### Dumping the NAND

User @KhoasT posted instructions for dumping the NAND. See the comment on devicelocksmith's site [here](https://www.devicelocksmith.com/2018/12/eap-tls-credentials-decoder-for-nvg-and.html?showComment=1549236760112#c5606196700989186087).

# IPv6 Setup

Once your netgraph setup is in place and working, there aren't any netgraph changes required to the setup to get IPv6 working. These instructions can also be followed with a different bypass method other than the netgraph method. Big thanks to @pyrodex1980's [post](http://www.dslreports.com/forum/r32118263-) on DSLReports for sharing your notes.

This setup assumes you have a fairly recent version of pfSense. I'm using 2.4.4.

**DUID Setup**

1. Go to _System > Advanced > Networking_
2. Configure **DHCP6 DUID** to _DUID-EN_
3. Configure **DUID-EN** to _3561_
4. Configure your **IANA Private Enterprise Number**. This number is unique for each customer and (I believe) based off your Residential Gateway serial number. You can generate your DUID using [gen-duid.sh](https://github.com/aus/pfatt/blob/master/bin/gen-duid.sh), which just takes a few inputs. Or, you can take a pcap of the Residential Gateway with some DHCPv6 traffic. Then fire up Wireshark and look for the value in _DHCPv6 > Client Identifier > Identifier_. Add the value as colon separated hex values `00:00:00`.
5. Save

**WAN Setup**

1. Go to _Interfaces > WAN_
1. Enable **IPv6 Configuration Type** as _DHCP6_
1. Scroll to _DCHP6 Client Configuration_
1. Enable **DHCPv6 Prefix Delegation size** as _60_
1. Enable _Send IPv6 prefix hint_
1. Enable _Do not wait for a RA_
1. Save

**LAN Setup**

1. Go to _Interfaces > LAN_
1. Change the **IPv6 Configuration Type** to _Track Interface_
1. Under Track IPv6 Interface, assign **IPv6 Interface** to your WAN interface.
1. Configure **IPv6 Prefix ID** to _1_. We start at _1_ and not _0_ because pfSense will use prefix/address ID _0_ for itself and it seems AT&T is flakey about assigning IPv6 prefixes when a request is made with a prefix ID that matches the prefix/address ID of the router.
1. Save

If you have additional LAN interfaces repeat these steps for each interface except be sure to provide an **IPv6 Prefix ID** that is not _0_ and is unique among the interfaces you've configured so far.

**DHCPv6 Server & RA**

1. Go to _Services > DHCPv6 Server & RA_
1. Enable DHCPv6 server on interface LAN
1. Configure a range of ::0001 to ::ffff:ffff:ffff:fffe
1. Configure a **Prefix Delegation Range** to _64_
1. Save
1. Go to the _Router Advertisements_ tab
1. Configure **Router mode** as _Stateless DHCP_
1. Save

That's it! Now your clients should be receiving public IPv6 addresses via DHCP6.

# Troubleshooting

If it freezes at waiting for EAP (give it a couple of minutes first, sometimes it takes a bit), Ctrl-C and drop to shell, then wpa_cli status. Have fun.

Oh and, is your clock set correctly? :)