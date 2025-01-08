# 2-multihomed lab

|name|router id|ASN|port/addr|advertised prefixes|
|x1|10.0.0.10|65100|<br>10.1.0.2/30@eth1</br><br>192.168.100.1/24@lo</br>|192.168.100.0/24|
|x2|10.0.0.11|65101|<br>10.1.0.6/30@eth1</br><br>192.168.101.1/24@lo</br>|192.168.101.0/24|
|rtr|10.0.0.1|65000|<br>10.1.0.1/30@eth1</br><br>10.1.0.5/30@eth2</br><br>10.0.0.1/32@lo</br>||


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
 ip address 192.168.100.1/24
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
  network 192.168.100.0/24
  neighbor 10.1.0.1 activate
  no neighbor 10.1.0.1 send-community extended
  neighbor 10.1.0.1 default-originate
 exit-address-family
exit
!
end
```

## x2 configuration
```
!
frr version 9.1_git
frr defaults datacenter
hostname x2
no ipv6 forwarding
service integrated-vtysh-config
!
interface eth1
 description x2 -> rtr [external]
 ip address 10.1.0.6/30
exit
!
interface lo
 ip address 192.168.101.1/24
exit
!
router bgp 65101
 bgp router-id 10.0.0.11
 no bgp default ipv4-unicast
 bgp bestpath as-path multipath-relax
 neighbor 10.1.0.5 remote-as 65000
 neighbor 10.1.0.5 description rtr
 !
 address-family ipv4 unicast
  network 192.168.101.0/24
  neighbor 10.1.0.5 activate
  no neighbor 10.1.0.5 send-community extended
  neighbor 10.1.0.5 default-originate
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
interface eth2
 description rtr -> x2 [external]
 ip address 10.1.0.5/30
exit
!
interface lo
 ip address 10.0.0.1/32
exit
!
router bgp 65000
 bgp router-id 10.0.0.1
 no bgp default ipv4-unicast
 bgp bestpath as-path multipath-relax
 neighbor 10.1.0.2 remote-as 65100
 neighbor 10.1.0.2 description x1
 neighbor 10.1.0.6 remote-as 65101
 neighbor 10.1.0.6 description x2
 !
 address-family ipv4 unicast
  network 10.0.0.1/32
  neighbor 10.1.0.2 activate
  neighbor 10.1.0.6 activate
  no neighbor 10.1.0.2 send-community extended
  no neighbor 10.1.0.6 send-community extended
  neighbor 10.1.0.2 default-originate
  neighbor 10.1.0.6 default-originate
 exit-address-family
exit
!
```

## output

```
rtr# show ip bgp
BGP table version is 4, local router ID is 10.0.0.1, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

    Network          Next Hop            Metric LocPrf Weight Path
 *= 0.0.0.0/0        10.1.0.6(x2)             0             0 65101 i
 *>                  10.1.0.2(x1)             0             0 65100 i
 *> 10.0.0.1/32      0.0.0.0(rtr)             0         32768 i
 *> 192.168.100.0/24 10.1.0.2(x1)             0             0 65100 i
 *> 192.168.101.0/24 10.1.0.6(x2)             0             0 65101 i

Displayed  4 routes and 5 total paths
rtr# show ip bgp sum

IPv4 Unicast Summary (VRF default):
BGP router identifier 10.0.0.1, local AS number 65000 vrf-id 0
BGP table version 4
RIB entries 5, using 480 bytes of memory
Peers 2, using 26 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
x1(10.1.0.2)    4      65100        11        11        4    0    0 00:00:13            2        4 x1
x2(10.1.0.6)    4      65101        11        11        4    0    0 00:00:13            2        4 x2

Total number of neighbors 2
rtr# show ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

B   0.0.0.0/0 [20/0] via 10.1.0.2, eth1, weight 1, 00:08:21
                     via 10.1.0.6, eth2, weight 1, 00:08:21
K>* 0.0.0.0/0 [0/0] via 192.168.121.1, eth0, 00:33:16                       // why the local route is selected, instead of the default route from x1/x2
C>* 10.0.0.1/32 is directly connected, lo, 00:32:54
C>* 10.1.0.0/30 is directly connected, eth1, 00:32:54
C>* 10.1.0.4/30 is directly connected, eth2, 00:32:54
B>* 192.168.100.0/24 [20/0] via 10.1.0.2, eth1, weight 1, 00:08:21
B>* 192.168.101.0/24 [20/0] via 10.1.0.6, eth2, weight 1, 00:08:21
C>* 192.168.121.0/24 is directly connected, eth0, 00:33:16
```

* FRR BGP route selection process prefers local routes to received routes.
    * so the default route from kernel is prefered than the default route from x1/x2

## router id vs neighbor id
* what is router id?
    * represents the **unique** identifier of a BGP router within a network
    * usually derived from the highest address on the router, often from a loopback.
    * **used in the BGP best path selection algorithm.**
    * route id IP address doesn't need to be reachable.
* what is neighbor id?
    * the address of the BGP peer router.
    * the neighbor id is used to establish the BGP session.
    * **not used in the BGP best path selection algorithm.**

## references
* https://docs.frrouting.org/en/latest/bgp.html#route-selection