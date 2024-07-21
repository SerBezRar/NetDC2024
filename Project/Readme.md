# Проектная работа на тему "Создание новой сети на базе технологий EVPN VXLAN"

## Введение

### Цель проекта

Тема проекта - "Создание новой сети на базе технологий EVPN VXLAN", выбрана в соответствии с вероятным сценарием при ответе на внутренний запрос бизнеса компании на разработку дизайна сети под предполагаемую новую локацию ЦОД.
Текущая локация насыщена по многим параметрам, которые указывают на необходимость разработки новой сети в новой локации

Стратегические прогнозы бизнеса предполагают, что сеть должна поддерживать масштабирование до 50 стоек оборудования на ближайшие 5 лет (в каждом может быть до 2х Top-of-Rack свичей для поддержки резервирования), т.е. до 100 Leaf свичей в сумме.

Соответственно цели проекта
- выбор и разработка архитектуры новой сети (топология, технологии)
- выбор и разработка дизайна сети для выбранной архитектуры (проектирования самих сетей с учётом выбора технологии, топологии, вендора, поддержки им фич)
тип проекта - Greenfield

Требования:
- поддержка масштабирования от 1 до 100 Leaf
- поддержка отказоустойчивости на уровне Spine, Leaf, отдельных линков от Leaf до хостов (Multi-Home Host)

### Задачи проекта

На основе цели проекта соответственно должны быть выполнены задачи
- выбор архитектуры сети
- разработка архитектуры сети
- разработка дизайна underlay сети
- разработка дизайна overlay сети
- разработка дизайна адресного пространства и идентификаторов

#### Выбор архитектуры сети

Требования:
быстрые и современные скорости
мультитенантность (поддержка изоляции на L2, L3)
масштабирование без изменения ключевых компонентов
резервирование ключевых компонентов

--- Сравнение Tier-3 и Clos

Здесь и далее фабрика - новая разрабатываемая в рамках проекта сеть.
Сравнение производится с текущей сетью организации построенной на Cisco vPC + MSTP

Фабрика решает вопросы:
- увеличение общего количества поддерживаемых в сети broadcast доменов более 4000 extends – примерно 16 миллионов.
- устраняет завязанность на протоколы семейства STP
- улучшает утилизацию линков при ECMP

Что выглядит лучше в фабрике в сравнении с сетью на vPC:
- позволяет БОЛЬШУЮ отказоустойчивость ключевых элементов, через которые проходит весь трафик (Spine в данном случае). В vPC может быть только 2 экземпляра устройств.
- позволяет БОЛЬШУЮ отвязанность от вендора в некоторых элементах (тех же Spine). vPC - проприетарный протокол. То что используется в фабрике в основном - открытые стандарты.
- позволяет БОЛЕЕ ПРОСТОЕ масштабирование по полосе
- позволяет ЭФФЕКТИВНЕЕ решать проблемы с петлями
- устраненяет unknown unicast
- позволяет увеличенить количества broadcast доменов

#### Разработка архитектуры сети

Архитектура сети предполагает выбор ключевых подходов и технологий

Из-за размера сети, количества линков, поддержки технологий, неактуальности постоянных ручных конфигураций, возможностей масштабирования были выбраны
Топология новой сети - 3-stage Clos (Leaf-Spine-Leaf)
Используемый Вендор для тестирования в лабораторных условиях - Arista
Используемый Overlay Data Plane - VXLAN
Используемый Overlay Control Plane - BGP EVPN

Соответственнно новая сеть будет разделена на Underlay, Overlay
Underlay - обеспечивает базовую ip-связность для передачу инкапсулированного Overlay трафика, установления BGP-сессий и т.д.
Overlay - обеспечивает передачу конечного клиентского трафика, инкапсуляцию, обмен информацией о конечных хостах.

Архитектура сети разрабатывалась на основе Design Guide выбранного вендора и лучших практик на текущий момент - Arista
соответственно архитектура сети с топологией Clos предполагает
- набор коммутаторов для подключения конечного оборудования - Leaf
- набор коммутаторов для эффективного промежуточного подключения между собой коммутаторов Leaf - Spine
- стандартные правила подключения для таких фабрик - Leaf может подключаться физически только к Spine, Spine не подключаются 

физически между собой, Leaf не подключаются физически между собой (кроме некоторых случаев парных подключений при M-LAG, но в данной работе такое не используется)
- 1 большой ip домен маршрутизации для обеспечения связности между устройствами
- работа VXLAN так как инкапсуляция полагается на IP-UDP протоколы, полагается на этот большой домен маршрутизации
- используются Lo интерфейсы для терминации VXLAN туннелей

Проект выполняется для сети Clos размером 2 spine 4 leaf (но поддерживает расширение до бОльших размеров)

#### Разработка дизайна Underlay сети

В качестве Underlay выбран OSPF с P2P линками и активированным BFD.
Прежде всего за
- достаточное масштабирование
- знание протокола инженерами
- простоту настройки и поддержки

#### Разработка дизайна Overlay сети

В качестве Overlay был выбран BGP EVPN VXLAN
Прежде всего за
- масштабирование
- переиспользование с минимальными доработками на других вендорах
- уже достаточную распространённость и отработанность VXLAN на рынке

#### Разработка дизайна адресного пространства и идентификаторов

1. underlay interface connection планировании (int plan)
2. underlay ip address and network underlay планировании (ip.addr-net plan)
3. overlay evpn vxlan instances - vni, macvrf, vrf, rd, rt, esi, virtual-mac


1. underlay interface connection планировании (int plan)
--- Нумерация подключения p2p линков для соединения Leaf-Spine, для удобства и для соответствия ip.addr-net plan формируется по 
следующим правилам:
- номер интерфейса Spine (EthN) указывает на номер Leaf<N> к которому подключен. N начинается с 1.
- номер интерфейса Leaf (EthM) указывает на номер Spine<M> к которому подключен. M начинается с 1.
т.е. номера интерфейсов выбраны соответственно удаленному оборудованию на второй стороне подключения интерфейса.

