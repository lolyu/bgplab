# 1-session lab

|name|ip addr|port|ASN|
|-|-|-|-|
|x1|10.1.0.2/30|eth1|65100|
|rtr|10.1.0.1|eth1|65000|

* the bgp configuration is stored in `/etc/frr/frr.conf`

## x1 configuration
```
!
frr version 9.1_git
frr defaults datacenter
hostname x1
no ipv6 forwarding
service integrated-vtysh-config
!
interface eth1
 description x1 -> rtr [external]
 ip address 10.1.0.2/30
exit
!
interface lo
 ip address 10.0.0.10/32
exit
!
router bgp 65100
 bgp router-id 10.0.0.10
 no bgp default ipv4-unicast
 bgp bestpath as-path multipath-relax
 neighbor 10.1.0.1 remote-as 65000
 neighbor 10.1.0.1 description rtr
 !
 address-family ipv4 unicast
  network 10.0.0.10/32
  neighbor 10.1.0.1 activate
  no neighbor 10.1.0.1 send-community extended
  neighbor 10.1.0.1 default-originate
 exit-address-family
exit
!
```

## rtr configuration
```
!
frr version 9.1_git
frr defaults datacenter
hostname rtr
no ipv6 forwarding
service integrated-vtysh-config
!
interface eth1
 description rtr -> x1 [external]
 ip address 10.1.0.1/30
exit
!
interface lo
 ip address 10.0.0.1/32
exit
!
router bgp 65000
 bgp router-id 10.0.0.1
 no bgp default ipv4-unicast                            # enable both ipv4 and ipv6
 bgp bestpath as-path multipath-relax                   # by default, BGP consider routes with the same AS path as candidates for multipath compuation
                                                        # this enables BGP to relax the criteria to consider routes with same AS path length as candidates
 neighbor 10.1.0.2 remote-as 65100
 neighbor 10.1.0.2 description rtr
 !
 address-family ipv4 unicast
  network 10.0.0.1/32
  neighbor 10.1.0.2 activate                            # enable announcing an address family to a specific neighbor
  no neighbor 10.1.0.2 send-community extended
  neighbor 10.1.0.2 default-originate                   # enable announcing the default route, by default, bgpd doesn't announce default routes
 exit-address-family
exit
!
```

## output
```
rtr# show ip bgp sum

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.1, local AS number 65000 vrf-id 0
BGP table version 3
RIB entries 4, using 384 bytes of memory
Peers 1, using 13 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
x1(10.1.0.2)    4      65100        12        12        3    0    0 00:00:20            2        3 rtr

Total number of neighbors 1
rtr# show ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

B   0.0.0.0/0 [20/0] via 10.1.0.2, eth1, weight 1, 00:00:23
K>* 0.0.0.0/0 [0/0] via 192.168.121.1, eth0, 00:13:03
C>* 10.0.0.1/32 is directly connected, lo, 00:12:57
B>* 10.0.0.10/32 [20/0] via 10.1.0.2, eth1, weight 1, 00:00:23
C>* 10.1.0.0/30 is directly connected, eth1, 00:12:57
C>* 192.168.121.0/24 is directly connected, eth0, 00:13:03
```