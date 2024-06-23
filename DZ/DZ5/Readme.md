# Домашнее задание 5 (Урок 11)

Для выполнения ДЗ необходимо
- составить план выполнения настройки
- создать конфигурации
- произвести проверку на лабораторном стенде

Выполнение задания сконцентрировано на настройке L2VPN VXLAN, полагаем что ip адресация и интерфейсы уже настроены в ДЗ1 (Урок 3).
Выбран OSPF Underlay из ДЗ2 (Урок 5).
Выбран eBGP EVPN Overlay (Урок 8).

## 1. План настройки:
### 1.1. активация vxlan интерфейса и привязка vlan-vni
### 1.2. активация router bgp (соответственно с разными номерами AS для eBGP)
### 1.3. настройка параметров работы router bgp на Leaf, Spine (router-id, peer-group, timers, neighbor, bfd, address-family и т.д.)
### 1.4. настройка параметров работы router bgp на Leaf для MACVRF (vlan, rd, rt)
### 1.5. настройка на Leaf локальных параметров - vlan, interface для подключения конечных клиентских машин
### 1.6. настройка локальных параметров конечных клиентских машин

Схема на основе которой производилась настройка

![](pictures/Topo.PNG)

## 2. Конфигурации, добавляемые в рамках данного ДЗ (остальное взято из ДЗ1)

=== Leaf1 10.1.0.1

```
vlan 10
!
interface Ethernet7
   switchport mode access
   switchport access vlan 10
   no shutdown
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
router bgp 65001
   router-id 10.1.0.1
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-SPINE peer group
   neighbor PG-SPINE remote-as 65000
   neighbor PG-SPINE update-source 10.1.0.1
   neighbor PG-SPINE bfd
   neighbor PG-SPINE ebgp-multihop
   neighbor PG-SPINE send-community extended
   neighbor 10.1.1.1 peer group PG-SPINE
   neighbor 10.1.1.2 peer group PG-SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   address-family evpn
      neighbor PG-SPINE activate
```

=== Leaf2 10.1.0.2

```
vlan 10
!
interface Ethernet7
   switchport mode access
   switchport access vlan 10
   no shutdown
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
router bgp 65002
   router-id 10.1.0.2
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-SPINE peer group
   neighbor PG-SPINE remote-as 65000
   neighbor PG-SPINE update-source 10.1.0.2
   neighbor PG-SPINE bfd
   neighbor PG-SPINE ebgp-multihop
   neighbor PG-SPINE send-community extended
   neighbor 10.1.1.1 peer group PG-SPINE
   neighbor 10.1.1.2 peer group PG-SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   address-family evpn
      neighbor PG-SPINE activate
```

=== Leaf3 10.1.0.3

```
vlan 10
!
interface Ethernet7
   switchport mode access
   switchport access vlan 10
   no shutdown
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
router bgp 65003
   router-id 10.1.0.3
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-SPINE peer group
   neighbor PG-SPINE remote-as 65000
   neighbor PG-SPINE update-source 10.1.0.3
   neighbor PG-SPINE bfd
   neighbor PG-SPINE ebgp-multihop
   neighbor PG-SPINE send-community extended
   neighbor 10.1.1.1 peer group PG-SPINE
   neighbor 10.1.1.2 peer group PG-SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   address-family evpn
      neighbor PG-SPINE activate
```

=== Spine1 10.1.1.1

```
router bgp 65000
   router-id 10.1.1.1
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-LEAF peer group
   neighbor PG-LEAF update-source 10.1.1.1
   neighbor PG-LEAF bfd
   neighbor PG-LEAF ebgp-multihop
   neighbor PG-LEAF send-community extended
   neighbor 10.1.0.1 peer group PG-LEAF
   neighbor 10.1.0.1 remote-as 65001
   neighbor 10.1.0.2 peer group PG-LEAF
   neighbor 10.1.0.2 remote-as 65002
   neighbor 10.1.0.3 peer group PG-LEAF
   neighbor 10.1.0.3 remote-as 65003
   !
   address-family evpn
      neighbor PG-LEAF activate
```

=== Spine2 10.1.1.2

```
router bgp 65000
   router-id 10.1.1.2
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-LEAF peer group
   neighbor PG-LEAF update-source 10.1.1.2
   neighbor PG-LEAF bfd
   neighbor PG-LEAF ebgp-multihop
   neighbor PG-LEAF send-community extended
   neighbor 10.1.0.1 peer group PG-LEAF
   neighbor 10.1.0.1 remote-as 65001
   neighbor 10.1.0.2 peer group PG-LEAF
   neighbor 10.1.0.2 remote-as 65002
   neighbor 10.1.0.3 peer group PG-LEAF
   neighbor 10.1.0.3 remote-as 65003
   !
   address-family evpn
      neighbor PG-LEAF activate
```
=== Client1 192.168.10.1

```
ip 192.168.10.1/24
save
```

=== Client2 192.168.10.2

```
ip 192.168.10.2/24
save
```


### 3. Проверка работы

Выполняется с помощью ping между Client1 (192.168.1.1) - Client2 (192.168.1.2)

~~~
VPCS> ping 192.168.10.2

84 bytes from 192.168.10.2 icmp_seq=1 ttl=64 time=25.942 ms
84 bytes from 192.168.10.2 icmp_seq=2 ttl=64 time=31.108 ms
84 bytes from 192.168.10.2 icmp_seq=3 ttl=64 time=32.178 ms
84 bytes from 192.168.10.2 icmp_seq=4 ttl=64 time=45.014 ms
84 bytes from 192.168.10.2 icmp_seq=5 ttl=64 time=38.786 ms

VPCS> arp

00:50:79:66:68:07  192.168.10.2 expires in 69 seconds


leaf1#sh mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0050.7966.6806    DYNAMIC     Et7        1       0:10:20 ago
  10    0050.7966.6807    DYNAMIC     Vx1        1       0:00:02 ago
Total Mac Addresses for this criterion: 2

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0


leaf1#sh bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:10 mac-ip 0050.7966.6806
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:10 mac-ip 0050.7966.6807
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:10 mac-ip 0050.7966.6807
                                 10.0.0.2              -       100     0       65000 65002 i


leaf1#sh bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.1.0.1, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:10 imet 10.0.0.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.1.0.2:10 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:10 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.3:10 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:10 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65000 65003 i


spine1#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:10 mac-ip 0050.7966.6806
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.2:10 mac-ip 0050.7966.6807
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.1:10 imet 10.0.0.1
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.2:10 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.3:10 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65003 i


~~~
