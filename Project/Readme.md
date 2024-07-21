# Проектная работа на тему "Создание новой сети на базе технологий EVPN VXLAN"

## Введение

### Цель проекта
Цель проекта выбрана в соответствии с вероятным сценарием при ответе на внутренний запрос бизнеса компании на разработку дизайна сети под предполагаемую новую локацию ЦОД.
Текущая локация насыщена по многим параметрам, которые указывают на необходимость разработки новой сети в новой локации

Стратегические прогнозы бизнеса предполагают, что сеть должна поддерживать масштабирование до 50 стоек оборудования на ближайшие 5 лет (в каждом может быть до 2х Top-of-Rack свичей для поддержки резервирования), т.е. до 100 Leaf свичей в сумме.

Соответственно цель проекта - выбор и разработка дизайна новой сети (Greenfield) с поддержкой масштабирования и т.д.

### Задачи проекта
На основе цели проекта соответственно должны быть выполнены задачи
- выбор архитектуры сети
- разработка архитектуры сети
- разработка дизайна underlay сети
- разработка дизайна overlay сети
- разработка дизайна адресного пространства и идентификаторов
- разработка архитектуры и дизайна сети для разворачивания в новом ЦОД

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
Используемый Data Plane - VXLAN
Используемый Control Plane - BGP EVPN

В рамках проекта задание выполняется для сети Clos размером 2 spine 4 leaf (но поддерживает расширение до бОльших размеров)

#### Разработка дизайна underlay сети и адресных пространств
Остановимся подробнее на
1. underlay interface connection планировании (int plan)
2. underlay ip address and network underlay планировании (ip.addr-net plan)
3. overlay evpn vxlan instances - vni, macvrf, vrf, rd, rt, esi, virtual-mac

--- разработка дизайна underlay сети и адресного пространства
--- разработка дизайна overlay сети и пространства идентификаторов
--- разработка дизайна адресного пространства и идентификаторов

выбран OSPF с P2P линками и BFD

===

--- разработка архитектуры сети топологии Clos на основе EVPN VXLAN для нового ЦОД

Архитектура сети разрабатывалась на основе Design Guide выбранного вендора и лучших практик на текущий момент - Arista
соответственно архитектура сети с топологией Clos предполагает
- набор коммутаторов для подключения конечного оборудования - Leaf
- набор коммутаторов для эффективного промежуточного подключения между собой коммутаторов Leaf - Spine
- стандартные правила подключения для таких фабрик - Leaf может подключаться физически только к Spine, Spine не подключаются 

физически между собой, Leaf не подключаются физически между собой (кроме некоторых случаев парных подключений при M-LAG, но в 

данной работе такое не используется)
- 1 большой ip домен маршрутизации для обеспечения связности между устройствами
- работа VXLAN так как инкапсуляция полагается на IP-UDP протоколы, полагается на этот большой домен маршрутизации
- используются Lo интерфейсы для терминации VXLAN туннелей




===

## 3. interface connection планирование

1.1. Нумерация подключения p2p линков для соединения Leaf-Spine, для удобства и для соответствия ip.addr-net plan формируется по 

следующим правилам:
- номер интерфейса Spine (EthN) указывает на номер Leaf<N> к которому подключен. N начинается с 1.
- номер интерфейса Leaf (EthM) указывает на номер Spine<M> к которому подключен. M начинается с 1.
т.е. номера интерфейсов выбраны соответственно удаленному оборудованию на второй стороне подключения интерфейса.

Например,
- Spine1 Eth1 - Leaf1 Eth1
- Spine1 Eth3 - Leaf3 Eth1
- Spine3 Eth2 - Leaf2 Eth3

1.2. Нумерация Lo интерфейсов:
- номер интерфейса указывает на номер выбранного диапазона DC-SERVICE-NUMBER.
Так как и сети и Lo доступны нумерации с 0, мы будем использовать нумерацию с 0 для удобства.

1.3. Нумерация Хостовых интерфейсов
подключения хостов будут выполняться на интерфейсы с конца

## 2. ip address and network underlay планирование (ip.addr-net plan) для Underlay

2.1. Общие принципы

Выбор диапазонов производится для DC и будет учитывать:
- расширение сети с учётом нескольких возможных DC, пока 1
- расширение сети с учётом нескольких возможных SuperSpine, пока 0
- резервы расширения сети в каждом DC будут заложены изначально
- возможность суммаризации адресного пространства всего DC

Используемые маски:
- сети для линков P2P - /30 (так привычнее по текущим сетям для бизнеса)
- сети для лупбэков - /32

Всего в рамках Underlay и данного ДЗ нужно 4 пространства нумерации, которые будем активно использовать
- Lo интерфейс и адрес для Routing на уровне Spine
- P2P интерфейс и адрес для Routing на уровнях Spine-Leaf
- Lo интерфейс и адрес для VXLAN на уровне Leaf (делаем немного заранее, предположительно нужно будет в следующих ДЗ)
- Lo интерфейс и адрес для Routing на уровне Leaf

