# Домашнее задание 7 (Урок 14)

Для выполнения ДЗ необходимо
- составить план выполнения настройки
- создать конфигурации
- произвести проверку на лабораторном стенде

Выполнение задания сконцентрировано на настройке EVPN ESI, полагаем что ip адресация и интерфейсы уже настроены в ДЗ1 (Урок 3).
Также полагаем Spine, Leaf настроенными на L2VPN из ДЗ5 (Урок 11).
Выбран OSPF Underlay из ДЗ2 (Урок 5).
Выбран eBGP EVPN Overlay ДЗ4 (Урок 8).

Для выполнения ДЗ выбран режим ESI LAG (без MC-LAG).
В качестве Multi-Homing Client выбрана Arista.

## 1. План настройки:
### 1.1. настройка на Leaf локальных параметров - vlan, interface (включая Port-Channel) для подключения конечных клиентских машин
### 1.2. настройка на Leaf локальных параметров - interface Port-Channel параметров - evpn ethernet-segment, lacp system-id
### 1.3. настройка локальных параметров конечных клиентских машин

Схема на основе которой производилась настройка

![](pictures/Topo.PNG)

## 2. Конфигурации, добавляемые в рамках данного ДЗ (остальное взято из ДЗ1)

Логика выбора ethernet-segment будет следующая:
0000:0000:0000:<leaf-pair-number>:<port-channel-number>

Логика выбора LACP system-id будет следующая:
0000.<leaf-pair-number>.<port-channel-number>

=== Leaf1 10.1.0.1

```
interface Ethernet8
   channel-group 12 mode active
!
interface Port-Channel12
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0012:0012
      route-target import 00:00:00:00:12:12
   lacp system-id 0000.0012.0012
!
```

=== Leaf2 10.1.0.2

```
interface Ethernet8
   channel-group 12 mode active
!
interface Port-Channel12
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0012:0012
      route-target import 00:00:00:00:12:12
   lacp system-id 0000.0012.0012
!
```

=== Client1 192.168.10.1

```
ip 192.168.10.1/24 192.168.10.254
save
```

=== MHClient1 192.168.10.12

```
interface Ethernet1
   channel-group 12 mode active
!
interface Ethernet2
   channel-group 12 mode active
!
interface Port-Channel12
   no switchport
   ip address 192.168.10.12/24
```


### 3. Проверка работы

Выполняется с помощью ping между Client1 (192.168.10.1) - MHClient1 (192.168.10.12)
Приведены выводы Client1 (VPCS) и MHClient1 (Arista)

~~~
VPCS> ping 192.168.10.12

84 bytes from 192.168.10.12 icmp_seq=1 ttl=64 time=11.624 ms
84 bytes from 192.168.10.12 icmp_seq=2 ttl=64 time=16.873 ms
84 bytes from 192.168.10.12 icmp_seq=3 ttl=64 time=13.118 ms
84 bytes from 192.168.10.12 icmp_seq=4 ttl=64 time=14.451 ms
84 bytes from 192.168.10.12 icmp_seq=5 ttl=64 time=12.114 ms

VPCS> sh arp

50:00:00:af:d3:f6  192.168.10.12 expires in 112 seconds


mhclient1#sh arp
Address         Age (sec)  Hardware Addr   Interface
192.168.10.1      0:19:36  0050.7966.6806  Port-Channel12



mhclient1#sh lacp peer detailed
State: A = Active, P = Passive; S=ShortTimeout, L=LongTimeout;
       G = Aggregable, I = Individual; s+=InSync, s-=OutOfSync;
       C = Collecting (aggregating incoming frames), X = state machine expired,
       D = Distributing (aggregating outgoing frames),
       d = default neighbor state
             |          |                       Partner
Port Status  | Select   | Sys-id                 Port# State   OperKey PortPri
---- --------|----------|----------------------- ----- ------- ------- --------
Port Channel Port-Channel12:
Et1  Bundled | Selected | 8000,00-00-00-12-00-12     8 ALGs+CD  0x000c   32768
Et2  Bundled | Selected | 8000,00-00-00-12-00-12     8 ALGs+CD  0x000c   32768

                         |   Partner    Collector
   Port         Status   |   Churn       MaxDelay
---------- --------------|------------- ---------
Port Channel Port-Channel12:
   Et1          Bundled  |   noChurn            0
   Et2          Bundled  |   noChurn            0


leaf1#sh bgp evpn instance vlan 10
EVPN instance: VLAN 10
  Route distinguisher: 0:0
  Route target import: Route-Target-AS:10:10010
  Route target export: Route-Target-AS:10:10010
  Service interface: VLAN-based
  Local VXLAN IP address: 10.0.0.1
  VXLAN: enabled
  MPLS: disabled
  Local ethernet segment:
    ESI: 0000:0000:0000:0012:0012
      Interface: Port-Channel12
      Mode: all-active
      State: up
      ES-Import RT: 00:00:00:00:12:12
      DF election algorithm: modulus
      Designated forwarder: 10.0.0.1
      Non-Designated forwarder: 10.0.0.2


spine1#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.1.1.1, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.1.0.1:10 auto-discovery 0 0000:0000:0000:0012:0012
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.2:10 auto-discovery 0 0000:0000:0000:0012:0012
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.0.0.1:1 auto-discovery 0000:0000:0000:0012:0012
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.0.0.2:1 auto-discovery 0000:0000:0000:0012:0012
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.1:10 mac-ip 0050.7966.6806
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.1:10 mac-ip 0050.7966.6806 192.168.10.1
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.1:10 mac-ip 5000.00af.d3f6
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.1:10 imet 10.0.0.1
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.2:10 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.3:10 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65003 i
 * >      RD: 10.0.0.1:1 ethernet-segment 0000:0000:0000:0012:0012 10.0.0.1
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.0.0.2:1 ethernet-segment 0000:0000:0000:0012:0012 10.0.0.2
                                 10.0.0.2              -       100     0       65002 i
 * >      RD: 10.1.0.1:20001 ip-prefix 192.168.10.0/24
                                 10.0.0.1              -       100     0       65001 i
 * >      RD: 10.1.0.3:20001 ip-prefix 192.168.20.0/24
                                 10.0.0.3              -       100     0       65003 i

~~~

Замечу что тут вывод содержит "лишние" mac-ip, ip-prefix маршруты, так как остались L2VPN, L3VPN VNI конфигурации.
