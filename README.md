# VxLAN. Аналоги VPC
## Задание:
 - Подключите клиентов 2-я линками к различным Leaf
 - Настроите агрегированный канал со стороны клиента
 - Настроите multihoming для работы в Overlay сети. Если используете Cisco NXOS - vPC, если иной вендор - то ESI LAG (либо MC-LAG с поддержкой VXLAN)
 
![alt-dtp](https://github.com/vk1391/OTUS_DC_L3VNI-MH/blob/main/L3vni%20mh.png)
- underlay - iBGP
1. Подготовка серверов
- был использован образ ubuntu-server 22.04
- установлен модуль ядра для поддержки bond интефейса(apt install ifenslave)
- отключена поддержка floppy интерфейса(не давал запустить bond интерфейс):
  - rmmod floppy
  - echo 'blacklist floppy' >> /etc/modprobe.d/blacklist.conf
  - dpkg-reconfigure initramfs-tools
  - reboot
- задана скорость на интерфейсах подключенных к коммутатору:
  - ethtool -s ens3 speed 10 duplex full
  - ethtool -s ens3 speed 10 duplex full
- применена следующая конфигурация netplan(меняется ip):
```
vi /etc/netplan/00-installer-config.yaml

network:
  ethernets:
    ens3:
      dhcp4:no
    ens4:
      dhcp4:no
  version: 2
bonds:
  bond0:
    addresses: [ 10.0.0.2/30]
    routes:
    - to: default
      via: 10.0.0.1
    interfaces: [ens3, ens4]
    parameters:
      mode: 802.3ad
      transmit-hash-policy: layer3+4
      mii-monitor-interval: 100
```
   - netplan apply
   - ifconfig bond0 up

2. Настройка Port-channel на Leaf:
- Leaf1:
```
interface Port-Channel1
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0001
      route-target import 12:34:12:34:12:34
   lacp system-id 1111.1111.1111
```
- Leaf2:
```
interface Port-Channel1
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0001
      route-target import 12:34:12:34:12:34
   lacp system-id 1111.1111.1111
interface Port-Channel2
   switchport access vlan 12
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0002
      route-target import 34:12:34:12:34:12
   lacp system-id 503a.6cdb.0bb5
```
- Leaf3:
```
interface Port-Channel1
   switchport access vlan 12
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0002
      route-target import 34:12:34:12:34:12
   lacp system-id 503a.6cdb.0bb5
```
3.Настройка L3VNI
- Leaf1:
```
service routing protocols model multi-agent
!
spanning-tree mode mstp
!
vlan 10
!
vrf instance L3VNI
!
interface Port-Channel1
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0001
      route-target import 12:34:12:34:12:34
   lacp system-id 1111.1111.1111
!
interface Ethernet1
   no switchport
   ip address 10.1.11.2/30
!
interface Ethernet2
   no switchport
   ip address 10.2.11.2/30
!
interface Ethernet3
   channel-group 1 mode active
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 11.11.11.11/32
!
interface Loopback1
   ip address 101.11.11.11/32
!
interface Management1
!
interface Vlan10
   vrf L3VNI
   ip address virtual 10.0.0.1/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 100
   vxlan vrf L3VNI vni 1000
   vxlan flood vtep 101.12.12.12 101.13.13.13
!
ip virtual-router mac-address aa:11:22:33:44:bb
!
ip routing
ip routing vrf L3VNI
!
route-map redist permit 10
   match interface Loopback0
!
route-map redist permit 20
   match interface Loopback1
!
router bgp 101
   router-id 11.11.11.11
   timers bgp 3 9
   maximum-paths 2
   neighbor 10.1.11.1 remote-as 101
   neighbor 10.1.11.1 next-hop-self
   neighbor 10.1.11.1 bfd
   neighbor 10.2.11.1 remote-as 101
   neighbor 10.2.11.1 next-hop-self
   neighbor 10.2.11.1 bfd
   neighbor 101.12.12.12 remote-as 101
   neighbor 101.12.12.12 update-source Loopback1
   neighbor 101.12.12.12 send-community extended
   neighbor 101.13.13.13 remote-as 101
   neighbor 101.13.13.13 update-source Loopback1
   neighbor 101.13.13.13 send-community extended
   redistribute connected route-map redist
   !
   vlan 10
      rd 101.13.13.13:1
      route-target both 101:1
      redistribute learned
   !
   address-family evpn
      no neighbor 10.1.11.1 activate
      no neighbor 10.2.11.1 activate
      no neighbor 101.1.1.1 activate
      no neighbor 101.2.2.2 activate
      neighbor 101.12.12.12 activate
      neighbor 101.13.13.13 activate
      no neighbor 101.22.22.22 activate
      no neighbor 101.33.33.33 activate
   !
   address-family ipv4
      neighbor 10.1.11.1 activate
      neighbor 10.2.11.1 activate
      no neighbor 101.1.1.1 activate
      no neighbor 101.2.2.2 activate
      no neighbor 101.12.12.12 activate
      no neighbor 101.13.13.13 activate
   !
   vrf L3VNI
      rd 101.11.11.11:1000
      route-target import 101:1000
      route-target import evpn 101:1000
      route-target export evpn 101:1000
      timers bgp 60 180 min-hold-time 3
      maximum-paths 1 ecmp 128
!
end
```
- Leaf2:
```
service routing protocols model multi-agent
!
spanning-tree mode mstp
!
vlan 10-12
!
vrf instance L3VNI
!
interface Port-Channel1
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0001
      route-target import 12:34:12:34:12:34
   lacp system-id 1111.1111.1111
!
interface Port-Channel2
   switchport access vlan 12
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0002
      route-target import 34:12:34:12:34:12
   lacp system-id 503a.6cdb.0bb5
!
interface Ethernet1
   no switchport
   ip address 10.1.12.2/30
!
interface Ethernet2
   no switchport
   ip address 10.2.12.2/30
!
interface Ethernet3
   channel-group 1 mode active
!
interface Ethernet4
   channel-group 2 mode active
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 12.12.12.12/32
!
interface Loopback1
   ip address 101.12.12.12/32
!
interface Management1
!
interface Vlan10
   vrf L3VNI
   ip address virtual 10.0.0.1/30
!
interface Vlan12
   vrf L3VNI
   ip address virtual 13.0.0.1/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 101
   vxlan vlan 12 vni 102
   vxlan vrf L3VNI vni 1000
   vxlan flood vtep 101.11.11.11
!
ip virtual-router mac-address aa:11:22:33:44:bb
ip virtual-router address subnet-routes
!
ip routing
ip routing vrf L3VNI
!
route-map redist permit 10
   match interface Loopback0
!
route-map redist permit 20
   match interface Loopback1
!
router bgp 101
   router-id 12.12.12.12
   timers bgp 3 9
   maximum-paths 2
   neighbor 10.1.12.1 remote-as 101
   neighbor 10.1.12.1 bfd
   neighbor 10.2.12.1 remote-as 101
   neighbor 10.2.12.1 bfd
   neighbor 101.11.11.11 remote-as 101
   neighbor 101.11.11.11 update-source Loopback1
   neighbor 101.11.11.11 send-community extended
   neighbor 101.13.13.13 remote-as 101
   neighbor 101.13.13.13 update-source Loopback1
   neighbor 101.13.13.13 send-community extended
   redistribute connected route-map redist
   !
   vlan 10
      rd 101.11.11.11:101
      route-target both 101:101
      redistribute learned
   !
   vlan 12
      rd 101.13.13.13:102
      route-target both 101:102
      redistribute learned
   !
   address-family evpn
      neighbor 101.11.11.11 activate
      neighbor 101.13.13.13 activate
   !
   address-family ipv4
      no neighbor 101.11.11.11 activate
      no neighbor 101.13.13.13 activate
   !
   vrf L3VNI
      rd 101.11.11.11:1000
      route-target import evpn 101:1000
      route-target export evpn 101:1000
      redistribute connected
!
end
```
- Leaf3:
```
service routing protocols model multi-agent
!
spanning-tree mode mstp
!
vlan 10,12
!
vrf instance L3VNI
!
interface Port-Channel2
   switchport access vlan 12
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0000:0002
      route-target import 34:12:34:12:34:12
   lacp system-id 503a.6cdb.0bb5
!
interface Ethernet1
   no switchport
   ip address 10.1.13.2/30
!
interface Ethernet2
   no switchport
   ip address 10.2.13.2/30
!
interface Ethernet3
   channel-group 2 mode active
!
interface Ethernet4
   switchport access vlan 12
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback0
   ip address 13.13.13.13/32
!
interface Loopback1
   ip address 101.13.13.13/32
!
interface Management1
!
interface Vlan11
!
interface Vlan12
   vrf L3VNI
   ip address virtual 13.0.0.1/30
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 12 vni 102
   vxlan vrf L3VNI vni 1000
   vxlan flood vtep 101.11.11.11
!
ip virtual-router mac-address aa:11:22:33:44:bb
!
ip routing
ip routing vrf L3VNI
!
route-map redist permit 10
   match interface Loopback0
!
route-map redist permit 20
   match interface Loopback1
!
router bgp 101
   router-id 13.13.13.13
   timers bgp 3 9
   maximum-paths 2
   neighbor 10.1.13.1 remote-as 101
   neighbor 10.1.13.1 bfd
   neighbor 10.2.13.1 remote-as 101
   neighbor 10.2.13.1 bfd
   neighbor 101.11.11.11 remote-as 101
   neighbor 101.11.11.11 update-source Loopback1
   neighbor 101.11.11.11 send-community extended
   neighbor 101.12.12.12 remote-as 101
   neighbor 101.12.12.12 update-source Loopback1
   neighbor 101.12.12.12 send-community extended
   redistribute connected route-map redist
   !
   vlan 12
      rd 101.11.11.11:102
      route-target both 101:102
      redistribute learned
   !
   address-family evpn
      neighbor 101.11.11.11 activate
      neighbor 101.12.12.12 activate
   !
   address-family ipv4
      no neighbor 101.11.11.11 activate
      no neighbor 101.12.12.12 activate
   !
   vrf L3VNI
      rd 101.13.13.13:1000
      route-target import evpn 101:1000
      route-target export evpn 101:1000
   !
```
4. Проверка:
- таблица соседства BGP EVPN leaf1:
```
localhost#sh bgp evpn sum
BGP summary information for VRF default
Router identifier 11.11.11.11, local AS number 101
Neighbor Status Codes: m - Under maintenance
  Neighbor     V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  101.12.12.12 4 101           108751    108468    0    0 01:58:17 Estab   7      7
  101.13.13.13 4 101           108606    108511    0    0 02:23:56 Estab   4      4
```
- таблица соседства BGP EVPN leaf2:
```
localhost#sh bgp evpn sum
BGP summary information for VRF default
Router identifier 12.12.12.12, local AS number 101
Neighbor Status Codes: m - Under maintenance
  Neighbor     V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  101.11.11.11 4 101           310515    310919    0    0 01:59:15 Estab   4      4
  101.13.13.13 4 101             2731      2727    0    0 01:52:59 Estab   4      4
```
- таблица соседства BGP EVPN leaf3:
```
localhost#sh bgp evpn sum
BGP summary information for VRF default
Router identifier 13.13.13.13, local AS number 101
Neighbor Status Codes: m - Under maintenance
  Neighbor     V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  101.11.11.11 4 101           342699    342918    0    0 02:25:45 Estab   4      4
  101.12.12.12 4 101             2751      2759    0    0 01:53:51 Estab   7      7
```
- статус port-channel Leaf1:
```
localhost#sh port-channel detailed 
Port Channel Port-Channel1 (Fallback State: Unconfigured):
Minimum links: unconfigured
Minimum speed: unconfigured
Current weight/Max weight: 1/16
  Active Ports:
       Port            Time Became Active       Protocol       Mode      Weight
    --------------- ------------------------ -------------- ------------ ------
       Ethernet3       12:11:18                 LACP           Active      1
```
- статус port-channel Leaf2:
```
localhost#SH PORT-CHANnel DETAiled
Port Channel Port-Channel1 (Fallback State: Unconfigured):
Minimum links: unconfigured
Minimum speed: unconfigured
Current weight/Max weight: 1/16
  Active Ports:
       Port            Time Became Active       Protocol       Mode      Weight
    --------------- ------------------------ -------------- ------------ ------
       Ethernet3       12:17:23                 LACP           Active      1   

Port Channel Port-Channel2 (Fallback State: Unconfigured):
Minimum links: unconfigured
Minimum speed: unconfigured
Current weight/Max weight: 1/16
  Active Ports:
       Port            Time Became Active       Protocol       Mode      Weight
    --------------- ------------------------ -------------- ------------ ------
       Ethernet4       14:09:18                 LACP           Active      1
```
- статус port-channel Leaf3:
```
localhost(config-if-Po2)#sh port-channel detailed 
Port Channel Port-Channel1 (Fallback State: Unconfigured):
Minimum links: unconfigured
Minimum speed: unconfigured
Current weight/Max weight: 0/0
  No Active Ports
Port Channel Port-Channel2 (Fallback State: Unconfigured):
Minimum links: unconfigured
Minimum speed: unconfigured
Current weight/Max weight: 1/16
  Active Ports:
       Port            Time Became Active       Protocol    Weight
    --------------- ------------------------ -------------- ------
       Ethernet3       14:08:09                 LACP          1
```
 - type 4 route bgp evpn leaf1:
```
localhost#sh bgp evpn route-type ethernet-segment 
BGP routing table information for VRF default
Router identifier 11.11.11.11, local AS number 101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 101.11.11.11:1 ethernet-segment 0000:0000:0000:0000:0001 101.11.11.11
                                 -                     -       -       0       i
 * >      RD: 101.12.12.12:1 ethernet-segment 0000:0000:0000:0000:0001 101.12.12.12
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.12.12.12:1 ethernet-segment 0000:0000:0000:0000:0002 101.12.12.12
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.13.13.13:1 ethernet-segment 0000:0000:0000:0000:0002 101.13.13.13
                                 101.13.13.13          -       100     0       i
```
 - type 4 route bgp evpn leaf2:
```
localhost#sh bgp evpn route-type ethernet-segment 
BGP routing table information for VRF default
Router identifier 12.12.12.12, local AS number 101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 101.11.11.11:1 ethernet-segment 0000:0000:0000:0000:0001 101.11.11.11
                                 101.11.11.11          -       100     0       i
 * >      RD: 101.12.12.12:1 ethernet-segment 0000:0000:0000:0000:0001 101.12.12.12
                                 -                     -       -       0       i
 * >      RD: 101.12.12.12:1 ethernet-segment 0000:0000:0000:0000:0002 101.12.12.12
                                 -                     -       -       0       i
 * >      RD: 101.13.13.13:1 ethernet-segment 0000:0000:0000:0000:0002 101.13.13.13
                                 101.13.13.13          -       100     0       i
```
 - type 4 route bgp evpn leaf3:
```
localhost#sh bgp evpn route-type ethernet-segment 
BGP routing table information for VRF default
Router identifier 13.13.13.13, local AS number 101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 101.11.11.11:1 ethernet-segment 0000:0000:0000:0000:0001 101.11.11.11
                                 101.11.11.11          -       100     0       i
 * >      RD: 101.12.12.12:1 ethernet-segment 0000:0000:0000:0000:0001 101.12.12.12
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.12.12.12:1 ethernet-segment 0000:0000:0000:0000:0002 101.12.12.12
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.13.13.13:1 ethernet-segment 0000:0000:0000:0000:0002 101.13.13.13
                                 -                     -       -       0       i
```
 - type 1 route bgp evpn leaf1:
```
localhost#sh bgp evpn route-type auto-discovery 
BGP routing table information for VRF default
Router identifier 11.11.11.11, local AS number 101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 101.11.11.11:101 auto-discovery 0 0000:0000:0000:0000:0001
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.13.13.13:1 auto-discovery 0 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >      RD: 101.11.11.11:1 auto-discovery 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >      RD: 101.12.12.12:1 auto-discovery 0000:0000:0000:0000:0001
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.11.11.11:102 auto-discovery 0 0000:0000:0000:0000:0002
                                 101.13.13.13          -       100     0       i
 * >      RD: 101.13.13.13:102 auto-discovery 0 0000:0000:0000:0000:0002
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.12.12.12:1 auto-discovery 0000:0000:0000:0000:0002
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.13.13.13:1 auto-discovery 0000:0000:0000:0000:0002
                                 101.13.13.13          -       100     0       i
```
 - type 1 route bgp evpn leaf2:
```
localhost#sh bgp evpn route-type auto-discovery 
BGP routing table information for VRF default
Router identifier 12.12.12.12, local AS number 101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 101.11.11.11:101 auto-discovery 0 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >      RD: 101.13.13.13:1 auto-discovery 0 0000:0000:0000:0000:0001
                                 101.11.11.11          -       100     0       i
 * >      RD: 101.11.11.11:1 auto-discovery 0000:0000:0000:0000:0001
                                 101.11.11.11          -       100     0       i
 * >      RD: 101.12.12.12:1 auto-discovery 0000:0000:0000:0000:0001
                                 -                     -       -       0       i
 * >      RD: 101.11.11.11:102 auto-discovery 0 0000:0000:0000:0000:0002
                                 101.13.13.13          -       100     0       i
 * >      RD: 101.13.13.13:102 auto-discovery 0 0000:0000:0000:0000:0002
                                 -                     -       -       0       i
 * >      RD: 101.12.12.12:1 auto-discovery 0000:0000:0000:0000:0002
                                 -                     -       -       0       i
 * >      RD: 101.13.13.13:1 auto-discovery 0000:0000:0000:0000:0002
                                 101.13.13.13          -       100     0       i
```
 - type 1 route bgp evpn leaf3:
```
localhost#sh bgp evpn route-type auto-discovery
BGP routing table information for VRF default
Router identifier 13.13.13.13, local AS number 101
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 101.11.11.11:101 auto-discovery 0 0000:0000:0000:0000:0001
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.13.13.13:1 auto-discovery 0 0000:0000:0000:0000:0001
                                 101.11.11.11          -       100     0       i
 * >      RD: 101.11.11.11:1 auto-discovery 0000:0000:0000:0000:0001
                                 101.11.11.11          -       100     0       i
 * >      RD: 101.12.12.12:1 auto-discovery 0000:0000:0000:0000:0001
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.11.11.11:102 auto-discovery 0 0000:0000:0000:0000:0002
                                 -                     -       -       0       i
 * >      RD: 101.13.13.13:102 auto-discovery 0 0000:0000:0000:0000:0002
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.12.12.12:1 auto-discovery 0000:0000:0000:0000:0002
                                 101.12.12.12          -       100     0       i
 * >      RD: 101.13.13.13:1 auto-discovery 0000:0000:0000:0000:0002
                                 -                     -       -       0       i
   ```
 - Пинг с серверов идёт, при отключении одного из агрегированных линков теряется некоторое кол-во пакетов,всегда разное, грешу на недостаток оперативной памяти на ESXI.