Например,
- Spine1 Eth1 - Leaf1 Eth1
- Spine1 Eth3 - Leaf3 Eth1
- Spine3 Eth2 - Leaf2 Eth3

--- Нумерация Lo интерфейсов
- номер интерфейса указывает на номер выбранного диапазона DC-SERVICE-NUMBER.
Так как и сети и Lo доступны нумерации с 0, мы будем использовать нумерацию с 0 для удобства.

2. underlay ip address and network underlay планировании (ip.addr-net plan)

2.1. Общие принципы

Выбор диапазонов производится для DC и будет учитывать:
- расширение сети с учётом нескольких возможных DC, пока 1, т.е. не будем занимать полностью всю выбранную сеть
- расширение сети с учётом нескольких возможных новых уровней, или новых сущностей, например SuperSpine, т.е. нужно оставить резервы
- возможность суммаризации адресного пространства всего DC

Используемые маски:
- сети для линков P2P - /30 (так привычнее по текущим сетям для инженеров)
- сети для лупбэков - /32

Всего в рамках Underlay и данного ДЗ нужно 4 пространства нумерации, которые будем активно использовать
- P2P интерфейсы и адреса для Routing на уровнях Spine-Leaf
- Lo интерфейсы и адреса для Routing на уровне Spine
- Lo интерфейсы и адреса для VXLAN на уровне Leaf (делаем немного заранее, предположительно нужно будет в следующих ДЗ)
- Lo интерфейсы и адреса для Routing на уровне Leaf

2.2. Общая формула адресного пространства

10.DC-SERVICE-NUMBER.SPINE-SERVICE-NUMBER.SUBNET-NUMBER
DC-SERVICE-NUMBER - Range адресов для масштабирования DC и сервисов. Номер позволит точно узнать какой DC, сервис и т.д..
SPINE-SERVICE-NUMBER - Range адресов для масштабирования и выбора идентификаторов в рамках DC-SERVICE-NUMBER. Номер позволит точно узнать какой Spine, сервис и т.д..
SUBNET-NUMBER - Range адресов для масштабирования и выбора идентификаторов в рамках SPINE-SERVICE-NUMBER. Номер уже не будет показывать числовым значением ничего, просто перебор сетей по порядку.

---
10.DC-SERVICE-NUMBER.x.x (x - dont care в данном случае)
DC-SERVICE-NUMBER разбит на диапазоны для суммаризации по 1 DC - /13, т.е. диапазоны 0-7(DC1), 8-15(DC2) и т.д.
- DC-SERVICE-NUMBER = 0 для лупбэков
- DC-SERVICE-NUMBER = 1 для P2P сетей - интерфейсов от Spine1, используемых для роутинга к Leaf. Leaf and Spine.
- DC-SERVICE-NUMBER = 2 для P2P сетей - интерфейсов от Spine2, используемых для роутинга к Leaf. Leaf and Spine.
- DC-SERVICE-NUMBER = 3 для P2P сетей - интерфейсов от Spine3, используемых для роутинга к Leaf. Leaf and Spine.
- DC-SERVICE-NUMBER = 4-7 для резервных назначений (например ещё для одних лупбэков, или расширения маршрутизации, SuperSpine).

---
10.x.SPINE-SERVICE-NUMBER.x (x - dont care в данном случае)
SPINE-SERVICE-NUMBER не разбивается на диапазоны
SPINE-SERVICE-NUMBER = 0-255 - позволяет перебирать для разных назначений номера
Для P2P сетей
- SPINE-SERVICE-NUMBER = 1 для лупбэков и линков P2P для Spinex-Leaf1
- SPINE-SERVICE-NUMBER = 2 для лупбэков и линков P2P для Spinex-Leaf2
Для Lo сетей
- SPINE-SERVICE-NUMBER = 0 для лупбэков Leaf, используемых для полезных сервисов, связанных с терминацией VXLAN туннелей. Только Leaf.
- SPINE-SERVICE-NUMBER = 1 для лупбэков Leaf, используемых для полезных сервисов, связанных с терминацией VXLAN туннелей. Только Leaf.
- SPINE-SERVICE-NUMBER = 2 для лупбэков Spine

---
10.x.x.SUBNET-NUMBER (x - dont care в данном случае)
SUBNET-NUMBER не разбивается на диапазоны
- SUBNET-NUMBER = 0-255 - позволяет перебирать для разных назначений номера Spine, Leaf
Например
DC1-Spine1-Leaf1 P2P линк - 10.2.1.0/24
DC1-Spine2-Leaf3 P2P линк - 10.2.2.8/24
DC1-Leaf2 Lo0 - 10.0.0.2
DC1-Leaf3 Lo0 - 10.0.0.3

3. overlay evpn vxlan instances - vni, macvrf, vrf, rd, rt, esi, virtual-mac

--- Логика нумерации для MACVRF не требуется - так как в Arista MACVRF создаётся по номеру vlan
Например 
vlan 10 = MACVRF10

--- Логика выбора ethernet-segment будет следующая:
0000:0000:0000:<leaf-pair-number-at-the-end>:port-channel-number
Например
0000:0000:0000:0012:0012

--- Логика выбора LACP system-id будет следующая:
0000.leaf-pair-number.port-channel-number
Например
0000.0012.0012

--- Логика выбора VNI будет следующая:
L2VNI - 1xxxx для vlan xxxx
Например
vlan 10 - vni 10010

L3VNI - 2xxxx для TENANT xxxx
Например
tenant 1 - vni 20001

--- Логика выбора rd, rt будет следующая:
rd - <source.leaf.ip.addr>:<vni>
Например
10.1.0.3:10010

rt - <tenant.number>:<vni>
Например
10:10010

## Тестовый стенд

Для тестового стенда выбрана платформа EVE-NG 6.0.1-12
Образы всех сетевых устройств - Arista vEOS-lab, EOS-4.29.2F
Для проведения настройки создана сеть следующей топологии:

<>

4 Leaf (1 из них в качестве Border Leaf)
3 Spine