### 2.2. Общая формула адресного пространства

10.DC-SERVICE-NUMBER.SPINE-SERVICE-NUMBER.SUBNET-NUMBER
DC-SERVICE-NUMBER - Range адресов для масштабирования DC и сервисов. Номер позволит точно узнать какой DC, сервис и т.д..
SPINE-SERVICE-NUMBER - Range адресов для масштабирования и выбора идентификаторов в рамках DC-SERVICE-NUMBER. Номер позволит 

точно узнать какой Spine, сервис и т.д..
SUBNET-NUMBER - Range адресов для масштабирования и выбора идентификаторов в рамках SPINE-SERVICE-NUMBER. Номер уже не будет 

показывать числовым значением ничего, просто перебор сетей по порядку.

---
10.DC-SERVICE-NUMBER.x.x (x - dont care в данном случае)
DC-SERVICE-NUMBER разбит на диапазоны для суммаризации по 1 DC - /13, т.е. диапазоны 0-7(DC1), 8-15(DC2) и т.д.
- DC-SERVICE-NUMBER = 0 для лупбэков
- DC-SERVICE-NUMBER = 1 для P2P сетей - интерфейсов от Spine1, используемых для роутинга к Leaf. Leaf and Spine.
- DC-SERVICE-NUMBER = 2 для P2P сетей - интерфейсов от Spine2, используемых для роутинга к Leaf. Leaf and Spine.
- DC-SERVICE-NUMBER = 3 для P2P сетей - интерфейсов от Spine3, используемых для роутинга к Leaf. Leaf and Spine.
- DC-SERVICE-NUMBER = 4-7 для резервных назначений (например ещё для одних лупбэков, или расширения маршрутизации). В данном ДЗ 

не использованы.

---
10.x.SPINE-SERVICE-NUMBER.x (x - dont care в данном случае)
SPINE-SERVICE-NUMBER не разбивается на диапазоны
SPINE-SERVICE-NUMBER = 0-255 - позволяет перебирать для разных назначений номера
Для P2P сетей
- SPINE-SERVICE-NUMBER = 1 для лупбэков и линков P2P для Spine1-Leafx
- SPINE-SERVICE-NUMBER = 2 для лупбэков и линков P2P для Spine2-Leafx
Для Lo сетей
- SPINE-SERVICE-NUMBER = 0 для лупбэков Leaf, используемых для полезных сервисов, связанных с терминацией VXLAN туннелей. Только 

Leaf.
- SPINE-SERVICE-NUMBER = 1 для лупбэков Leaf, используемых для полезных сервисов, связанных с терминацией VXLAN туннелей. Только 

Leaf.
- SPINE-SERVICE-NUMBER = 2 для лупбэков Spine

---
10.x.x.SUBNET-NUMBER (x - dont care в данном случае)
SUBNET-NUMBER не разбивается на диапазоны
- SUBNET-NUMBER = 0-255 - позволяет перебирать для разных назначений номера Spine, Leaf

---
Примеры
DC1-Spine1-Leaf1 P2P линк - 10.2.1.0/24
DC1-Spine2-Leaf3 P2P линк - 10.2.2.8/24
DC1-Leaf2 Lo0 - 10.0.0.2
DC1-Leaf3 Lo0 - 10.0.0.3

Логика нумерации для MACVRF не требуется - так как в Arista MACVRF создаётся по номеру vlan
Например 
vlan 10 = MACVRF10

Логика выбора ethernet-segment будет следующая:
0000:0000:0000:<leaf-pair-number-at-the-end>:port-channel-number
Например
0000:0000:0000:0012:0012

Логика выбора LACP system-id будет следующая:
0000.leaf-pair-number.port-channel-number
Например
0000.0012.0012

Логика выбора rd, rt будет следующая:
rd - <source.leaf.ip.addr>:<vni>
Например
10.1.0.3:10010

rt - <tenant.number>:<vni>
Например
10.1.0.3:10010



=== Последовательность настройки

Приведены примеры настройки для каждого типа устройства или раздела.
Полные конфигурации доступны в разделе Итоговые конфигурации ниже.

В реальной эксплуатации в зависимости от типа оборудования могут быть добавлены конфигурации
System Profile (TCAM Resource)
System (Clock, NTP и т.д. )
Mgmt Plane (Log, SNMP и т.д.)

--- Предварительная и общая конфигурация Leaf, Spine

конфигурация имени устройства
hostname leaf1

конфигурация актуального Route Table Manager
service routing protocols model multi-agent
reload

--- Конфигурация интерфейсов для Leaf

конфигурация интерфейсов в сторону Spine, включаем режим L3 порта, настраиваем адресацию
interface Ethernet1
   no switchport
   ip address 10.2.1.2/30
interface Ethernet2
   no switchport
   ip address 10.2.2.2/30
interface Ethernet3
   no switchport
   ip address 10.2.3.2/30

