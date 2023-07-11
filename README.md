# cake-qos-simple

cake-qos-simple sets up instances of cake based on wan egress and ingress and provides a simple means to leverage the diffserv functionality in cake using DSCPs, which is useful when bandwidth is constrained. 

1) DSCPs can be set either by LAN clients and/or by the router on upload and are restored from conntracks on download; and 
2) cake is set up on upload (wan egress) and download (ifb based on wan) and packets are tinned according to their DSCPs

The principle of operation of cake-qos-simple is as follows:

1) set up an intermediate functional block for use with wan ingress (ifb-wan); and
2) mirror packets from wan ingress to ifb-wan having restored DSCPs from the conntracks
3) optionally overwrite ECN bits on upload and/or download
4) set up cake on wan egress (for upload) and on ifb-wan (for download)
5) set up appropriate nftables rules

This facilitates the optional use of the diffserv functionality in cake for improving qos when bandwidth is constrained by leveraging connection tracking (conntrack) in Linux.

To make use of this optional diffserv functionality, DSCPs must be:
a) written to packets on upload by LAN clients or route; and
b) written to conntracks on upload for restoration on download

## Service file: 'cake-qos-simple'

This service script:

- writes out a default customizable config file
- writes out a default customizable nftables file
- creates necessary interfaces/tc calls on start/stop
- loads/unloads the nftables rules on start/stop
- checks the nftables rules before loading

## config

cake-qos-simple is configured using a simple configuraiton file kept in /root/cake-qos-simple/

## nftables script 'nft.rules`

cake-qos-simple generated an initial default nftables script nft.rules , which provides a template for classifying DSCPs in the router and storing DSCPs set on upload (wan egress) in the router of by LAN clients in conntracks for restoration on download (wan ingress). This diagram is useful to understand the nftables script: 