На основе вышеуказанной сети можно проверить следующие технологии фабрики VXLAN BGP EVPN
- подключение хоста Client c один линком Ethernet и работу L2VNI
- подключение хоста MHClient с двумя резервируемыми линками Ethernet и работу L2VNI, ES-LAG
- маршрутизацию внутри фарбики на основе Symmetric IRB и работу L3VNI
- подключение фабрики к внешним сетевым устройствам - extrouter в качестве внешнего маршрутизатора или фаервола.

## Настройка

Приведены примеры настройки для каждого типа устройства или раздела.
Полные конфигурации доступны в разделе Итоговые конфигурации ниже.

В реальной эксплуатации в зависимости от типа оборудования могут быть добавлены конфигурации
System Profile (TCAM Resource)
System (Clock, NTP и т.д. )
Mgmt Plane (Log, SNMP и т.д.)

--- Предварительная и общая конфигурация Leaf, Spine (на примере Leaf1)

конфигурация имени устройства
```
hostname leaf1
```

конфигурация актуального Route Table Manager
```
service routing protocols model multi-agent
write
reload
```

--- Конфигурация интерфейсов для Leaf (на примере Leaf1)

конфигурация интерфейсов в сторону Spine, включаем режим L3 порта, настраиваем адресацию
```
interface Ethernet1
   no switchport
   ip address 10.1.1.2/30
!
interface Ethernet2
   no switchport
   ip address 10.2.1.2/30
!
interface Ethernet3
   no switchport
   ip address 10.3.1.2/30
```

конфигурация Loopback интерфейса для задач терминации eBGP EVPN сессий
конфигурация Loopback интерфейса для задач терминации VXLAN туннелей
```
interface Loopback0
   ip address 10.0.0.1/32
interface Loopback1
   ip address 10.1.0.1/32
```

--- Конфигурация интерфейсов для Spine (на примере Spine1)
```
interface Ethernet1
   no switchport
   ip address 10.1.1.1/30
!
interface Ethernet2
   no switchport
   ip address 10.1.2.1/30
!
interface Ethernet3
   no switchport
   ip address 10.1.3.1/30
!
interface Ethernet4
   no switchport
   ip address 10.1.4.1/30
```
--- Конфигурация маршрутизации Underlay для Leaf и Spine (на примере Leaf1)

конфигурация общая для работы Underlay маршрутизации
```
ip routing
```

конфигурация OSPF - основная для работы Underlay маршрутизации
```
router ospf 1
```

конфигурация router-id, будет соответствовать Lo1 интерфейсу
```
   router-id 10.0.1.1
```

конфигурация passive-interface по умолчанию
```
   passive-interface default
```

конфигурация passive-interface
```
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   network 0.0.0.0/0 area 0.0.0.0
   bfd default
   max-lsa 12000
```

конфигурация OSPF на интерфейсах, включаем режим point-to-point
```
interface Ethernet1
   ip ospf network point-to-point
interface Ethernet2
   ip ospf network point-to-point
interface Ethernet3
   ip ospf network point-to-point
```

--- Конфигурация Leaf для конечных подключений Overlay (на примере Leaf1)

конфигурация vlan
```
vlan 10
```

конфигурация интерфейса в сторону конечного хоста
```
interface Ethernet7
   switchport access vlan 10
```

конфигурация VXLAN интерфейса c назначением L2VNI
```
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
```

также настроим MACVRF для vlan 10
```
router bgp 65001
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
```

=== Конфигурация Leaf для конечных подключений с локальным L3 GW Overlay (на примере Leaf2)

конфигурация vlan
```
vlan 10
```

конфигурация интерфейса в сторону конечного хоста
```
interface Ethernet7
   switchport access vlan 50
```
конфигурация VRF
```
vrf instance TENANT2
```

конфигурация интерфейса VRF
```
interface Vlan50
   vrf TENANT2
   ip address virtual 10.99.50.254/24
```

конфигурация маршрутизации для VRF
```
ip routing vrf TENANT2
```

конфигурация VXLAN интерфейса c назначением L2VNI, L3VNI
```
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 50 vni 10050
   vxlan vrf TENANT2 vni 20002
```

также настроим MACVRF для vlan 50
```
router bgp 65002
   vlan 50
      rd auto
      route-target both 50:10050
      redistribute learned
```

также настроим VRF для TENANT2
```
router bgp 65002
   vrf TENANT2
      rd 10.1.0.2:20002
      route-target import evpn 2:20002
      route-target export evpn 2:20002
      redistribute connected
```

=== Конфигурация Leaf для Overlay (на примере Leaf1)

конфигурация BGP, настроим router-id, таймеры, максимальное количество путей, BFD
также настроим eBGP multihop
также настроим пересылку community
также используем Peer Group для упрощения настройки

```
router bgp 65001
   router-id 10.0.1.1
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-SPINE peer group
   neighbor PG-SPINE remote-as 65000
   neighbor PG-SPINE update-source 10.0.1.1
   neighbor PG-SPINE bfd
   neighbor PG-SPINE ebgp-multihop
   neighbor PG-SPINE send-community extended
   neighbor 10.0.2.1 peer group PG-SPINE
   neighbor 10.0.2.2 peer group PG-SPINE
   neighbor 10.0.2.3 peer group PG-SPINE
```

также настроим активацию Peer Group для EVPN AFI:SAFI
```
   address-family evpn
      neighbor PG-SPINE activate
```

=== Конфигурация Spine для Overlay (на примере Spine1)

конфигурация BGP, настроим router-id, таймеры, максимальное количество путей, BFD
также настроим eBGP multihop
также настроим пересылку community
также используем Peer Group для упрощения настройки
```
router bgp 65000
   router-id 10.0.2.1
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-LEAF peer group
   neighbor PG-LEAF next-hop-unchanged
   neighbor PG-LEAF update-source 10.0.2.1
   neighbor PG-LEAF bfd
   neighbor PG-LEAF ebgp-multihop
   neighbor PG-LEAF send-community extended
   neighbor 10.0.1.1 peer group PG-LEAF
   neighbor 10.0.1.1 remote-as 65001
   neighbor 10.0.1.2 peer group PG-LEAF
   neighbor 10.0.1.2 remote-as 65002
   neighbor 10.0.1.3 peer group PG-LEAF
   neighbor 10.0.1.3 remote-as 65003
   neighbor 10.0.1.4 peer group PG-LEAF
   neighbor 10.0.1.4 remote-as 65004
   !
   address-family evpn
      neighbor PG-LEAF activate
```

