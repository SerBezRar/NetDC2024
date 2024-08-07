# Домашнее задание 6 (Урок 12)

Для выполнения ДЗ необходимо
- составить план выполнения настройки
- создать конфигурации
- произвести проверку на лабораторном стенде

Выполнение задания сконцентрировано на настройке L3VPN VXLAN, полагаем что ip адресация и интерфейсы уже настроены в ДЗ1 (Урок 3).
Также полагаем Spine, Leaf настроенными на L2VPN из ДЗ5 (Урок 11).
Выбран OSPF Underlay из ДЗ2 (Урок 5).
Выбран eBGP EVPN Overlay ДЗ4 (Урок 8).

Для выполнения ДЗ выбран Symmetric IRB.

## 1. План настройки:
### 1.1. настройка на Leaf локальных параметров - vlan, interface для подключения конечных клиентских машин
### 1.2. настройка на Leaf локальных параметров - vrf,  svi в vrf, ip/mac virtual-router address
### 1.3. настройка vxlan интерфейса и привязка vrf-l3vni для sIRB
### 1.4. активация router bgp (соответственно с разными номерами AS для eBGP)
### 1.5. настройка параметров работы router bgp на Leaf, Spine (router-id, peer-group, timers, neighbor, bfd, address-family и т.д.)
### 1.6. настройка параметров работы router bgp на Leaf для vrf (rd, rt)
### 1.7. настройка локальных параметров конечных клиентских машин

Схема на основе которой производилась настройка

![](pictures/Topo.PNG)

## 2. Конфигурации, добавляемые в рамках данного ДЗ (остальное взято из ДЗ1)

Логика выбора rd, rt будет следующая:
rd - <source.leaf.ip.addr>:<vni>
rt - <tenant.number>:<vni>

=== Leaf1 10.1.0.1

```
vlan 10
!
vrf instance TENANT1
!
interface Ethernet7
   switchport mode access
   switchport access vlan 10
   no shutdown
!
interface Vlan10
   vrf TENANT1
   ip address virtual 192.168.10.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vrf TENANT1 vni 20001
!
ip virtual-router mac-address 00:00:aa:00:00:01
!
ip routing vrf TENANT1
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
   address-family evpn
      neighbor PG-SPINE activate
   !
   vrf TENANT1
      rd 10.1.0.1:20001
      route-target import evpn 1:20001
      route-target export evpn 1:20001
      redistribute connected
```

=== Leaf3 10.1.0.3

```
!
vlan 20
!
vrf instance TENANT1
!
interface Ethernet8
   switchport mode access
   switchport access vlan 20
   no shutdown
!
interface Vlan20
   vrf TENANT1
   ip address virtual 192.168.20.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vrf TENANT1 vni 20001
!
ip virtual-router mac-address 00:00:aa:00:00:03
!
ip routing vrf TENANT1
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
   address-family evpn
      neighbor PG-SPINE activate
   !
   vrf TENANT1
      rd 10.1.0.3:20001
      route-target import evpn 1:20001
      route-target export evpn 1:20001
      redistribute connected
```

=== Client1 192.168.10.1

```
ip 192.168.10.1/24 192.168.10.254
save
```

=== Client4 192.168.20.3

```
ip 192.168.20.3/24 192.168.20.254
save
```


### 3. Проверка работы

Выполняется с помощью ping между Client1 (192.168.10.1) - Client4 (192.168.20.3)
Приведены выводы Client1 (VPCS)

~~~
VPCS> ping 192.168.20.3

84 bytes from 192.168.20.3 icmp_seq=1 ttl=62 time=167.201 ms
84 bytes from 192.168.20.3 icmp_seq=2 ttl=62 time=34.171 ms
84 bytes from 192.168.20.3 icmp_seq=3 ttl=62 time=33.956 ms
84 bytes from 192.168.20.3 icmp_seq=4 ttl=62 time=33.350 ms

VPCS> arp

00:00:aa:00:00:01  192.168.10.254 expires in 58 seconds


leaf1#
leaf1#sh mac address-table vlan 10
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0000.aa00.0001    STATIC      Cpu
  10    0050.7966.6806    DYNAMIC     Et7        1       0:02:23 ago
Total Mac Addresses for this criterion: 2

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0


leaf1#sh ip ro vrf TENANT1

VRF: TENANT1
Codes: C - connected, S - static, K - kernel,
       O - OSPF, IA - OSPF inter area, E1 - OSPF external type 1,
       E2 - OSPF external type 2, N1 - OSPF NSSA external type 1,
       N2 - OSPF NSSA external type2, B - Other BGP Routes,
       B I - iBGP, B E - eBGP, R - RIP, I L1 - IS-IS level 1,
       I L2 - IS-IS level 2, O3 - OSPFv3, A B - BGP Aggregate,
       A O - OSPF Summary, NG - Nexthop Group Static Route,
       V - VXLAN Control Service, M - Martian,
       DH - DHCP client installed default route,
       DP - Dynamic Policy Route, L - VRF Leaked,
       G  - gRIBI, RC - Route Cache Route

Gateway of last resort is not set

 C        192.168.10.0/24 is directly connected, Vlan10
 B E      192.168.20.0/24 [200/0] via VTEP 10.0.0.3 VNI 20001 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1


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
 * >      RD: 10.1.0.1:10 mac-ip 0050.7966.6806 192.168.10.1
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.1:10 imet 10.0.0.1
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.2:10 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.3:10 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65003 i
 * >      RD: 10.1.0.1:20001 ip-prefix 192.168.10.0/24
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.3:20001 ip-prefix 192.168.20.0/24
                                 10.0.0.3              -       100     0       65003 i


~~~

Замечу что тут вывод содержит "лишние" mac-ip маршруты, так как на Leaf1 осталась L2VPN VNI конфигурация для vlan10, которой нет для vlan20 (там для проверки оставил ТОЛЬКО L3VPN VNI). Сделано специально для наглядности отличия vlan10, vlan20...
