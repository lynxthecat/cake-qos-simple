# cake-qos-simple

cake-qos-simple sets up instances of cake based on wan egress and ingress and provides a simple means to leverage the diffserv functionality in cake using DSCPs, which is useful when bandwidth is constrained. 

1) DSCPs can be set either by LAN clients and/or by the router on upload and are restored from conntracks on download; and 
2) cake is set up on upload (wan egress) and download (ifb based on wan) and packets are tinned according to their DSCPs

## Service file: 'cake-qos-simple'

The service file 'cake-qos-simple':

1) sets up an intermediate functional block for use with wan ingress (ifb-wan); and
2) redirects packets from wan ingress to ifb-wan having restored DSCPs from the conntracks
3) sets up cake on wan egress (for upload) and on ifb-wan (for download)

DSCPs must be:

1) written to packets on upload by LAN clients or router (e.g. using cake-qos-simple.nft or otherwise); and
2) written to conntracks on upload for restoration on download

## nftables script: 'cake-qos-simple.nft'

This nftables script provides a template for classifying DSCPs in the router and storing DSCPs set on upload (wan egress) in the router of by LAN clients in conntracks for restoration on download (wan ingress). This diagram is useful to understand the nftables script: 

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
- edit configuration lines in cake-qos-simple to set interface(s) and CAKE parameters
- place 11-cake-qos-simple in /etc/hotplug.d/iface/
- chmod +x 11-cake-qos-simple
- place cake-qos-simple.nft in /usr/share/nftables.d/ruleset-post/
- edit cake-dual-ifb.nft for your use case 
- service firewall restart
- verify interfaces (e.g. replace or delete br-lan / br-guest lines as required)

Here is a guide to completing the above steps in your SSH client.

Firstly, install the requisite packages:
```
opkg update && opkg install tc-tiny kmod-ifb kmod-sched kmod-sched-core kmod-sched-cake kmod-sched-ctinfo
```

Next, obtain the service script 'cake-qos-simple':
```
wget -O /etc/init.d/cake-qos-simple "https://raw.githubusercontent.com/lynxthecat/cake-qos-simple/master/cake-qos-simple"
```

Now edit the cake-qos-simple service script to set interface(s) and CAKE parameters:
```
vi /etc/init.d/cake-qos-simple
```

Next, install the hotplug script and nftables file:
```
wget -O /etc/hotplug.d/iface/11-cake-qos-simple "https://raw.githubusercontent.com/lynxthecat/cake-qos-simple/master/11-cake-qos-simple"
mkdir /usr/share/nftables.d/ruleset-post/
wget -O /usr/share/nftables.d/ruleset-post/cake-qos-simple.nft "https://raw.githubusercontent.com/lynxthecat/cake-qos-simple/master/cake-qos-simple.nft"
```

Now edit the nftables file as required:
```
vi /usr/share/nftables.d/ruleset-post/cake-qos-simple.nft
```

And to use cake-qos-simple - see

```
root@OpenWrt-1:~# service cake-qos-simple
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

        status          show status summary
        upload          show stats for upload interface
        download        show stats for download interface
 ```

So e.g.:

```
# Start cake-qos-simple
/etc/init.d/cake-qos-simple start

# Stop cake-qos-simple
/etc/init.d/cake-qos-simple stop

# Check download stats
/etc/init.d/cake-qos-simple download

# Check upload stats
/etc/init.d/cake-qos-simple upload

# Check status
/etc/init.d/cake-qos-simple upload
```

### To setup DSCP setting by the router ###

- amend cake-qos-simple.nft as appropriate

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