=== Конфигурация Leaf для настройки ES-LAG (на примере Leaf1)

конфигурация интерфейса для работы в PortChannel
```
interface Ethernet8
   channel-group 12 mode active
```

конфигурация PortChannel для работы в ES-LAG
```
interface Port-Channel12
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0012:0012
      route-target import 00:00:00:00:12:12
   lacp system-id 0000.0012.0012
```

=== Конфигурация со стороны клиента с одним подключением (на примере Client1)

конфигурация интерфейса для настройки ip.addr и Default Gateway
```
ip 192.168.10.1/24 192.168.10.254
```

=== Конфигурация со стороны клиента Multi-Home Client (на примере MHClient1)

конфигурация интерфейсов для работы в PortChannel
```
interface Ethernet1
   channel-group 12 mode active
!
interface Ethernet2
   channel-group 12 mode active
```

конфигурация интерфейса для настройки ip.addr
```
interface Port-Channel12
   no switchport
   ip address 192.168.10.12/24

конфигурация Default Gateway
ip route 0.0.0.0/0 192.168.10.254
```

## Итоговые конфигурации

</code></pre>
</details>

<details>
<summary>Конфигурация Leaf1</summary>
<pre><code>
! Command: show running-config
! device: leaf1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname leaf1
!
spanning-tree mode mstp
!
vlan 10
!
vrf instance TENANT1
!
interface Port-Channel12
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0012:0012
      route-target import 00:00:00:00:12:12
   lacp system-id 0000.0012.0012
!
interface Ethernet1
   no switchport
   ip address 10.1.1.2/30
   ip ospf network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.2.1.2/30
   ip ospf network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.3.1.2/30
   ip ospf network point-to-point
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   switchport access vlan 10
!
interface Ethernet8
   channel-group 12 mode active
!
interface Loopback0
   ip address 10.0.0.1/32
!
interface Loopback1
   ip address 10.0.1.1/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
!
ip routing
no ip routing vrf TENANT1
!
router bgp 65001
   router-id 10.0.1.1
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-SPINE peer group
   neighbor PG-SPINE remote-as 65000
   neighbor PG-SPINE update-source 10.0.1.1
   neighbor PG-SPINE bfd
   neighbor PG-SPINE ebgp-multihop
   neighbor PG-SPINE send-community extended
   neighbor 10.0.2.1 peer group PG-SPINE
   neighbor 10.0.2.2 peer group PG-SPINE
   neighbor 10.0.2.3 peer group PG-SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   address-family evpn
      neighbor PG-SPINE activate
!
router ospf 1
   router-id 10.0.1.1
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   network 0.0.0.0/0 area 0.0.0.0
   max-lsa 12000
!
end
</code></pre>
</details>

<details>
<summary>Конфигурация Leaf2</summary>
<pre><code>
! Command: show running-config
! device: leaf2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname leaf2
!
spanning-tree mode mstp
!
vlan 10,50
!
vrf instance TENANT2
!
interface Port-Channel12
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0012:0012
      route-target import 00:00:00:00:12:12
   lacp system-id 0000.0012.0012
!
interface Ethernet1
   no switchport
   ip address 10.1.2.2/30
   ip ospf network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.2.2.2/30
   ip ospf network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.3.2.2/30
   ip ospf network point-to-point
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   switchport access vlan 50
!
interface Ethernet8
   channel-group 12 mode active
!
interface Loopback0
   ip address 10.0.0.2/32
!
interface Loopback1
   ip address 10.0.1.2/32
!
interface Management1
!
interface Vlan50
   vrf TENANT2
   ip address virtual 10.99.50.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 50 vni 10050
   vxlan vrf TENANT2 vni 20002
!
ip routing
ip routing vrf TENANT2
!
router bgp 65002
   router-id 10.0.1.2
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-SPINE peer group
   neighbor PG-SPINE remote-as 65000
   neighbor PG-SPINE update-source 10.0.1.2
   neighbor PG-SPINE bfd
   neighbor PG-SPINE ebgp-multihop
   neighbor PG-SPINE send-community extended
   neighbor 10.0.2.1 peer group PG-SPINE
   neighbor 10.0.2.2 peer group PG-SPINE
   neighbor 10.0.2.3 peer group PG-SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   vlan 50
      rd auto
      route-target both 50:10050
      redistribute learned
   !
   address-family evpn
      neighbor PG-SPINE activate
   !
   vrf TENANT2
      rd 10.1.0.2:20002
      route-target import evpn 2:20002
      route-target export evpn 2:20002
      redistribute connected
!
router ospf 1
   router-id 10.0.1.2
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   network 0.0.0.0/0 area 0.0.0.0
   max-lsa 12000
!
end
</code></pre>
</details>


<details>
<summary>Конфигурация Leaf3</summary>
<pre><code>
! Command: show running-config
! device: leaf3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname leaf3
!
spanning-tree mode mstp
!
vlan 20,60
!
vrf instance TENANT2
!
interface Ethernet1
   no switchport
   ip address 10.1.3.2/30
   ip ospf network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.2.3.2/30
   ip ospf network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.3.3.2/30
   ip ospf network point-to-point
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
   switchport access vlan 20
!
interface Ethernet8
   switchport access vlan 60
!
interface Loopback0
   ip address 10.0.0.3/32
!
interface Loopback1
   ip address 10.0.1.3/32
!
interface Management1
!
interface Vlan60
   vrf TENANT2
   ip address virtual 10.99.60.254/24
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 20 vni 10020
   vxlan vlan 60 vni 10060
   vxlan vrf TENANT2 vni 20002
