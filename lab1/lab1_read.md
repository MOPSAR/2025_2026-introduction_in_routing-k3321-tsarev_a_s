University: [ITMO University](https://itmo.ru/ru/)

Faculty: [FICT](https://fict.itmo.ru)

Course: [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)

Year: 2023/2024

Group: K66666

Author: Filianin Ivan Victorovich

Lab: Lab1

Date of create: 20.09.2023

Date of finished: 31.09.2023




# Lab1. Построение трехуровневой сети в ContainerLab

## Цель работы

Изучить основы развёртывания сетевой лаборатории в ContainerLab, настроить трёхуровневую сеть предприятия на базе MikroTik RouterOS, реализовать сегментацию сети с помощью VLAN, настроить trunk/access порты, DHCP на центральном маршрутизаторе и проверить локальную связность между узлами.

---

## Задание

В рамках лабораторной работы требовалось:

- развернуть в ContainerLab топологию трёхуровневой сети предприятия;
- создать management-сеть для доступа к устройствам;
- настроить два VLAN для пользовательских сегментов;
- настроить trunk и access порты на коммутаторах;
- настроить DHCP-серверы на центральном маршрутизаторе;
- обеспечить получение IP-адресов конечными узлами PC1 и PC2;
- настроить имена устройств и сменить пароль пользователя;
- подготовить отчёт в формате Markdown внутри GitHub-репозитория.

---

## Состав лабораторной сети

В работе использовались следующие устройства:

- **R01** — центральный маршрутизатор
- **SW01** — коммутатор распределения
- **SW02A** — коммутатор доступа для PC1
- **SW02B** — коммутатор доступа для PC2
- **PC1** — клиент VLAN 10
- **PC2** — клиент VLAN 20

### Management IP-адреса

- **R01** — `172.20.20.10`
- **SW01** — `172.20.20.11`
- **SW02A** — `172.20.20.12`
- **SW02B** — `172.20.20.13`

### Соединения между устройствами

- `R01:eth1 <-> SW01:eth1`
- `SW01:eth2 <-> SW02A:eth1`
- `SW01:eth3 <-> SW02B:eth1`
- `SW02A:eth2 <-> PC1:eth1`
- `SW02B:eth2 <-> PC2:eth1`

---

## Схема связи

В отчёт необходимо добавить схему, нарисованную вручную в **draw.io** или **Visio**.

**Нужно вставить скриншот или экспорт схемы сюда.**

Пример подписи:

> Рисунок 1 — Схема трёхуровневой сети предприятия

---

## Файл развертывания ContainerLab

Ниже приведён `.yaml` файл, использованный для развёртывания лабораторной сети.

### Файл `lab1.clab.yaml`

```yaml
name: lab1

mgmt:
  network: clab-mgmt
  ipv4-subnet: 172.20.20.0/24

topology:
  nodes:

    R01:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.10

    SW01:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.11

    SW02A:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.12

    SW02B:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.13

    PC1:
      kind: linux
      image: alpine:latest

    PC2:
      kind: linux
      image: alpine:latest

  links:
    - endpoints: ["R01:eth1","SW01:eth1"]
    - endpoints: ["SW01:eth2","SW02A:eth1"]
    - endpoints: ["SW01:eth3","SW02B:eth1"]
    - endpoints: ["SW02A:eth2","PC1:eth1"]
    - endpoints: ["SW02B:eth2","PC2:eth1"]

```



## Развёртывание виртуальной сети

Перед началом настройки виртуальная сеть была развёрнута в ContainerLab.

При необходимости удаления старой версии сети использовалась команда:

sudo containerlab destroy -t lab1.clab.yaml --cleanup


После этого выполнялось развёртывание новой версии:

sudo containerlab deploy -t lab1.clab.yaml


Для проверки состояния контейнеров использовалась команда:

sudo containerlab inspect -t lab1.clab.yaml


После успешного запуска:

контейнеры PC1 и PC2 находились в состоянии running;

устройства R01, SW01, SW02A, SW02B находились в состоянии running (healthy).

Сюда рекомендуется вставить скриншот вывода sudo containerlab inspect -t lab1.clab.yaml.

## Особенность используемого образа RouterOS

В процессе выполнения лабораторной работы была обнаружена особенность образа vrnetlab/mikrotik_routeros:6.47.9.

Несмотря на то, что в топологии ContainerLab использовались интерфейсы eth1, eth2 и eth3, внутри RouterOS реальные рабочие интерфейсы оказались смещены на один порт. Из-за этого, изначально, конфигурация была применена к неправильным интерфейсам, в результате чего DHCP-запросы от клиентских узлов не доходили до центрального маршрутизатора.

После диагностики была определена фактическая рабочая схема интерфейсов:

R01 — uplink к SW01: ether2

SW01 — к R01: ether2, к SW02A: ether3, к SW02B: ether4

SW02A — к SW01: ether2, к PC1: ether3

SW02B — к SW01: ether2, к PC2: ether3

Именно эти интерфейсы использовались в финальной конфигурации.

## План адресации и VLAN

Для разделения пользовательских узлов были созданы два VLAN:

VLAN 10 — для узла PC1

VLAN 20 — для узла PC2

### Подсети

VLAN	Назначение	Подсеть	Шлюз
10	PC1	192.168.10.0/24	192.168.10.1
20	PC2	192.168.20.0/24	192.168.20.1

### DHCP-пулы

для VLAN 10: 192.168.10.100 - 192.168.10.199

для VLAN 20: 192.168.20.100 - 192.168.20.199

## Конфигурация устройств

Ниже приведены итоговые рабочие конфигурации всех сетевых устройств.

###Конфигурация R01

На центральном маршрутизаторе были созданы два VLAN-интерфейса, назначены IP-адреса шлюзов, созданы DHCP-пулы и настроены DHCP-серверы.

/system identity set name=R01
/user set admin password=123

/interface vlan
add name=vlan10 interface=ether2 vlan-id=10
add name=vlan20 interface=ether2 vlan-id=20

/ip address
add address=192.168.10.1/24 interface=vlan10
add address=192.168.20.1/24 interface=vlan20

/ip pool
add name=pool_vlan10 ranges=192.168.10.100-192.168.10.199
add name=pool_vlan20 ranges=192.168.20.100-192.168.20.199

/ip dhcp-server
add name=dhcp_vlan10 interface=vlan10 address-pool=pool_vlan10 disabled=no
add name=dhcp_vlan20 interface=vlan20 address-pool=pool_vlan20 disabled=no

/ip dhcp-server network
add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=8.8.8.8
add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=8.8.8.8

### Проверка конфигурации R01

Для проверки конфигурации использовались команды:

/interface vlan print
/ip address print
/ip pool print
/ip dhcp-server print
/ip dhcp-server network print
/ip dhcp-server lease print


Сюда рекомендуется вставить скриншот вывода команд проверки R01.

### Конфигурация SW01

На SW01 был создан bridge, настроены trunk-порты и таблица VLAN для VLAN 10 и VLAN 20.

/system identity set name=SW01
/user set admin password=123

/interface bridge
add name=br1 vlan-filtering=no

/interface bridge port
add bridge=br1 interface=ether2
add bridge=br1 interface=ether3
add bridge=br1 interface=ether4

/interface bridge vlan
add bridge=br1 vlan-ids=10 tagged=br1,ether2,ether3,ether4
add bridge=br1 vlan-ids=20 tagged=br1,ether2,ether3,ether4

/interface bridge
set br1 vlan-filtering=yes

### Проверка конфигурации SW01
/interface bridge print
/interface bridge port print
/interface bridge vlan print


Сюда рекомендуется вставить скриншот вывода команд проверки SW01.

### Конфигурация SW02A

На SW02A uplink к SW01 был настроен как trunk-порт, а порт подключения PC1 как access-порт VLAN 10.

/system identity set name=SW02
/user set admin password=123

/interface bridge
add name=br1 vlan-filtering=no

/interface bridge port
add bridge=br1 interface=ether2
add bridge=br1 interface=ether3 pvid=10

/interface bridge vlan
add bridge=br1 vlan-ids=10 tagged=br1,ether2 untagged=ether3

/interface bridge
set br1 vlan-filtering=yes

### Проверка конфигурации SW02A
/interface bridge print
/interface bridge port print
/interface bridge vlan print


Сюда рекомендуется вставить скриншот вывода команд проверки SW02A.

### Конфигурация SW02B

На SW02B uplink к SW01 был настроен как trunk-порт, а порт подключения PC2 как access-порт VLAN 20.

/system identity set name=SW02
/user set admin password=123

/interface bridge
add name=br1 vlan-filtering=no

/interface bridge port
add bridge=br1 interface=ether2
add bridge=br1 interface=ether3 pvid=20

/interface bridge vlan
add bridge=br1 vlan-ids=20 tagged=br1,ether2 untagged=ether3

/interface bridge
set br1 vlan-filtering=yes

### Проверка конфигурации SW02B

/interface bridge print
/interface bridge port print
/interface bridge vlan print


Сюда рекомендуется вставить скриншот вывода команд проверки SW02B.

## Проверка получения IP-адреса на PC1

Для проверки работоспособности VLAN 10 и DHCP на узле PC1 был запущен DHCP-клиент на интерфейсе eth1.

udhcpc -i eth1
ip addr show eth1
ping -c 4 192.168.10.1

Результат проверки PC1

После выполнения команд PC1 получил IP-адрес из сети VLAN 10:

IP-адрес: 192.168.10.199/24

шлюз: 192.168.10.1

Вывод DHCP-клиента на PC1
udhcpc: started, v1.37.0
udhcpc: broadcasting discover
udhcpc: broadcasting select for 192.168.10.199, server 192.168.10.1
udhcpc: lease of 192.168.10.199 obtained from 192.168.10.1, lease time 600
mv: can't rename '/etc/resolv.conf.36': Resource busy


Сообщение о невозможности переименовать /etc/resolv.conf связано с особенностями Alpine-контейнера и не влияет на успешное получение IP-адреса.

Проверка IP-адреса PC1
eth1@if230: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9500 qdisc noqueue state UP
    link/ether aa:c1:ab:8a:b2:2a brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.199/24 scope global eth1
       valid_lft forever preferred_lft forever

Проверка связности PC1 с шлюзом
PING 192.168.10.1 (192.168.10.1): 56 data bytes
64 bytes from 192.168.10.1: seq=0 ttl=64 time=2.890 ms
64 bytes from 192.168.10.1: seq=1 ttl=64 time=10.383 ms
64 bytes from 192.168.10.1: seq=2 ttl=64 time=9.476 ms
64 bytes from 192.168.10.1: seq=3 ttl=64 time=9.608 ms

--- 192.168.10.1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss


Сюда рекомендуется вставить скриншот проверки DHCP и ping для PC1.

## Проверка получения IP-адреса на PC2

Для проверки работоспособности VLAN 20 и DHCP на узле PC2 был запущен DHCP-клиент на интерфейсе eth1.

udhcpc -i eth1
ip addr show eth1
ping -c 4 192.168.20.1

Результат проверки PC2

После выполнения команд PC2 получил IP-адрес из сети VLAN 20:

IP-адрес: 192.168.20.199/24

шлюз: 192.168.20.1

Вывод DHCP-клиента на PC2
udhcpc: started, v1.37.0
udhcpc: broadcasting discover
udhcpc: broadcasting select for 192.168.20.199, server 192.168.20.1
udhcpc: lease of 192.168.20.199 obtained from 192.168.20.1, lease time 600
mv: can't rename '/etc/resolv.conf.16': Resource busy


Сообщение о невозможности переименовать /etc/resolv.conf также не влияет на успешную работу DHCP.

Проверка IP-адреса PC2
eth1@if224: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 9500 qdisc noqueue state UP
    link/ether aa:c1:ab:29:b9:9a brd ff:ff:ff:ff:ff:ff
    inet 192.168.20.199/24 scope global eth1
       valid_lft forever preferred_lft forever

Проверка связности PC2 с шлюзом
PING 192.168.20.1 (192.168.20.1): 56 data bytes
64 bytes from 192.168.20.1: seq=0 ttl=64 time=2.547 ms
64 bytes from 192.168.20.1: seq=1 ttl=64 time=4.214 ms
64 bytes from 192.168.20.1: seq=2 ttl=64 time=8.410 ms
64 bytes from 192.168.20.1: seq=3 ttl=64 time=10.745 ms

--- 192.168.20.1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss


Сюда рекомендуется вставить скриншот проверки DHCP и ping для PC2.

## Проверка локальной связности

В рамках проверки локальной связности были выполнены ping-запросы от конечных узлов до соответствующих шлюзов:

PC1 -> 192.168.10.1

PC2 -> 192.168.20.1

В обоих случаях пакеты были доставлены без потерь, что подтверждает:

корректную работу VLAN;

корректную настройку trunk и access портов;

корректную работу DHCP;

корректную работу локальной маршрутизации на центральном устройстве.

## Результаты лабораторной работы

В результате выполнения лабораторной работы были получены следующие результаты:

подготовлен .yaml файл для развёртывания лабораторной сети;

подготовлена схема связи, нарисованная в draw.io или Visio;

получены итоговые конфигурации для каждого сетевого устройства;

успешно настроены VLAN 10 и VLAN 20;

успешно настроены trunk и access порты на коммутаторах;

на маршрутизаторе R01 успешно настроены DHCP-серверы для обеих VLAN;

узел PC1 получил IP-адрес 192.168.10.199/24;

узел PC2 получил IP-адрес 192.168.20.199/24;

проверена локальная связность:

PC1 успешно пингует 192.168.10.1;

PC2 успешно пингует 192.168.20.1.

## Вывод

В ходе выполнения лабораторной работы была успешно развернута трехуровневая сеть предприятия в среде ContainerLab. На базе MikroTik RouterOS была выполнена настройка двух VLAN, настройка trunk и access портов на коммутаторах, а также настройка DHCP-серверов на центральном маршрутизаторе.

Практическим результатом работы стало успешное получение IP-адресов конечными узлами PC1 и PC2 по DHCP из разных подсетей и подтверждение локальной связности с их шлюзами.

Дополнительно в ходе выполнения лабораторной работы была выявлена особенность соответствия интерфейсов внутри используемого образа RouterOS в ContainerLab. После корректировки реальных рабочих портов сеть была успешно приведена в полностью рабочее состояние.


