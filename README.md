# ⚡cake-qos-simple

If you like cake-qos-simple and can benefit from it, then please leave a ⭐ (top right) and become a [stargazer](https://github.com/lynxthecat/cake-qos-simple/stargazers)! And feel free to post any feedback on the official OpenWrt thread [here](https://forum.openwrt.org/t/cake-w-dscps-cake-qos-simple/147087). Thank you for your support.

cake-qos-simple sets up CAKE on WAN egress and ingress (using an IFB for download shaping). By default it provides a minimal best-effort CAKE setup. Optional DSCP classification/restoration can be enabled when diffserv prioritisation is wanted.

The principle of operation of cake-qos-simple is as follows:

1) set up an intermediate functional block for use with wan ingress (ifb-wan);
2) mirror packets from wan ingress to ifb-wan;
3) optionally classify/store DSCPs and restore them from conntracks;
4) optionally overwrite ECN bits on upload and/or download; and
5) set up cake on wan egress (for upload) and on ifb-wan (for download).

The default configuration uses CAKE `besteffort` and does not load cake-qos-simple nftables DSCP classification or ctinfo restoration. This keeps the core installation small.

To enable cake-qos-simple DSCP classification/storage and ctinfo restoration, set:

```
enable_dscp_restoration=1
```

and select an appropriate CAKE diffserv mode such as `diffserv4` in the upload and download CAKE options. CAKE itself can still honour DSCPs already present on packets whenever a diffserv mode is selected.

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

When `enable_dscp_restoration=1`, cake-qos-simple generates and loads an initial default nftables script `nft.rules`, which provides a template for classifying DSCPs in the router and storing DSCPs set on upload (WAN egress) by the router or LAN clients in conntracks for restoration on download (WAN ingress).

When `enable_dscp_restoration=0` (the default), the nftables rules are not required or loaded. This diagram is useful to understand the nftables script when DSCP restoration is enabled:

![image](https://user-images.githubusercontent.com/10721999/188932157-881bd4ef-e1ab-46d7-bd1b-966e78f00429.png)

Source: https://wiki.nftables.org/wiki-nftables/index.php/Netfilter_hooks

## Required packages

The core CAKE/IFB functionality requires:

- **tc-tiny**
- **kmod-ifb**
- **kmod-sched-core**
- **kmod-sched-cake**

DSCP restoration using conntrack additionally requires:

- **kmod-sched-ctinfo**

Optional ECN rewriting additionally requires:

- **kmod-sched**

## Installation on OpenWrt

To install:

- install requisite packages as required
- place cake-qos-simple in /etc/init.d
- chmod +x cake-qos-simple
- place 11-cake-qos-simple in /etc/hotplug.d/iface/
- chmod +x 11-cake-qos-simple
- generate default config in /root/cake-qos-simple/config using: `service cake-qos-simple gen_config`
- edit default configuration lines in config to set interface(s), CAKE parameters and whether DSCP restoration is enabled
- when `enable_dscp_restoration=1`, generate default nftables rules based on config in /root/cake-qos-simple/nft.rules using: `service cake-qos-simple gen_nft_rules`
- optionally edit the default nftables rules in nft.rules as desired
- when `enable_dscp_restoration=1` and using an OpenWrt version earlier than 23.05, edit nft.rules to replace the lines underneath 'chain store-dscp-in-conntrack' as directed in the comments
- service cake-qos-simple enable
- service cake-qos-simple start
- verify correct operation by running `service cake-qos-simple status`, `service cake-qos-simple download` and `service cake-qos-simple upload` and optionally by running tcpdump with the -v switch to inspect TOS values in packets

Here is a guide to completing the above steps in your SSH client.

Firstly, install the core packages:
```
apk update && apk add tc-tiny kmod-ifb kmod-sched-core kmod-sched-cake
```

If enabling DSCP restoration, additionally install:
```
apk add kmod-sched-ctinfo
```

Install `kmod-sched` as well if using the optional ECN rewriting settings.

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

If `enable_dscp_restoration=1`, generate a default nft.rules file in /root/cake-qos-simple/:
```
service cake-qos-simple gen_nft_rules
```

Then optionally edit the default nftables rules and/or, if using an OpenWrt version earlier than 23.05, edit nft.rules to replace the lines underneath 'chain store-dscp-in-conntrack' as directed in the comments:
```
vi /root/cake-qos-simple/nft.rules
```

The generated configuration already defaults to a minimal best-effort setup:
```
enable_dscp_restoration=0
cake_ul_options="besteffort triple-isolate nat wash ack-filter noatm overhead 0"
cake_dl_options="besteffort triple-isolate nat nowash ingress no-ack-filter noatm overhead 0"
```

To enable DSCP classification/restoration, install `kmod-sched-ctinfo`, set `enable_dscp_restoration=1`, select a CAKE diffserv mode, and generate `nft.rules`, for example:
```
enable_dscp_restoration=1
cake_ul_options="diffserv4 triple-isolate nat wash ack-filter noatm overhead 0"
cake_dl_options="diffserv4 triple-isolate nat nowash ingress no-ack-filter noatm overhead 0"

service cake-qos-simple gen_nft_rules
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

ECN rewriting is optional and disabled by default. It requires **kmod-sched** because the implementation uses `pedit` (and `csum` for IPv4).

The four settings operate independently on upload and download and on the two ECN-capable transport values:

```
overwrite_ul_ect_0_val=""
overwrite_ul_ect_1_val=""
overwrite_dl_ect_0_val=""
overwrite_dl_ect_1_val=""
```

Leave a value empty to make no change. Set a value to the decimal ECN field value to write when packets matching that ECT state are seen. The ECN field values are:

- `0` = Not-ECT
- `1` = ECT(1)
- `2` = ECT(0)
- `3` = CE

For example, to clear ECT(0) and ECT(1) on upload so that CAKE drops rather than ECN-marks those packets under congestion:

```
overwrite_ul_ect_0_val=0
overwrite_ul_ect_1_val=0
```

One reason to consider this is an upstream network that bleaches ECN after packets leave the router: CAKE could otherwise mark packets on upload even though those marks will not survive the path. Leave these settings empty unless there is a specific reason to rewrite ECN.

### To setup DSCP setting by the router ###

- amend the PROTO_DPORT_DSCP_MAP variable in the config:
```
# correspondence between protocol, destination port and DSCPs
# the format is:
# 'protocol' . 'destination port' . dscp_set_bulk OR dscp_set_besteffort OR dscp_set_video OR dscp_set_voice
# a port range can be specified using the form: X-Y
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
      apk update && apk add tcpdump
      # First check correct flows and DSCPs correctly set by your LAN client on upload
      tcpdump -i wan -vv
      # Second check correct flows and corresponding DSCPs are getting set by router on download
      tcpdump -i ifb-wan -vv
   ``` 
   
## VPN

If using a VPN then cake-qos-simple is not appropriate because cake will not see all the flows and so flow fairness will not work properly. Instead check out the following alternatives to deal with mixture of encrypted and unencrypted flows:

https://github.com/lynxthecat/cake-dual-ifb

https://github.com/lynxthecat/cake-wg-pbr
