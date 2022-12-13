# Route-Based & Policy-Based VPN

## Content

- [Introduction](#introduction)
- [Static Route-Based VPN](#Static-Route-Based-VPN)
- [Dynamic Route-Based VPN](#Dynamic-Route-Based-VPN)
- [Route-Based VPN with High-Availability](#Route-Based-VPN-with-High-Availability)
- [Route-Based VPN SLA](#Route-Based-VPN-SLA)
- [Route-Based VPN Azure](#Route-Based-VPN-Azure)
- [Policy-Based VPN](#Policy-Based-VPN)
- [Summary](#Summary)

## Introduction 
Site-to-Site VPN tunnels can be configured in two different ways:

- Route-Based VPN Tunnels
- Policy-Based VPN Tunnels

This guideline is to help understand the characteristics of each method, the distinction between the two and considerations when selecting your tunnel-type. Main goal here is to inform and help network/security engineers or anyone who is interested in this topic. We will be using simple architectures, starting with a couple of Cisco devices in an emulator. Then we will take a look at VPN-configurations in Azure.
> __Note__
that some of the wording might be used interchangeably, e.g.: 
Phase 1 and Phase 2 is for IKEv1, but we will stick to the same wording for IKEv2 which has its own naming convention.


## Static Route-Based VPN
![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/RoutePolicyVPN.png)

For this guide we will be using Cisco 3745 routers emulated in GNS3. Please note that this is for demo purposes only, other vendors offer similar capabilities. You can apply lessons learned with other appliances. For this scenario we want to connect two offices together over an IPSEC tunnel using the route-based method. Take a look at the configuration of router R1:

```
# Configuration Phase 1 
crypto isakmp policy 1
 encr aes 256
 authentication pre-share
crypto isakmp key myvpnkey address 2.2.2.1

# Configuration Phase 2
crypto ipsec transform-set DemoSet esp-aes 256 esp-sha-hmac
!
crypto ipsec profile DemoProfile
 set transform-set DemoSet 

# Create Virtual Tunnel Interface
 interface Tunnel1 
 ip address 169.254.220.1 255.255.255.0 
 tunnel source FastEthernet0/0
 tunnel destination 2.2.2.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile DemoProfile
```


Phase 1 and Phase 2 configuration is not too dissimilar to a policy-based configuration.  What sets the route-based apart is the Virtual Tunnel Interface (VTI) that is created. 

- A tunnel interface needs an IP-address, this IP-address is only used inside the VPN-tunnel. Free to select the IP-address, if the other side of the Virtual Tunnel Interface is within the same subnet address space and the IP does not conflict with other addresses on your router. The 169.254.x.x range is therefore a popular choice.
- Because we now have a VTI, we can route traffic through this tunnel like we would with a regular interface. E.g.:

```
ip route 10.20.20.0/24 via FastEthernet0
```
This command tells our router: if you want to get to the network 10.20.20.0/24, send that traffic out to interface FastEthernet0. A Virtual Tunnel Interface works in the same way:

```
ip route 10.20.20.0/24 via Tunnel1
```
Now we are telling the router: if you want to get to network 10.20.20.0/24, send that traffic to the VPN-tunnel. This allows us with greater flexibility compared to the policy-based VPN:

- High-availability, multiple VPN-tunnels can exist at the same time. This allows for Active-Active and Active-Passive setup.
- Dynamic routing, over this VPN tunnel we can run a dynamic routing protocol like BGP. If a new network becomes available, or even obsolete, the other side will inform us of that change. As a result, no traffic will traverse the VPN-tunnel if the other side says the network is no longer an option.



![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/RoutePolicyVPN%20-%20Configuration.png)

Configuration of the routers R1 and R2, keep note that this configuration is for the Cisco IOS-images used in this lab. Your appliance is likely to have a different syntax and configuration method. Now, look at the state of the VPN-tunnels:
```
R1#show crypto isakmp sa
dst             src             state          conn-id slot status
1.1.1.1         2.2.2.1         QM_IDLE              1    0 ACTIVE

R1#show interfaces tunnel 1
Tunnel1 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 169.254.220.1/24
  MTU 1514 bytes, BW 9 Kbit/sec, DLY 500000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation TUNNEL, loopback not set
  Keepalive not set
  Tunnel source 1.1.1.1 (FastEthernet0/0), destination 2.2.2.1
  Tunnel protocol/transport IPSEC/IP
```

```
R2#show crypto isakmp sa
dst             src             state          conn-id slot status
1.1.1.1         2.2.2.1         QM_IDLE              1    0 ACTIVE

R2#show interfaces tunnel1
Tunnel1 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 169.254.220.2/24
  MTU 1514 bytes, BW 9 Kbit/sec, DLY 500000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation TUNNEL, loopback not set
  Keepalive not set
  Tunnel source 2.2.2.1 (FastEthernet0/0), destination 1.1.1.1
  Tunnel protocol/transport IPSEC/IP
```

The tunnels are up, meaning our configuration should be good. However, ping between PC1 and PC2 is not working:
```
PC1 : 10.10.10.1 255.255.255.0 gateway 10.10.10.254

PC1> ping 10.20.20.1

10.20.20.1 icmp_seq=1 timeout
10.20.20.1 icmp_seq=2 timeout
10.20.20.1 icmp_seq=3 timeout
10.20.20.1 icmp_seq=4 timeout
10.20.20.1 icmp_seq=5 timeout
```
```
PC2 : 10.20.20.1 255.255.255.0 gateway 10.20.20.254

PC2> ping 10.10.10.1

10.10.10.1 icmp_seq=1 timeout
10.10.10.1 icmp_seq=2 timeout
10.10.10.1 icmp_seq=3 timeout
10.10.10.1 icmp_seq=4 timeout
10.10.10.1 icmp_seq=5 timeout
```

Routing table of R1 and R2:
```
R1#sh ip route
Gateway of last resort is 1.1.1.2 to network 0.0.0.0

     1.0.0.0/24 is subnetted, 1 subnets
C       1.1.1.0 is directly connected, FastEthernet0/0
     169.254.0.0/24 is subnetted, 1 subnets
C       169.254.220.0 is directly connected, Tunnel1
     10.0.0.0/24 is subnetted, 1 subnets
C       10.10.10.0 is directly connected, FastEthernet0/1
S*   0.0.0.0/0 [1/0] via 1.1.1.2
```

```
R2#sh ip route
Gateway of last resort is 2.2.2.2 to network 0.0.0.0

     2.0.0.0/24 is subnetted, 1 subnets
C       2.2.2.0 is directly connected, FastEthernet0/0
     169.254.0.0/24 is subnetted, 1 subnets
C       169.254.220.0 is directly connected, Tunnel1
     10.0.0.0/24 is subnetted, 1 subnets
C       10.20.20.0 is directly connected, FastEthernet0/1
S*   0.0.0.0/0 [1/0] via 2.2.2.2
```

This shows us the connected address spaces as well as a default route towards the internet. But no routing entries for the LAN address spaces. Meaning that traffic destined for the LAN also goes through the internet (or attempts to). To confirm:
```
R1#traceroute 10.20.20.1

Type escape sequence to abort.
Tracing the route to 10.20.20.1

  1 1.1.1.2 8 msec 36 msec 8 msec
  2 1.1.1.2 !H  !H  !H 
```

```
R2#traceroute 10.10.10.1

Type escape sequence to abort.
Tracing the route to 10.10.10.1

  1 2.2.2.2 16 msec 8 msec 12 msec
  2 2.2.2.2 !H  !H  !H 
```

On both sides, traffic is sent directly to the internet. With a route-based VPN, we have two methods of sending traffic through our VPN-interface:

Option 1 is to add a static route:
```
R1(config)#ip route 10.20.20.0 255.255.255.0 tunnel 1
R1(config)#do sh ip route
Gateway of last resort is 1.1.1.2 to network 0.0.0.0

     1.0.0.0/24 is subnetted, 1 subnets
C       1.1.1.0 is directly connected, FastEthernet0/0
     169.254.0.0/24 is subnetted, 1 subnets
C       169.254.220.0 is directly connected, Tunnel1
     10.0.0.0/24 is subnetted, 2 subnets
S       10.20.20.0 is directly connected, Tunnel1
C       10.10.10.0 is directly connected, FastEthernet0/1
S*   0.0.0.0/0 [1/0] via 1.1.1.2
```

```
R2(config)#ip route 10.10.10.0 255.255.255.0 tunnel 1
R2(config)#do sh ip route
Gateway of last resort is 2.2.2.2 to network 0.0.0.0

     2.0.0.0/24 is subnetted, 1 subnets
C       2.2.2.0 is directly connected, FastEthernet0/0
     169.254.0.0/24 is subnetted, 1 subnets
C       169.254.220.0 is directly connected, Tunnel1
     10.0.0.0/24 is subnetted, 2 subnets
C       10.20.20.0 is directly connected, FastEthernet0/1
S       10.10.10.0 is directly connected, Tunnel1
S*   0.0.0.0/0 [1/0] via 2.2.2.2
```

On both routers, we added a static entry pointing the destination prefix to the interface tunnel. These entries now show up in the routing table. When we try to ping again:
```
PC1> ping 10.20.20.1

84 bytes from 10.20.20.1 icmp_seq=1 ttl=62 time=64.741 ms
84 bytes from 10.20.20.1 icmp_seq=2 ttl=62 time=61.632 ms
84 bytes from 10.20.20.1 icmp_seq=3 ttl=62 time=63.490 ms
84 bytes from 10.20.20.1 icmp_seq=4 ttl=62 time=74.101 ms
84 bytes from 10.20.20.1 icmp_seq=5 ttl=62 time=56.547 ms
```
```
PC2> ping 10.10.10.1

84 bytes from 10.10.10.1 icmp_seq=1 ttl=62 time=61.872 ms
84 bytes from 10.10.10.1 icmp_seq=2 ttl=62 time=54.334 ms
84 bytes from 10.10.10.1 icmp_seq=3 ttl=62 time=52.099 ms
84 bytes from 10.10.10.1 icmp_seq=4 ttl=62 time=70.722 ms
84 bytes from 10.10.10.1 icmp_seq=5 ttl=62 time=52.876 ms
```

We can also confirm packets being encrypted and decrypted:
```
R1#show crypto ipsec sa

interface: Tunnel1
    Crypto map tag: Tunnel1-head-0, local addr 1.1.1.1

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   current_peer 2.2.2.1 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 12, #pkts encrypt: 12, #pkts digest: 12
    #pkts decaps: 17, #pkts decrypt: 17, #pkts verify: 17
```

## Dynamic Route-Based VPN
Option 2 is to use a dynamic routing protocol, one of the benefits of a route-based VPN compared to a policy-based VPN. 
We will remove the static routing entries we introduced with Option 1 and try pinging again:
```
R1(config)#no ip route 10.20.20.0 255.255.255.0 Tunnel1
R1(config)#do sh ip route

Gateway of last resort is 1.1.1.2 to network 0.0.0.0

     1.0.0.0/24 is subnetted, 1 subnets
C       1.1.1.0 is directly connected, FastEthernet0/0
     169.254.0.0/24 is subnetted, 1 subnets
C       169.254.220.0 is directly connected, Tunnel1
     10.0.0.0/24 is subnetted, 1 subnets
C       10.10.10.0 is directly connected, FastEthernet0/1
S*   0.0.0.0/0 [1/0] via 1.1.1.2
```
```
PC1> ping 10.20.20.1

10.20.20.1 icmp_seq=1 timeout
10.20.20.1 icmp_seq=2 timeout
10.20.20.1 icmp_seq=3 timeout
10.20.20.1 icmp_seq=4 timeout
10.20.20.1 icmp_seq=5 timeout
```

Next step is to use a dynamic routing policy, for this demo we will be using BGP. Keep in mind that this is for demo purposes only and the configuration of BGP is kept to the bare minimum. With BGP both routers talk to each other and advertise networks that they know about. 

```
R1#sh run | sect router bgp
router bgp 65501
 no synchronization
 bgp log-neighbor-changes
 network 10.10.10.0 mask 255.255.255.0
 neighbor 169.254.220.2 remote-as 65502
 no auto-summary

R1#show ip bgp
BGP table version is 3, local router ID is 169.254.220.1
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.10.10.0/24    0.0.0.0                  0         32768 i
*> 10.20.20.0/24    169.254.220.2            0             0 65502 i
```

```
R2#sh run | sect router bgp
router bgp 65502
 no synchronization
 bgp log-neighbor-changes
 network 10.20.20.0 mask 255.255.255.0
 neighbor 169.254.220.1 remote-as 65501
 no auto-summary

R2#show ip bgp
BGP table version is 4, local router ID is 169.254.220.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.10.10.0/24    169.254.220.1            0             0 65501 i
*> 10.20.20.0/24    0.0.0.0                  0         32768 i
```

The key area to focus on however is the routing table, the BGP-output is great but could be a bit much if one is not familiar with BGP. 

```
R1#show ip route
B       10.20.20.0 [20/0] via 169.254.220.2, 00:06:47

R2#show ip route
B       10.10.10.0 [20/0] via 169.254.220.1, 00:06:52
```

Here we see that a new entry has been added on both ends, pointing towards the LAN-address space on the other side of the tunnel. The capital letter 'B' tells us that this entry has been added by BGP. Ping test to confirm:

```
PC1> ping 10.20.20.1

84 bytes from 10.20.20.1 icmp_seq=1 ttl=62 time=83.415 ms
84 bytes from 10.20.20.1 icmp_seq=2 ttl=62 time=65.851 ms
84 bytes from 10.20.20.1 icmp_seq=3 ttl=62 time=64.339 ms
84 bytes from 10.20.20.1 icmp_seq=4 ttl=62 time=73.805 ms
84 bytes from 10.20.20.1 icmp_seq=5 ttl=62 time=57.829 ms
```

## Route-Based VPN with High-Availability
This dynamic nature of the route-based VPN can be very beneficial for automatic fail-over or load-balancing traffic over VPN-tunnels. Let's change our architecture by adding a new ISP:

![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/RoutePolicyVPN%20-%20HA_Route.png)

In this scenario we are dealing with a dual-homed setup. All we need to do is add in a secondary VTI to our routers and configure the new public interface/IP. We can use the same Phase 1 and Phase 2 policies, no changes required. We do need to select a new IP-range for our VTI and apply it to the new public IP-addresses:

```
R1#
interface Tunnel2
 ip address 169.255.240.1 255.255.255.0
 tunnel source FastEthernet1/0
 tunnel destination 4.4.4.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile DemoProfile

 interface FastEthernet1/0
 ip address 3.3.3.1 255.255.255.0
 duplex auto
 speed auto
```
```
R2#
interface Tunnel2
 ip address 169.255.240.2 255.255.255.0
 tunnel source FastEthernet1/0
 tunnel destination 3.3.3.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile DemoProfile

 interface FastEthernet1/0
 ip address 4.4.4.1 255.255.255.0
 duplex auto
 speed auto
```

Do not forget to add the pre-shared VPN-key on both routers for the new public IP-addresses and a static route to the new IP-addresses:
```
On R1: 
crypto isakmp key myvpnkey address 4.4.4.1 
ip route 4.4.4.0 255.255.255.0 3.3.3.254
----------------------------------------------------------------------------------------
On R2: 
crypto isakmp key myvpnkey address 3.3.3.1
ip route 3.3.3.0 255.255.255.0 4.4.4.254
```

To confirm both of our tunnels are up and running, ping is still working and using Tunnel 1:
```
R1#sh int tunnel 2
Tunnel2 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 169.255.240.1/24
  MTU 1514 bytes, BW 9 Kbit/sec, DLY 500000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation TUNNEL, loopback not set
  Keepalive not set
  Tunnel source 3.3.3.1 (FastEthernet1/0), destination 4.4.4.1
  Tunnel protocol/transport IPSEC/IP
----------------------------------------------------------------------------------------
R1#sh ip int br
Interface                  IP-Address      OK? Method Status                Protocol
FastEthernet0/0            1.1.1.1         YES NVRAM  up                    up
FastEthernet0/1            10.10.10.254    YES NVRAM  up                    up
FastEthernet1/0            3.3.3.1         YES manual up                    up
Tunnel1                    169.254.220.1   YES NVRAM  up                    up
Tunnel2                    169.255.240.1   YES NVRAM  up                    up 
----------------------------------------------------------------------------------------
R1#sh ip route
Gateway of last resort is 1.1.1.2 to network 0.0.0.0

B       10.20.20.0 [20/0] via 169.254.220.2, 00:45:12
----------------------------------------------------------------------------------------
PC1> ping 10.20.20.1

84 bytes from 10.20.20.1 icmp_seq=1 ttl=62 time=44.769 ms
84 bytes from 10.20.20.1 icmp_seq=2 ttl=62 time=46.096 ms
84 bytes from 10.20.20.1 icmp_seq=3 ttl=62 time=69.090 ms
84 bytes from 10.20.20.1 icmp_seq=4 ttl=62 time=56.378 ms
84 bytes from 10.20.20.1 icmp_seq=5 ttl=62 time=39.281 ms
```

The next step is to load-balance over the VPN-tunnels. We start by building a BGP-session on our new Virtual Tunnel Interface:
```
R1(config)#router bgp 65501
R1(config-router)#neighbor 169.255.240.2 remote-as 65502
```

```
R2(config)#router bgp 65502
R2(config-router)#neighbor 169.255.240.1 remote-as 65501
```

```
R1#show ip bgp summary
BGP router identifier 169.255.240.1, local AS number 65501
BGP table version is 3, main routing table version 3
2 network entries using 234 bytes of memory
3 path entries using 156 bytes of memory
3/2 BGP path/bestpath attribute entries using 372 bytes of memory
1 BGP AS-PATH entries using 24 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 786 total bytes of memory
BGP activity 2/0 prefixes, 3/0 paths, scan interval 60 secs

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
169.254.220.2   4 65502      54      54        3    0    0 00:50:31        1
169.255.240.2   4 65502       6       6        3    0    0 00:00:07        1
```

We can confirm the new BGP-peering. At this point all traffic is still going over Tunnel 1, despite our routers knowing another way to get to the other side:
```
R1#sh ip route
B       10.20.20.0 [20/0] via 169.254.220.2, 00:50:08
----------------------------------------------------------------------------------------
R1#sh ip bgp
*  10.20.20.0/24    169.255.240.2            0             0 65502 i
*>                  169.254.220.2            0             0 65502 i
```
The '>' symbol tells us the preferred route, only this entry is active in our routing table. By modifying our BGP-configuration, we can enable load-balancing for our VPN-traffic:
```
R1(config)#router bgp 65501
R1(config-router)#maximum-paths 2  
```
For the purposes of this guide, all we need to know is that by enabling this feature, BGP will install both routes into the routing table:
```
R1#sh ip route
B       10.20.20.0 [20/0] via 169.255.240.2, 00:09:12
                   [20/0] via 169.254.220.2, 01:08:14
```

Running a trace on both outgoing interfaces, we see that our VPN-traffic is being sent out on both interfaces:

![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/VPN_Capture.png)

Keep in mind that route-based VPN give us the capability to do load-balancing, the implementation of your routing (dynamic/static, multiple interfaces) determines whether your VPN-connection will be highly available. In our demo, it is the BGP multi-path option that installs both routes. Route-based VPN just gives us the ability to utilize such features. 

## Route-Based VPN SLA
![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/RoutePolicyVPN%20-%20HA_Route.png)

Route-Based VPN allows us to select a VPN-tunnel based on monitoring data. If one of the tunnels is experiencing latency or packet drops, we can failover to the other VPN tunnel. For the purposes of this demo, we will be sending icmp-traffic from R1 to PC2. R1 will insert a static routing entry whilst tracking our ping. When it determines our ping-traffic is failing, it will remove the route using Tunnel 1 and insert the route using Tunnel 2 

```
# Demo purposes only! Please check your routers capability when it comes to routing with SLA
 ip sla monitor 1
 type echo protocol ipIcmpEcho 10.20.20.1 source-interface FastEthernet0/1
 frequency 10
ip sla monitor schedule 1 start-time now

track 1 rtr 1 reachability

ip route 10.20.20.0 255.255.255.0 Tunnel1 track 1
ip route 10.20.20.0 255.255.255.0 Tunnel2 5
```

Check that our monitoring and routing is working as expected:
```
R1#sh run | sect ip sla
ip sla monitor 1
 type echo protocol ipIcmpEcho 10.20.20.1 source-interface FastEthernet0/1
 frequency 10
ip sla monitor schedule 1 start-time now
----------------------------------------------------------------------------------------
R1#show track
Track 1
  Response Time Reporter 1 reachability
  Reachability is Up
    1 change, last change 00:13:09
  Latest operation return code: OK
  Latest RTT (millisecs) 24
  Tracked by:
    STATIC-IP-ROUTING 0
----------------------------------------------------------------------------------------
R1#sh ip route
S       10.20.20.0 is directly connected, Tunnel1
----------------------------------------------------------------------------------------
PC1> ping 10.20.20.1

84 bytes from 10.20.20.1 icmp_seq=1 ttl=62 time=70.832 ms
84 bytes from 10.20.20.1 icmp_seq=2 ttl=62 time=72.841 ms
84 bytes from 10.20.20.1 icmp_seq=3 ttl=62 time=60.891 ms
84 bytes from 10.20.20.1 icmp_seq=4 ttl=62 time=49.035 ms
84 bytes from 10.20.20.1 icmp_seq=5 ttl=62 time=52.047 ms
```

All is working well; we can now shut down the PC2 interface so our pings will start failing. There are several ways to trigger our monitoring, this is the easiest method for the demo. Once we shut down our interface, we can see that ping is no longer working:
```
PC1> ping 10.20.20.1

10.20.20.1 icmp_seq=1 timeout
10.20.20.1 icmp_seq=2 timeout
10.20.20.1 icmp_seq=3 timeout
10.20.20.1 icmp_seq=4 timeout
10.20.20.1 icmp_seq=5 timeout
```
Check R1:
```
*Mar  1 01:04:55.779: %TRACKING-5-STATE: 1 rtr 1 reachability Up->Down
R1#show ip route
S       10.20.20.0 is directly connected, Tunnel2
----------------------------------------------------------------------------------------
R1#show track 1
Track 1
  Response Time Reporter 1 reachability
  Reachability is Down
    2 changes, last change 00:02:40
  Latest operation return code: Timeout
  Tracked by:
    STATIC-IP-ROUTING 0
```

We see a notification that our tracking went from Up->Down. Routing table now shows traffic going through Tunnel 2. Pinging from PC1 to another IP-address in Office 2:
```
PC1> ping 10.20.20.254

84 bytes from 10.20.20.254 icmp_seq=1 ttl=254 time=57.528 ms
84 bytes from 10.20.20.254 icmp_seq=2 ttl=254 time=57.402 ms
84 bytes from 10.20.20.254 icmp_seq=3 ttl=254 time=59.099 ms
84 bytes from 10.20.20.254 icmp_seq=4 ttl=254 time=56.114 ms
84 bytes from 10.20.20.254 icmp_seq=5 ttl=254 time=53.098 ms
```
We have demonstrated how to failover from a VPN-tunnel, while this demo was mainly showcasing what happens if connectivity is lost, modern routers are capable of much more. Tracking can be done by in-depth monitoring of traffic such as latency and jitter. Allowing network engineers to select the best path available based on desired metrics.  


# Route-Based VPN Azure
![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/RoutePolicyVPN%20-%20AzureVPN.png)

For this demo, a physical Cisco ISR 1100 Router is being used to configure the VPN-tunnel and emulates the on-premises LAN-components with Loopback Interfaces. Important to note that the Cisco ISR Router does not have a public IP address and is located behind a modem containing the actual IP-address. The Azure configuration will use this public IP, but the VPN-tunnel can only be initiated from our Cisco router (unless you enable some type of port forwarding). 

We won't be covering the configuration on Azure in-depth, please refer to the official Microsoft documentation for insight into configuring the VPN Gateway and Local Network Gateway. 

![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/DemoVPNGateway.png)
![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/LocalNetworkGateway.png)
![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/VPNConnectionObject.png)

The connection object provides a configuration file. Configuration of the Cisco ISR looks very similar to the previous demos, with some minor tweaks:
```
# IKEv2 PROPOSAL
crypto ikev2 proposal azure-proposal-Home_VPN
 encryption aes-cbc-256 aes-cbc-128
 integrity sha1 sha256
 group 2
----------------------------------------------------------------------------------------
# IKEv2 POLICY
crypto ikev2 policy azure-policy-Home_VPN
 proposal azure-proposal-Home_VPN
----------------------------------------------------------------------------------------
# IKEv2 KEYRING (PRE-SHARED KEY)
crypto ikev2 keyring azure-keyring-Home_VPN
 peer 20.4.225.214
  address 20.4.225.214
  pre-shared-key SecretVPNKeyDemo
----------------------------------------------------------------------------------------
# IKEv2 PROFILE 
crypto ikev2 profile azure-profile-Home_VPN
 match identity remote address 20.4.225.214 255.255.255.255
 authentication remote pre-share
 authentication local pre-share
 keyring local azure-keyring-Home_VPN
 lifetime 28800
 dpd 10 5 on-demand

----------------------------------------------------------------------------------------
# IPSEC TRANSFORM 
crypto ipsec transform-set azure-ipsec-proposal-set esp-aes 256 esp-sha256-hmac
 mode tunnel

crypto ipsec profile azure-ipsec-profile-Home_VPN
 set security-association lifetime kilobytes 102400000
 set transform-set azure-ipsec-proposal-set
 set ikev2-profile azure-profile-Home_VPN
----------------------------------------------------------------------------------------
# Tunnel Interface
interface Tunnel100
 ip address 169.254.21.10 255.255.255.0
 ip tcp adjust-mss 1350
 tunnel source GigabitEthernet0/0/0
 tunnel mode ipsec ipv4
 tunnel destination 20.4.225.214
 tunnel protection ipsec profile azure-ipsec-profile-Home_VPN
----------------------------------------------------------------------------------------
# BGP Configuration
router bgp 65501
 bgp log-neighbor-changes
 neighbor 169.254.21.20 remote-as 65515
 neighbor 169.254.21.20 ebgp-multihop 255
 neighbor 169.254.21.20 update-source Tunnel100
 !
 address-family ipv4
  network 10.10.10.0 mask 255.255.255.0
  network 172.16.80.0 mask 255.255.255.0
  network 172.16.90.0 mask 255.255.255.0
  neighbor 169.254.21.20 activate
 exit-address-family
```

Confirm the state of the VTI and BGP:
```
BOZK#sh int tun100
Tunnel100 is up, line protocol is up
  Hardware is Tunnel
  Internet address is 169.254.21.10/24
  MTU 9922 bytes, BW 100 Kbit/sec, DLY 50000 usec,
     reliability 255/255, txload 1/255, rxload 1/255
  Encapsulation TUNNEL, loopback not set
----------------------------------------------------------------------------------------
BOZK#sh ip bgp summary
BGP router identifier 172.16.90.1, local AS number 65501
BGP table version is 12, main routing table version 12
5 network entries using 1240 bytes of memory
5 path entries using 720 bytes of memory
2/2 BGP path/bestpath attribute entries using 576 bytes of memory
1 BGP AS-PATH entries using 24 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 2560 total bytes of memory
BGP activity 5/0 prefixes, 5/0 paths, scan interval 60 secs
5 networks peaked at 23:23:06 Dec 12 2022 Amsterd (14:03:14.313 ago).

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
169.254.21.20   4        65515    1057    1020       12    0    0 15:17:29        2
----------------------------------------------------------------------------------------
BOZK#sh ip route | incl B
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
B        10.1.0.0/16 [20/0] via 169.254.21.20, 15:18:18
B        172.16.0.0/16 [20/0] via 169.254.21.20, 14:21:54
```

And finally, test connectivity by pinging from the Loopback Interface to the VM in Azure:
```
BOZK#ping 172.16.0.4 source Loopback 10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 172.16.0.4, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 48/50/52 ms
```

When building redundancy into our VPN-tunnels, it is important to identify single points of failure. While in our architecture the VPN Gateway has been deployed redundantly with a secondary Public IP, this is only one of the components in the connectivity chain. Make sure other components, such as the on-premises routers and even the cabling/power supply are all redundant. Ideally, the on-premises routers would be physically separated from each other. 

# Policy-Based VPN
![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/RoutePolicyVPN.png)

These types of VPN-tunnels are mostly used on legacy systems and offer limited benefit compared to the route-based VPN. It was/is mostly used in environments where route-based VPNs are no viable option. 

```
# Configuration of Phase 1
crypto isakmp policy 1
 encr aes 256
 authentication pre-share
 group 2
crypto isakmp key myvpnkey address 2.2.2.1

# Configuration of Phase 2
crypto ipsec transform-set DemoSet esp-aes 256 esp-sha-hmac

# Configuration of ACL
ip access-list extended VPN_Traffic_ACL
permit ip 10.10.10.0 0.0.0.255 10.20.20.0 0.0.0.255

# Creating a crypto map
crypto map DemoMap 1 ipsec-isakmp
 set peer 3.3.3.1
 set transform-set DemoSet
 match address VPN_Traffic_ACL

# Apply crypto map in interface

interface fastEthernet0/0
 crypto map DemoMap
```

We no longer create a virtual interface, an access-list is configured instead. It is this access-list that determines what traffic will pass over the VPN-tunnel. To visualize:

![](https://github.com/infobozk/RouteAndPolicyVPN/blob/14bde615c0bfb286f80ad98e82acc97378e15a75/images/RoutePolicyVPN%20-%20PolicyACL.png)

If both ends do not agree on what traffic is allowd into the VPN-tunnel, a mismatch occurs, and the VPN-tunnel does not complete. 

> __Note__ it is very likely that NAT is configured on your router to provide internet access. Translating internal IP-addresses into a public IP. Consider this when deploying a policy-based VPN and deny network-address translation for your VPN-traffic. For example, with our Cisco device:

```
R1(config)#ip nat inside source list 1 interface fastEthernet 0/0 overload
R1(config)#access-list 100 deny ip 10.10.10.0 0.0.0.255 10.20.20.0 0.0.0.255
R1(config)#access-list 100 permit ip 10.10.10.0 0.0.0.255 any
```
This tells our router:

Traffic from the 10.10.10.0/24 network destined for 10.20.20.0/24 should not go through NAT.
All other traffic is allowed to go through NAT. 

# Summary
The Route-Based VPN offers a lot of benefits compared to the Policy-Based VPN. 

+ High-Availability.
+ QoS and monitoring.
+ Support of dynamic routing protocols.
+ Works better with Hub-Spoke Architectures.
+ Cloud Native.
+ More scalable, limited to amount of route entries and interface tunnels on the router.
+ Feature rich, such as performing NAT over the tunnel.

Policy-Based VPN tunnels are becoming less common, however there might still be some use-cases to deploy one:

+ Supported by most (legacy) vendors.
+ Deny/Allowing Traffic is part of the VPN-configuration, might be preferred. 
+ Limited connectivity required on the address-spaces.