конфигурация Loopback интерфейса для задач терминации eBGP EVPN сессий
конфигурация Loopback интерфейса для задач терминации VXLAN туннелей
interface Loopback0
   ip address 10.0.0.1/32
interface Loopback1
   ip address 10.1.0.1/32

--- Конфигурация интерфейсов для Spine

interface Ethernet1
   no switchport
   ip address 10.2.1.1/30
   ip ospf network point-to-point
!
interface Ethernet2
   no switchport
   ip address 10.2.1.5/30
   ip ospf network point-to-point
!
interface Ethernet3
   no switchport
   ip address 10.2.1.9/30
   ip ospf network point-to-point

--- Конфигурация маршрутизации Underlay для Leaf и Spine (на примере Leaf1)

конфигурация общая для работы Underlay маршрутизации
ip routing

конфигурация OSPF основная для работы Underlay маршрутизации
router ospf 1

конфигурация router-id, будет соответствовать Lo1 интерфейсу
   router-id 10.0.1.1

конфигурация passive-interface по умолчанию
   passive-interface default

конфигурация passive-interface
   no passive-interface Ethernet1
   no passive-interface Ethernet2
   no passive-interface Ethernet3
   network 0.0.0.0/0 area 0.0.0.0
   bfd default
   max-lsa 12000

конфигурация OSPF на интерфейсах, включаем режим point-to-point
interface Ethernet1
   ip ospf network point-to-point
interface Ethernet2
   ip ospf network point-to-point
interface Ethernet3
   ip ospf network point-to-point

--- Конфигурация Leaf для конечных подключений Overlay (на примере Leaf1)

конфигурация vlan
vlan 10

конфигурация интерфейса в сторону конечного хоста
interface Ethernet7
   switchport access vlan 10

конфигурация VXLAN интерфейса c назначением L2VNI
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 10 vni 10010

также настроим MACVRF для vlan 10
router bgp 65001
   vlan 10
      rd auto
      route-target both 10:10010
      redistribute learned

=== Конфигурация Leaf для конечных подключений с локальным L3 GW Overlay (на примере Leaf2)

конфигурация vlan
vlan 10

конфигурация интерфейса в сторону конечного хоста
interface Ethernet7
   switchport access vlan 50

конфигурация VRF
vrf instance TENANT2

конфигурация интерфейса VRF
interface Vlan50
   vrf TENANT2
   ip address virtual 10.99.50.254/24

конфигурация маршрутизации для VRF
ip routing vrf TENANT2

конфигурация VXLAN интерфейса c назначением L2VNI, L3VNI
interface Vxlan1
   vxlan source-interface Loopback0
   vxlan udp-port 4789
   vxlan vlan 50 vni 10050
   vxlan vrf TENANT2 vni 20002

также настроим MACVRF для vlan 50
router bgp 65002
   vlan 50
      rd auto
      route-target both 50:10050
      redistribute learned

также настроим VRF для TENANT2
router bgp 65002
   vrf TENANT2
      rd 10.1.0.2:20002
      route-target import evpn 2:20002
      route-target export evpn 2:20002
      redistribute connected

=== Конфигурация Leaf для Overlay (на примере Leaf1)

конфигурация BGP, настроим router-id, таймеры, максимальное количество путей, BFD
также настроим eBGP multihop
также настроим пересылку community
также используем Peer Group для упрощения настройки

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

также настроим активацию Peer Group для EVPN AFI:SAFI
   address-family evpn
      neighbor PG-SPINE activate

=== Конфигурация Spine для Overlay (на примере Spine1)

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

=== Конфигурация Leaf для настройки ES-LAG (на примере Leaf1)

конфигурация интерфейса для работы в PortChannel
interface Ethernet8
   channel-group 12 mode active

конфигурация PortChannel для работы в ES-LAG
interface Port-Channel12
   switchport access vlan 10
   !
   evpn ethernet-segment
      identifier 0000:0000:0000:0012:0012
      route-target import 00:00:00:00:12:12
   lacp system-id 0000.0012.0012

=== Конфигурация со стороны клиента с одним подключением (на примере Client1)

конфигурация интерфейса для настройки ip.addr и Default Gateway
ip 192.168.10.1/24 192.168.10.254

=== Конфигурация со стороны клиента Multi-Home Client (на примере MHClient1)

конфигурация интерфейсов для работы в PortChannel
interface Ethernet1
   channel-group 12 mode active
!
interface Ethernet2
   channel-group 12 mode active

конфигурация интерфейса для настройки ip.addr
interface Port-Channel12
   no switchport
   ip address 192.168.10.12/24

конфигурация Default Gateway
ip route 0.0.0.0/0 192.168.10.254

=== Итоговые конфигурации

=== Проверка

--- Выводы

--- Нужно показать
Client (Orphan)
MHClient
Border Leaf
IRB

=== Проверка работы

=== Выводы

В ходе выполнения проекта цели и задачи выполнены.