!
ip routing
ip routing vrf TENANT2
!
router bgp 65003
   router-id 10.0.1.3
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-SPINE peer group
   neighbor PG-SPINE remote-as 65000
   neighbor PG-SPINE update-source 10.0.1.3
   neighbor PG-SPINE bfd
   neighbor PG-SPINE ebgp-multihop
   neighbor PG-SPINE send-community extended
   neighbor 10.0.2.1 peer group PG-SPINE
   neighbor 10.0.2.2 peer group PG-SPINE
   neighbor 10.0.2.3 peer group PG-SPINE
   !
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
   !
   vlan 60
      rd auto
      route-target both 60:10060
      redistribute learned
   !
   address-family evpn
      neighbor PG-SPINE activate
   !
   vrf TENANT2
      rd 10.1.0.3:20002
      route-target import evpn 2:20002
      route-target export evpn 2:20002
      redistribute connected
!
router ospf 1
   router-id 10.0.1.3
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   network 0.0.0.0/0 area 0.0.0.0
   max-lsa 12000
!
end
</code></pre>
</details>

<details>
<summary>Конфигурация Leaf4</summary>
<pre><code>
! Command: show running-config
! device: leaf4 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname leaf4
!
spanning-tree mode mstp
!
vlan 10,20
!
vrf instance TENANT2
!
interface Ethernet1
   no switchport
   ip address 10.1.4.2/30
   ip ospf network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.2.4.2/30
   ip ospf network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.3.4.2/30
   ip ospf network point-to-point
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
   switchport access vlan 10
!
interface Ethernet7
   switchport access vlan 20
!
interface Ethernet8
   no switchport
   vrf TENANT2
   ip address 172.16.1.1/24
!
interface Loopback0
   ip address 10.0.0.4/32
!
interface Loopback1
   ip address 10.0.1.4/32
!
interface Management1
!
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010
   vxlan vlan 20 vni 10020
   vxlan vrf TENANT2 vni 20002
!
ip routing
ip routing vrf TENANT2
!
router bgp 65004
   router-id 10.0.1.4
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-SPINE peer group
   neighbor PG-SPINE remote-as 65000
   neighbor PG-SPINE update-source 10.0.1.4
   neighbor PG-SPINE bfd
   neighbor PG-SPINE ebgp-multihop
   neighbor PG-SPINE send-community extended
   neighbor 10.0.2.1 peer group PG-SPINE
   neighbor 10.0.2.2 peer group PG-SPINE
   neighbor 10.0.2.3 peer group PG-SPINE
   !
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned
   !
   vlan 20
      rd auto
      route-target both 20:10020
      redistribute learned
   !
   address-family evpn
      neighbor PG-SPINE activate
   !
   vrf TENANT2
      rd 10.1.0.4:20002
      route-target import evpn 2:20002
      route-target export evpn 2:20002
      neighbor 172.16.1.254 remote-as 64999
      redistribute connected
!
router ospf 1
   router-id 10.0.1.4
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   network 0.0.0.0/0 area 0.0.0.0
   max-lsa 12000
!
end
</code></pre>
</details>

<details>
<summary>Конфигурация Spine1</summary>
<pre><code>
! Command: show running-config
! device: spine1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname spine1
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.1.1.1/30
   ip ospf network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.1.2.1/30
   ip ospf network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.1.3.1/30
   ip ospf network point-to-point
!
interface Ethernet4
   no switchport
   ip address 10.1.4.1/30
   ip ospf network point-to-point
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   ip address 10.0.2.1/32
!
interface Management1
!
ip routing
!
router bgp 65000
   router-id 10.0.2.1
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-LEAF peer group
   neighbor PG-LEAF next-hop-unchanged
   neighbor PG-LEAF update-source 10.0.2.1
   neighbor PG-LEAF bfd
   neighbor PG-LEAF ebgp-multihop
   neighbor PG-LEAF send-community extended
   neighbor 10.0.1.1 peer group PG-LEAF
   neighbor 10.0.1.1 remote-as 65001
   neighbor 10.0.1.2 peer group PG-LEAF
   neighbor 10.0.1.2 remote-as 65002
   neighbor 10.0.1.3 peer group PG-LEAF
   neighbor 10.0.1.3 remote-as 65003
   neighbor 10.0.1.4 peer group PG-LEAF
   neighbor 10.0.1.4 remote-as 65004
   !
   address-family evpn
      neighbor PG-LEAF activate
!
router ospf 1
   router-id 10.0.2.1
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   no passive-interface Ethernet4
   network 0.0.0.0/0 area 0.0.0.0
   max-lsa 12000
!
end
</code></pre>
</details>

<details>
<summary>Конфигурация Spine2</summary>
<pre><code>
! Command: show running-config
! device: spine2 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname spine2
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.2.1.1/30
   ip ospf network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.2.2.1/30
   ip ospf network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.2.3.1/30
   ip ospf network point-to-point
!
interface Ethernet4
   no switchport
   ip address 10.2.4.1/30
   ip ospf network point-to-point
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   ip address 10.0.2.2/32
!
interface Management1
!
ip routing
!
router bgp 65000
   router-id 10.0.2.2
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-LEAF peer group
   neighbor PG-LEAF next-hop-unchanged
   neighbor PG-LEAF update-source 10.0.2.2
   neighbor PG-LEAF bfd
   neighbor PG-LEAF ebgp-multihop
   neighbor PG-LEAF send-community extended
   neighbor 10.0.1.1 peer group PG-LEAF
   neighbor 10.0.1.1 remote-as 65001
   neighbor 10.0.1.2 peer group PG-LEAF
   neighbor 10.0.1.2 remote-as 65002
   neighbor 10.0.1.3 peer group PG-LEAF
   neighbor 10.0.1.3 remote-as 65003
   neighbor 10.0.1.4 peer group PG-LEAF
   neighbor 10.0.1.4 remote-as 65004
   !
   address-family evpn
      neighbor PG-LEAF activate
!
router ospf 1
   router-id 10.0.2.2
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   no passive-interface Ethernet4
   network 0.0.0.0/0 area 0.0.0.0
   max-lsa 12000
!
end
</code></pre>
</details>

<details>
<summary>Конфигурация Spine3</summary>
<pre><code>
! Command: show running-config
! device: spine3 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname spine3
!
spanning-tree mode mstp
!
interface Ethernet1
   no switchport
   ip address 10.3.1.1/30
   ip ospf network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.3.2.1/30
   ip ospf network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.3.3.1/30
   ip ospf network point-to-point
