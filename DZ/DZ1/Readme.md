# Домашнее задание 1 (Урок3)

Для выполнения проектирования адресного пространства в данном ДЗ остановимся подробнее
на
1. interface connection планировании (int plan)
2. ip address and network underlay планировании (ip.addr-net plan)

Задание выполняется для сети клоза размером
2 spine
3 leaf
но поддерживает расширение до бОльших размеров

===

## 1. interface connection планирование

1.1. Нумерация подключения p2p линков, для удобства и для соответствия ip.addr-net plan формируется по следующим правилам:
- номер интерфейса Spine (EthN) указывает на номер Leaf<N> к которому подключен. N начинается с 1.
- номер интерфейса Leaf (EthM) указывает на номер Spine<M> к которому подключен. M начинается с 1.
т.е. номера интерфейсов выбраны соответственно удаленному оборудованию на второй стороне подключения интерфейса.

Например,
- Spine1 Eth1 - Leaf1 Eth1
- Spine1 Eth3 - Leaf3 Eth1
- Spine4 Eth2 - Leaf2 Eth4

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
- сети для линков P2P - /30 (привычнее)
- сети для лупбэков - /32

Всего в рамках Underlay и данного ДЗ нужно 4 пространства нумерации, которые будем активно использовать
- Lo интерфейс и адрес для Routing на уровне Spine
- P2P интерфейс и адрес для Routing на уровнях Spine-Leaf
- Lo интерфейс и адрес для VXLAN на уровне Leaf (делаем немного заранее, предположительно нужно будет в следующих ДЗ)
- Lo интерфейс и адрес для Routing на уровне Leaf

### 2.2. Общая формула адресного пространства

10.DC-SERVICE-NUMBER.SPINE-SERVICE-NUMBER.SUBNET-NUMBER
DC-SERVICE-NUMBER - Range адресов для масштабирования DC и сервисов. Номер позволит точно узнать какой DC, сервис и т.д..
SPINE-SERVICE-NUMBER - Range адресов для масштабирования и выбора идентификаторов в рамках DC-SERVICE-NUMBER. Номер позволит точно узнать какой Spine, сервис и т.д..
SUBNET-NUMBER - Range адресов для масштабирования и выбора идентификаторов в рамках SPINE-SERVICE-NUMBER. Номер уже не будет показывать числовым значением ничего, просто перебор сетей по порядку.

---
10.DC-SERVICE-NUMBER.x.x (x - dont care в данном случае)
DC-SERVICE-NUMBER разбит на диапазоны для суммаризации по 1 DC - /13, т.е. диапазоны 0-7(DC1), 8-15(DC2) и т.д.
- DC-SERVICE-NUMBER = 0 для Lo0 лупбэков, используемых для полезных сервисов, связанных с терминацией VXLAN туннелей. Только Leaf.
- DC-SERVICE-NUMBER = 1 для Lo1 лупбэков, используемых для роутинга. Leaf and Spine.
- DC-SERVICE-NUMBER = 2 для P2P сетей физический интерфейсов между S-L, используемых для роутинга. Leaf and Spine.
- DC-SERVICE-NUMBER = 3-7 для резервных назначений (например ещё для одних лупбэков, или расширения маршрутизации). В данном ДЗ не использованы.

---
10.x.SPINE-SERVICE-NUMBER.x (x - dont care в данном случае)
SPINE-SERVICE-NUMBER не разбивается на диапазоны
SPINE-SERVICE-NUMBER = 0-255 - позволяет перебирать для разных назначений номера
Для P2P сетей
- SPINE-SERVICE-NUMBER = 1 для лупбэков и линков P2P для Spine1-Leafx
- SPINE-SERVICE-NUMBER = 2 для лупбэков и линков P2P для Spine2-Leafx
Для Lo сетей
- SPINE-SERVICE-NUMBER = 0 для лупбэков Leaf
- SPINE-SERVICE-NUMBER = 1 для лупбэков Spine

---
10.x.x.SUBNET-NUMBER (x - dont care в данном случае)
SUBNET-NUMBER не разбивается на диапазоны
- SUBNET-NUMBER = 0-255 - позволяет перебирать для разных назначений номера

---
Примеры
DC1-Spine1-Leaf1 P2P линк - 10.2.1.0/24
DC1-Spine2-Leaf3 P2P линк - 10.2.2.8/24
DC1-Leaf2 Lo0 - 10.0.0.
DC1-Leaf3 Lo0 - 10.0.0.

## 3. Конечные адреса и интерфейсы в рамках данной фабрики представлены на рисунке и в конфигурации

![](pictures/Topo.png)

## 4. Конфигурация

=== Spine1

```
interface Ethernet1
   no switchport
   ip address 10.2.1.1/30
!
interface Ethernet2
   no switchport
   ip address 10.2.1.5/30
!
interface Ethernet3
   no switchport
   ip address 10.2.1.9/30
!
interface Lo1
   ip address 10.1.1.1/32
```

=== Spine2

```
interface Ethernet1
   no switchport
   ip address 10.2.2.1/30
!
interface Ethernet2
   no switchport
   ip address 10.2.2.5/30
!
interface Ethernet3
   no switchport
   ip address 10.2.2.9/30
!
interface Lo1
   ip address 10.1.1.2/32
```

=== Leaf1

```
interface Ethernet1
   no switchport
   ip address 10.2.1.2/30
!
interface Ethernet2
   no switchport
   ip address 10.2.2.2/30
!
interface Lo0
   ip address 10.0.0.1/32
!
interface Lo1
   ip address 10.1.0.1/32
```

=== Leaf2

```
interface Ethernet1
   no switchport
   ip address 10.2.1.6/30
!
interface Ethernet2
   no switchport
   ip address 10.2.2.6/30
!
interface Lo0
   ip address 10.0.0.2/32
!
interface Lo1
   ip address 10.1.0.2/32
```

=== Leaf3

```
interface Ethernet1
   no switchport
   ip address 10.2.1.10/30
!
interface Ethernet2
   no switchport
   ip address 10.2.2.10/30
!
interface Lo0
   ip address 10.0.0.3/32
!
interface Lo1
   ip address 10.1.0.3/32
```

## 5. Пример проверки работы ip P2P связности

~~~
spine2#sh ip int br
                                                                        Address
Interface        IP Address       Status      Protocol           MTU    Owner
---------------- ---------------- ----------- ------------- ----------- -------
Ethernet1        10.2.2.1/30      up          up                1500
Ethernet2        10.2.2.5/30      up          up                1500
Ethernet3        10.2.2.9/30      up          up                1500
Loopback1        10.1.1.2/32      up          up               65535
Management1      unassigned       up          up                1500

spine2#ping 10.2.2.2
PING 10.2.2.2 (10.2.2.2) 72(100) bytes of data.
80 bytes from 10.2.2.2: icmp_seq=1 ttl=64 time=44.3 ms
80 bytes from 10.2.2.2: icmp_seq=2 ttl=64 time=45.5 ms
80 bytes from 10.2.2.2: icmp_seq=3 ttl=64 time=39.3 ms
80 bytes from 10.2.2.2: icmp_seq=4 ttl=64 time=35.5 ms
80 bytes from 10.2.2.2: icmp_seq=5 ttl=64 time=5.82 ms

--- 10.2.2.2 ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 81ms
rtt min/avg/max/mdev = 5.828/34.149/45.582/14.607 ms, pipe 4, ipg/ewma 20.293/38             .215 ms
spine2#
~~~
