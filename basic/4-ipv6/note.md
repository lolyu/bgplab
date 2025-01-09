# 4-ipv6 note

| Node/Interface | IPv4 Address | IPv6 Address | Description |
|----------------|-------------:|-------------:|-------------|
| **rtr** |  10.0.0.1/32 |  | Loopback |
| Ethernet1 | 10.1.0.1/30 | 2001:db8:42::1/64 | rtr -> x1 |
| Ethernet2 | 10.1.0.5/30 | 2001:db8:42:1::1/64 | rtr -> x2 |
| **x1** |  192.168.100.1/24 | 2001:db8:100:1::1/48 | Loopback |
| swp1 | 10.1.0.2/30 | 2001:db8:42::2/64 | x1 -> rtr |
| **x2** |  192.168.101.1/24 | 2001:db8:101:1::1/48 | Loopback |
| swp1 | 10.1.0.6/30 | 2001:db8:42:1::2/64 | x2 -> rtr |

## x1 configuration
```
!
frr version 9.1_git
frr defaults datacenter
hostname x1
service integrated-vtysh-config
!
interface eth1
 description x1 -> rtr [external]
 ip address 10.1.0.2/30
 ipv6 address 2001:db8:42::2/64
 ipv6 nd ra-interval 5
 no ipv6 nd suppress-ra
exit
!
interface lo
 ip address 192.168.100.1/24
 ipv6 address 2001:db8:100:1::1/48
exit
!
router bgp 65100
 bgp router-id 10.0.0.10
 no bgp default ipv4-unicast
 bgp bestpath as-path multipath-relax
 neighbor 10.1.0.1 remote-as 65000
 neighbor 10.1.0.1 description rtr
 neighbor 2001:db8:42::1 remote-as 65000
 neighbor 2001:db8:42::1 description rtr
 !
 address-family ipv4 unicast
  network 192.168.100.0/24
  neighbor 10.1.0.1 activate
  no neighbor 10.1.0.1 send-community extended
  neighbor 10.1.0.1 default-originate
 exit-address-family
 !
 address-family ipv6 unicast
  network 2001:db8:100::/48
  neighbor 2001:db8:42::1 activate
  no neighbor 2001:db8:42::1 send-community extended
  neighbor 2001:db8:42::1 default-originate
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
service integrated-vtysh-config
!
interface eth1
 description x2 -> rtr [external]
 ip address 10.1.0.6/30
 ipv6 address 2001:db8:42:1::2/64
 ipv6 nd ra-interval 5
 no ipv6 nd suppress-ra
exit
!
interface lo
 ip address 192.168.101.1/24
 ipv6 address 2001:db8:101:1::1/48
exit
!
router bgp 65101
 bgp router-id 10.0.0.11
 no bgp default ipv4-unicast
 bgp bestpath as-path multipath-relax
 neighbor 10.1.0.5 remote-as 65000
 neighbor 10.1.0.5 description rtr
 neighbor 2001:db8:42:1::1 remote-as 65000
 neighbor 2001:db8:42:1::1 description rtr
 !
 address-family ipv4 unicast
  network 192.168.101.0/24
  neighbor 10.1.0.5 activate
  no neighbor 10.1.0.5 send-community extended
  neighbor 10.1.0.5 default-originate
 exit-address-family
 !
 address-family ipv6 unicast
  network 2001:db8:101::/48
  neighbor 2001:db8:42:1::1 activate
  no neighbor 2001:db8:42:1::1 send-community extended
  neighbor 2001:db8:42:1::1 default-originate
 exit-address-family
exit
!
```

## rtr configuration
```
rtr(bash)#ifconfig lo 2001:db8:1::/48
```

```
!
frr version 9.1_git
frr defaults datacenter
hostname rtr
service integrated-vtysh-config
!
ip route 192.168.42.0/24 Null0
!
interface eth1
 description rtr -> x1 [external]
 ip address 10.1.0.1/30
 ipv6 address 2001:db8:42::1/64
 ipv6 nd ra-interval 5
 no ipv6 nd suppress-ra
exit
!
interface eth2
 description rtr -> x2 [external]
 ip address 10.1.0.5/30
 ipv6 address 2001:db8:42:1::1/64
 ipv6 nd ra-interval 5
 no ipv6 nd suppress-ra
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
 neighbor 2001:db8:42::2 remote-as 65100
 neighbor 2001:db8:42::2 description x1
 neighbor 2001:db8:42:1::2 remote-as 65101
 neighbor 2001:db8:42:1::2 description x2
 !
 address-family ipv4 unicast
  network 10.0.0.1/32
  network 192.168.42.0/24
  neighbor 10.1.0.2 activate
  no neighbor 10.1.0.2 send-community extended
  neighbor 10.1.0.6 activate
  no neighbor 10.1.0.6 send-community extended
 exit-address-family
 address-family ipv6 unicast
  network 2001:db8:1::/48
  neighbor 2001:db8:42::2 activate
  no neighbor 2001:db8:42::2 send-community extended
  neighbor 2001:db8:42:1::2 activate
  no neighbor 2001:db8:42:1::2 send-community extended
 exit-address-family
exit
!
```

# **NOTE**:
    * to allow bgpd to announce local address:
        1. the local address must be reachable
        2. the local address must be defined in the `address-family` block
    * the `neighbor *.*.*.* activate` does two things:
        1. establish bgp session with the neighbor
        2. advertise the defined networks

## references