![image](https://user-images.githubusercontent.com/10721999/188932157-881bd4ef-e1ab-46d7-bd1b-966e78f00429.png)

Source: https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks

## Required packages

This script requires at least the following packages:

- **tc-tiny**
- **kmod-ifb**
- **kmod-sched**
- **kmod-sched-core**
- **kmod-sched-cake**
- **kmod-sched-ctinfo**

## Installation on OpenWrt

To install:

- install requisite packages as required
- place cake-qos-simple in /etc/init.d
- chmod +x cake-qos-simple
- place 11-cake-qos-simple in /etc/hotplug.d/iface/
- chmod +x 11-cake-qos-simple
- generate default config in /root/cake-qos-simple/config using: `service cake-qos-simple gen_config`
- edit default configuration lines in config to set interface(s), CAKE parameters and nftables variables (will be imported to auto-generated nft.rules file)
- generate default nftables rules based on config in /root/cake-qos-simple/nft.rules using: `service cake-qos-simple gen_nft_rules`
- edit default nftables rules in nft.rules as desired
- service cake-qos-simple enable
- service cake-qos-simple start
- verify correct operation by running `service cake-qos-simple status`, `service cake-qos-simple download` and `service cake-qos-simple upload` and optionally by running tcpdump with the -v switch to inspect TOS values in packets

Here is a guide to completing the above steps in your SSH client.

Firstly, install the requisite packages:
```
opkg update && opkg install tc-tiny kmod-ifb kmod-sched kmod-sched-core kmod-sched-cake kmod-sched-ctinfo
```

Next, obtain the service script 'cake-qos-simple' and set the executable bit:
```
wget -O /etc/init.d/cake-qos-simple "https://raw.githubusercontent.com/lynxthecat/cake-qos-simple/master/cake-qos-simple"
chmod +x /etc/init.d/cake-qos-simple
```

Next, generate a default config file in /root/cake-qos-simple/:
```
service cake-qos-simple gen_config
```

Next, edit the cake-qos-simple service script to set interface(s), CAKE parameters and nftables variables:
```
vi /root/cake-qos-simple/config
```

Next, generate a default nft.rules file in /root/cake-qos-simple/:
```
service cake-qos-simple gen_nft_rules
```

Next, edit the cake-qos-simple service script to set interface(s), CAKE parameters and nftables variables:
```
vi /root/cake-qos-simple/nft.rules
```

Next, install the hotplug script and set the exectuable bit:
```
wget -O /etc/hotplug.d/iface/11-cake-qos-simple "https://raw.githubusercontent.com/lynxthecat/cake-qos-simple/master/11-cake-qos-simple"
chmod +x /etc/hotplug.d/iface/11-cake-qos-simple
```

And to use cake-qos-simple - see

```
root@OpenWrt-1:/# service cake-qos-simple
Syntax: /etc/init.d/cake-qos-simple [command]

Available commands:
        start           Start the service
        stop            Stop the service
        restart         Restart the service
        reload          Reload configuration files (or restart if service does not implement reload)
        enable          Enable service autostart
        disable         Disable service autostart
        enabled         Check if service is started on boot

        cake-qos-simple custom commands:

        status                  show status summary
        upload                  show stats for upload interface
        download                show stats for download interface
        gen_config              generate default config
        gen_nft_rules           generate default nftables rules based on config
 ```

So e.g.:

```
# Start cake-qos-simple
service cake-qos-simple start

# Stop cake-qos-simple
service cake-qos-simple stop

# Check download stats
service cake-qos-simple download

# Check upload stats
service cake-qos-simple upload

# Check status
service cake-qos-simple status

# Generate default config
service cake-qos-simple gen_config

# Generate default nft.rules based on config
service cake-qos-simple gen_nft_rules
```

### Overwriting ECN bits ###

There are situations in which it is desirable to prevent cake from marking rather than dropping packets. 

Firstly, see discussion on OpenWrt forums around [here](https://forum.openwrt.org/t/effect-of-set-tcp-ecn-to-off-on-ecn/63921/13). 

Secondly, whenever an ISP bleaches ECN bits (which is common for mobile operators), the bleaching occurs prior to cake on download, but after cake on upload - thus cake is blind to the bleaching in the upload direction, and this means that by default cake will mark packets on upload in response to saturation, which is futile because the ISP ultimately bleaches those markings anyway. So in this situation it strikes me as better to proactively scrub the ECN bits on upload before cake sees the packets to prevent cake form ineffectively marking the ECN bits.

cake-qos-simple facilitates overwriting ECN bits before the cake instances see the packets on upload and/or download. 

This is controlled by setting appropriate values for:

```
overwrite_ecn_val_ul=0 # overwrite existing ecn bits with decimal value (e.g. 0, 1, 2, 3), else "" to disable
overwrite_ecn_val_dl=0 # overwirte existing ecn bits with decimal value (e.g. 0, 1, 2, 3), else "" to disable
```

### To setup DSCP setting by the router ###

- amend the PROTO_DPORT_DSCP_MAP variable in the config:
```
# correspondence between protocol, destination port and DSCPs
define PROTO_DPORT_DSCP_MAP = {
        tcp . 53 : goto dscp_set_voice,  # DNS
        udp . 53 : goto dscp_set_voice,  # DNS
        tcp . 853 : goto dscp_set_voice, # DNS-over-TLS
        udp . 853 : goto dscp_set_voice, # DNS-over-TLS
        udp . 123 : goto dscp_set_voice  # NTP
}
```
as required to assign DSCPs out of bulk, besteffort, video and voice in dependence of protocol 'tcp' or 'udp' and destination port.

- amend nft.rules with any further desired nftables rules. 

This can optionally override anything set by the LAN clients. 

### Setting DSCPs in Microsoft Windows Based LAN Clients ###

If using Microsoft Windows, DSCPs can be set at the application level by creating the registry key 'QoS' (it not present) as in:

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\QoS\

And then creating the string "Do not use NLA" inside the QoS key with value "1"

![image](https://user-images.githubusercontent.com/10721999/187535155-d4fd286b-9f20-40ce-8ff9-98ed36591721.png)

And then by creating appropriate QoS policies in the Local Group Policy Editor:

![image](https://user-images.githubusercontent.com/10721999/187747512-4c608e11-92a9-4484-b07f-3695baa98b85.png)

### Verifying Correct Operation and DSCP Handling ###

 Verify correct operation and DSCP handling using tcpdump:
 
   ```bash
      opkg update; opkg install tcpdump
      # First check correct flows and DSCPs correctly set by your LAN client on upload
      tcpdump -i wan -vv
      # Second check correct flows and corresponding DSCPs are getting set by router on download
      tcpdump -i ifb-wan -vv
   ``` 
   
## VPN

If using a VPN then cake-qos-simple is not appropriate because cake will not see all the flows and so flow fairness will not work properly. Instead check out the following alternatives to deal with mixture of encrypted and unencrypted flows:

https://github.com/lynxthecat/cake-dual-ifb

https://github.com/lynxthecat/cake-wg-pbr
