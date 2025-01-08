# 3-originate lab

| Node/Interface | IPv4 Address | IPv6 Address | Description |ASN|router id|
|----------------|-------------:|-------------:|-------------|-|-|
| **rtr** |  10.0.0.1/32 |  | Loopback |65000|10.0.0.1/32|
| Ethernet1 | 10.1.0.1/30 |  | rtr -> x1 |||
| Ethernet2 | 10.1.0.5/30 |  | rtr -> x2 |||
| **x1** |  192.168.100.1/24 |  | Loopback |65100|10.0.0.10|
| eth1 | 10.1.0.2/30 |  | x1 -> rtr |||
| **x2** |  192.168.101.1/24 |  | Loopback |65101|10.0.0.11|
| eth1 | 10.1.0.6/30 |  | x2 -> rtr |||


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
* add a dummy `Loopback0` with IP `
```
rtr(bash)#ip link add Loopback0 type dummy
rtr(bash)#ifconfig Loopback0 up
rtr(bash)#ip addr add 192.168.42.1/24 dev Loopback0
```
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
  network 192.168.42.0/24
  neighbor 10.1.0.2 activate
  no neighbor 10.1.0.2 send-community extended
  neighbor 10.1.0.6 activate
  no neighbor 10.1.0.6 send-community extended
 exit-address-family
exit
!
```

## output

```
rtr# show ip bgp
BGP table version is 13, local router ID is 10.0.0.1, vrf id 0
Default local pref 100, local AS 65000
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

    Network          Next Hop            Metric LocPrf Weight Path
 *= 0.0.0.0/0        10.1.0.2(x1)             0             0 65100 i
 *>                  10.1.0.6(x2)             0             0 65101 i
 *> 192.168.42.0/24  0.0.0.0(rtr)             0         32768 i
 *> 192.168.100.0/24 10.1.0.2(x1)             0             0 65100 i
 *> 192.168.101.0/24 10.1.0.6(x2)             0             0 65101 i

Displayed  4 routes and 5 total paths
```

## reference