!
interface Ethernet4
   no switchport
   ip address 10.3.4.1/30
   ip ospf network point-to-point
!
interface Ethernet5
!
interface Ethernet6
!
interface Ethernet7
!
interface Ethernet8
!
interface Loopback1
   ip address 10.0.2.3/32
!
interface Management1
!
ip routing
!
router bgp 65000
   router-id 10.0.2.3
   timers bgp 3 9
   maximum-paths 8
   neighbor PG-LEAF peer group
   neighbor PG-LEAF next-hop-unchanged
   neighbor PG-LEAF update-source 10.0.2.3
   neighbor PG-LEAF bfd
   neighbor PG-LEAF ebgp-multihop
   neighbor PG-LEAF send-community extended
   neighbor 10.0.1.1 peer group PG-LEAF
   neighbor 10.0.1.1 remote-as 65001
   neighbor 10.0.1.2 peer group PG-LEAF
   neighbor 10.0.1.2 remote-as 65002
   neighbor 10.0.1.3 peer group PG-LEAF
   neighbor 10.0.1.3 remote-as 65003
   neighbor 10.0.1.4 peer group PG-LEAF
   neighbor 10.0.1.4 remote-as 65004
   !
   address-family evpn
      neighbor PG-LEAF activate
!
router ospf 1
   router-id 10.0.2.3
   bfd default
   passive-interface default
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   no passive-interface Ethernet4
   network 0.0.0.0/0 area 0.0.0.0
   max-lsa 12000
!
end
</code></pre>
</details>


<details>
<summary>Конфигурация Extrouter</summary>
<pre><code>
! Command: show running-config
! device: extrouter (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model multi-agent
!
hostname extrouter
!
spanning-tree mode mstp
!
vrf instance TENANT1
!
interface Ethernet1
!
interface Ethernet2
!
interface Ethernet3
!
interface Ethernet4
!
interface Ethernet5
!
interface Ethernet6
   no switchport
   vrf TENANT1
   ip address 192.168.10.254/24
!
interface Ethernet7
   no switchport
   vrf TENANT1
   ip address 192.168.20.254/24
!
interface Ethernet8
   no switchport
   ip address 172.16.1.254/24
!
interface Loopback8
   vrf TENANT1
   ip address 8.8.8.8/32
!
interface Loopback9
   ip address 9.9.9.9/32
!
interface Management1
!
ip routing
ip routing vrf TENANT1
!
ip route 0.0.0.0/0 Null0
!
router bgp 64999
   router-id 9.9.9.9
   timers bgp 3 9
   neighbor FABRIC peer group
   neighbor FABRIC bfd
   neighbor FABRIC ebgp-multihop
   neighbor 172.16.1.1 peer group FABRIC
   neighbor 172.16.1.1 remote-as 65004
   neighbor 172.16.1.1 default-originate
   redistribute connected
!
end
</code></pre>
</details>

<details>
<summary>Конфигурация MHClient1</summary>
<pre><code>
! Command: show running-config
! device: mhclient1 (vEOS-lab, EOS-4.29.2F)
!
! boot system flash:/vEOS-lab.swi
!
no aaa root
!
transceiver qsfp default-mode 4x10G
!
service routing protocols model ribd
!
hostname mhclient1
!
spanning-tree mode mstp
!
interface Port-Channel12
   no switchport
   ip address 192.168.10.12/24
!
interface Ethernet1
   channel-group 12 mode active
!
interface Ethernet2
   channel-group 12 mode active
!
interface Ethernet3
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
interface Management1
!
no ip routing
!
ip route 0.0.0.0/0 192.168.10.254
!
end
</code></pre>
</details>

## Проверка тестового стенда

Проверка Underlay

```
leaf1#sh ip route

VRF: default
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

 C        10.0.0.1/32 is directly connected, Loopback0
 O        10.0.0.2/32 [110/30] via 10.1.1.1, Ethernet1
                               via 10.2.1.1, Ethernet2
                               via 10.3.1.1, Ethernet3
 O        10.0.0.3/32 [110/30] via 10.1.1.1, Ethernet1
                               via 10.2.1.1, Ethernet2
                               via 10.3.1.1, Ethernet3
 O        10.0.0.4/32 [110/30] via 10.1.1.1, Ethernet1
                               via 10.2.1.1, Ethernet2
                               via 10.3.1.1, Ethernet3
 C        10.0.1.1/32 is directly connected, Loopback1
 O        10.0.1.2/32 [110/30] via 10.1.1.1, Ethernet1
                               via 10.2.1.1, Ethernet2
                               via 10.3.1.1, Ethernet3
 O        10.0.1.3/32 [110/30] via 10.1.1.1, Ethernet1
                               via 10.2.1.1, Ethernet2
                               via 10.3.1.1, Ethernet3
 O        10.0.1.4/32 [110/30] via 10.1.1.1, Ethernet1
                               via 10.2.1.1, Ethernet2
                               via 10.3.1.1, Ethernet3
 O        10.0.2.1/32 [110/20] via 10.1.1.1, Ethernet1
 O        10.0.2.2/32 [110/20] via 10.2.1.1, Ethernet2
 O        10.0.2.3/32 [110/20] via 10.3.1.1, Ethernet3
 C        10.1.1.0/30 is directly connected, Ethernet1
 O        10.1.2.0/30 [110/20] via 10.1.1.1, Ethernet1
 O        10.1.3.0/30 [110/20] via 10.1.1.1, Ethernet1
 O        10.1.4.0/30 [110/20] via 10.1.1.1, Ethernet1
 C        10.2.1.0/30 is directly connected, Ethernet2
 O        10.2.2.0/30 [110/20] via 10.2.1.1, Ethernet2
 O        10.2.3.0/30 [110/20] via 10.2.1.1, Ethernet2
 O        10.2.4.0/30 [110/20] via 10.2.1.1, Ethernet2
 C        10.3.1.0/30 is directly connected, Ethernet3
 O        10.3.2.0/30 [110/20] via 10.3.1.1, Ethernet3
 O        10.3.3.0/30 [110/20] via 10.3.1.1, Ethernet3
 O        10.3.4.0/30 [110/20] via 10.3.1.1, Ethernet3
```

```
leaf1#sh ip ospf neighbor
Neighbor ID     Instance VRF      Pri State                  Dead Time   Address         Interface
10.0.2.1        1        default  0   FULL                   00:00:29    10.1.1.1        Ethernet1
10.0.2.3        1        default  0   FULL                   00:00:33    10.3.1.1        Ethernet3
10.0.2.2        1        default  0   FULL                   00:00:34    10.2.1.1        Ethernet2
```

```
leaf1#sh bfd peers
VRF name: default
-----------------
DstAddr      MyDisc    YourDisc  Interface/Transport      Type          LastUp
-------- ----------- ----------- -------------------- --------- ---------------
10.0.2.1 1458752227  2180990018                   NA  multihop  07/20/24 17:53
10.0.2.2 2063007030   543346504                   NA  multihop  07/20/24 17:53
10.0.2.3 1582795974  1141633842                   NA  multihop  07/20/24 17:54
10.1.1.1  881530894  3836181073        Ethernet1(15)    normal  07/21/24 07:30
10.2.1.1 3095622364  4240107659        Ethernet2(16)    normal  07/21/24 07:31
10.3.1.1 3740663026  2344695893        Ethernet3(17)    normal  07/21/24 07:31

         LastDown            LastDiag    State
-------------------- ------------------- -----
   07/20/24 17:53       No Diagnostic       Up
   07/20/24 17:53       No Diagnostic       Up
               NA       No Diagnostic       Up
               NA       No Diagnostic       Up
               NA       No Diagnostic       Up
               NA       No Diagnostic       Up
```


Проверка Overlay

```
leaf1#sh bgp evpn summary
BGP summary information for VRF default
Router identifier 10.0.1.1, local AS number 65001
Neighbor Status Codes: m - Under maintenance
  Neighbor V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.0.2.1 4 65000          36054     36090    0    0    1d00h Estab   15     15
  10.0.2.2 4 65000          36065     36069    0    0    1d00h Estab   15     15
  10.0.2.3 4 65000          35212     35186    0    0    1d00h Estab   15     15
```

```
leaf1#sh bgp evpn
BGP routing table information for VRF default
Router identifier 10.0.1.1, local AS number 65001
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending BGP convergence
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.0.1.1:10 auto-discovery 0 0000:0000:0000:0012:0012
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.1.2:10 auto-discovery 0 0000:0000:0000:0012:0012
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.1.2:10 auto-discovery 0 0000:0000:0000:0012:0012
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.1.2:10 auto-discovery 0 0000:0000:0000:0012:0012
                                 10.0.0.2              -       100     0       65000 65002 i
 * >      RD: 10.0.0.1:1 auto-discovery 0000:0000:0000:0012:0012
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.2:1 auto-discovery 0000:0000:0000:0012:0012
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.0.2:1 auto-discovery 0000:0000:0000:0012:0012
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.0.2:1 auto-discovery 0000:0000:0000:0012:0012
                                 10.0.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.0.1.4:10 mac-ip 5000.0088.fe27
                                 10.0.0.4              -       100     0       65000 65004 i
 *  ec    RD: 10.0.1.4:10 mac-ip 5000.0088.fe27
                                 10.0.0.4              -       100     0       65000 65004 i
 *  ec    RD: 10.0.1.4:10 mac-ip 5000.0088.fe27
                                 10.0.0.4              -       100     0       65000 65004 i
 * >      RD: 10.0.1.1:10 mac-ip 5000.00af.d3f6
                                 -                     -       -       0       i
 * >      RD: 10.0.1.1:10 imet 10.0.0.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.1.2:10 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.1.2:10 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.1.2:10 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.0.1.2:50 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.1.2:50 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.1.2:50 imet 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.0.1.3:20 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.0.1.3:20 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.0.1.3:20 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.0.1.3:60 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.0.1.3:60 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.0.1.3:60 imet 10.0.0.3
                                 10.0.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.0.1.4:10 imet 10.0.0.4
                                 10.0.0.4              -       100     0       65000 65004 i
 *  ec    RD: 10.0.1.4:10 imet 10.0.0.4
                                 10.0.0.4              -       100     0       65000 65004 i
 *  ec    RD: 10.0.1.4:10 imet 10.0.0.4
                                 10.0.0.4              -       100     0       65000 65004 i
 * >Ec    RD: 10.0.1.4:20 imet 10.0.0.4
                                 10.0.0.4              -       100     0       65000 65004 i
 *  ec    RD: 10.0.1.4:20 imet 10.0.0.4
                                 10.0.0.4              -       100     0       65000 65004 i
 *  ec    RD: 10.0.1.4:20 imet 10.0.0.4
                                 10.0.0.4              -       100     0       65000 65004 i
 * >      RD: 10.0.0.1:1 ethernet-segment 0000:0000:0000:0012:0012 10.0.0.1
                                 -                     -       -       0       i
 * >Ec    RD: 10.0.0.2:1 ethernet-segment 0000:0000:0000:0012:0012 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.0.2:1 ethernet-segment 0000:0000:0000:0012:0012 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.0.0.2:1 ethernet-segment 0000:0000:0000:0012:0012 10.0.0.2
                                 10.0.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.4:20002 ip-prefix 0.0.0.0/0
                                 10.0.0.4              -       100     0       65000 65004 64999 ?
 *  ec    RD: 10.1.0.4:20002 ip-prefix 0.0.0.0/0
                                 10.0.0.4              -       100     0       65000 65004 64999 ?
 *  ec    RD: 10.1.0.4:20002 ip-prefix 0.0.0.0/0
                                 10.0.0.4              -       100     0       65000 65004 64999 ?
 * >Ec    RD: 10.1.0.4:20002 ip-prefix 9.9.9.9/32
                                 10.0.0.4              -       100     0       65000 65004 64999 i
 *  ec    RD: 10.1.0.4:20002 ip-prefix 9.9.9.9/32
                                 10.0.0.4              -       100     0       65000 65004 64999 i
 *  ec    RD: 10.1.0.4:20002 ip-prefix 9.9.9.9/32
                                 10.0.0.4              -       100     0       65000 65004 64999 i
 * >Ec    RD: 10.1.0.2:20002 ip-prefix 10.99.50.0/24
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:20002 ip-prefix 10.99.50.0/24
                                 10.0.0.2              -       100     0       65000 65002 i
 *  ec    RD: 10.1.0.2:20002 ip-prefix 10.99.50.0/24
                                 10.0.0.2              -       100     0       65000 65002 i
 * >Ec    RD: 10.1.0.3:20002 ip-prefix 10.99.60.0/24
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:20002 ip-prefix 10.99.60.0/24
                                 10.0.0.3              -       100     0       65000 65003 i
 *  ec    RD: 10.1.0.3:20002 ip-prefix 10.99.60.0/24
                                 10.0.0.3              -       100     0       65000 65003 i
 * >Ec    RD: 10.1.0.4:20002 ip-prefix 172.16.1.0/24
                                 10.0.0.4              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.4:20002 ip-prefix 172.16.1.0/24
                                 10.0.0.4              -       100     0       65000 65004 i
 *  ec    RD: 10.1.0.4:20002 ip-prefix 172.16.1.0/24
                                 10.0.0.4              -       100     0       65000 65004 i
```

```
leaf1#sh mac address-table
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  10    0050.7966.6806    DYNAMIC     Et7        1       0:00:06 ago
  10    5000.0088.fe27    DYNAMIC     Vx1        1       2:48:43 ago
  10    5000.00af.d3f6    DYNAMIC     Po12       1       2:45:28 ago
4094    5000.0003.3766    DYNAMIC     Vx1        1       23:51:58 ago
4094    5000.0015.f4e8    DYNAMIC     Vx1        1       23:34:04 ago
4094    5000.00f6.ad37    DYNAMIC     Vx1        1       23:08:31 ago

Total Mac Addresses for this criterion: 5

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
```

```
leaf2#sh ip ro vrf TENANT2

VRF: TENANT2
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

Gateway of last resort:
 B E      0.0.0.0/0 [200/0] via VTEP 10.0.0.4 VNI 20002 router-mac 50:00:00:f6:ad:37 local-interface Vxlan1

 B E      9.9.9.9/32 [200/0] via VTEP 10.0.0.4 VNI 20002 router-mac 50:00:00:f6:ad:37 local-interface Vxlan1
 C        10.99.50.0/24 is directly connected, Vlan50
 B E      10.99.60.0/24 [200/0] via VTEP 10.0.0.3 VNI 20002 router-mac 50:00:00:15:f4:e8 local-interface Vxlan1
 B E      172.16.1.0/24 [200/0] via VTEP 10.0.0.4 VNI 20002 router-mac 50:00:00:f6:ad:37 local-interface Vxlan1
```

Проверка с помощью конечных хостов

TENANT1
- Проверка ping через extrouter в другую подсеть
Client1 (192.168.10.1)
ping 192.168.20.3

```
VPCS> ping 192.168.20.3

84 bytes from 192.168.20.3 icmp_seq=1 ttl=63 time=68.461 ms
84 bytes from 192.168.20.3 icmp_seq=2 ttl=63 time=90.018 ms
84 bytes from 192.168.20.3 icmp_seq=3 ttl=63 time=71.003 ms
84 bytes from 192.168.20.3 icmp_seq=4 ttl=63 time=83.174 ms
84 bytes from 192.168.20.3 icmp_seq=5 ttl=63 time=66.711 ms
```

- Проверка ping через extrouter в "Интернет" (8.8.8.8)
Client12 (192.168.10.12)
ping 8.8.8.8

```
mhclient1#ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 72(100) bytes of data.
80 bytes from 8.8.8.8: icmp_seq=1 ttl=64 time=67.6 ms
80 bytes from 8.8.8.8: icmp_seq=2 ttl=64 time=76.4 ms
80 bytes from 8.8.8.8: icmp_seq=3 ttl=64 time=70.0 ms
80 bytes from 8.8.8.8: icmp_seq=4 ttl=64 time=64.3 ms
80 bytes from 8.8.8.8: icmp_seq=5 ttl=64 time=59.4 ms

--- 8.8.8.8 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 47ms
rtt min/avg/max/mdev = 59.409/67.585/76.454/5.695 ms, pipe 5, ipg/ewma 11.759/67.205 ms
```

TENANT2

- Проверка ping через extrouter в другую подсеть
Client2 (10.99.50.2)
ping 10.99.60.4

```
VPCS> ping 10.99.60.4

84 bytes from 10.99.60.4 icmp_seq=1 ttl=62 time=141.502 ms
84 bytes from 10.99.60.4 icmp_seq=2 ttl=62 time=40.325 ms
84 bytes from 10.99.60.4 icmp_seq=3 ttl=62 time=35.809 ms
84 bytes from 10.99.60.4 icmp_seq=4 ttl=62 time=53.096 ms
84 bytes from 10.99.60.4 icmp_seq=5 ttl=62 time=31.612 ms
```

- Проверка ping через extrouter во внешние сети (9.9.9.9)
Client4 (10.99.60.4)
ping 9.9.9.9

```
VPCS> ping 9.9.9.9

84 bytes from 9.9.9.9 icmp_seq=1 ttl=62 time=73.947 ms
84 bytes from 9.9.9.9 icmp_seq=2 ttl=62 time=32.509 ms
84 bytes from 9.9.9.9 icmp_seq=3 ttl=62 time=32.574 ms
84 bytes from 9.9.9.9 icmp_seq=4 ttl=62 time=32.172 ms
84 bytes from 9.9.9.9 icmp_seq=5 ttl=62 time=42.430 ms
```

## Выводы

В ходе выполнения проекта цели и задачи выполнены.
Достигнута цель:
- создана новая сеть в виде тестового стенда.
Решены задачи:
- выбор архитектуры сети
- разработка архитектуры сети
- разработка дизайна underlay сети
- разработка дизайна overlay сети
- разработка дизайна адресного пространства и идентификаторов
