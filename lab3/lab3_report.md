# Лабораторная работа №3  
# «Эмуляция распределённой корпоративной сети связи, настройка OSPF и MPLS, организация первого EoMPLS»

## Цель работы

Изучить принципы построения распределённой корпоративной сети связи, а также освоить настройку протокола динамической маршрутизации OSPF, технологий MPLS/LDP и механизма EoMPLS (VPLS) в среде ContainerLab.

---

## Описание работы

В рамках лабораторной работы требовалось развернуть в ContainerLab распределённую IP/MPLS-сеть связи компании **RogaIKopita Games**, объединяющую несколько офисов.  

Особенностью задания являлась необходимость подключения удалённого офиса в Нью-Йорке к общей магистральной сети, а также организация канала **EoMPLS** между сервером **SGI Prism** и рабочей станцией инженеров в Санкт-Петербурге.

В результате выполнения лабораторной работы необходимо было:

- развернуть топологию сети в ContainerLab;
- настроить IP-адресацию на всех магистральных соединениях;
- настроить OSPF для обмена маршрутной информацией;
- настроить MPLS и LDP;
- настроить EoMPLS/VPLS между узлами `R01.NY` и `R01.SPB`;
- назначить IP-адреса конечным узлам `SGI Prism` и `PC1`;
- проверить связность как на уровне соседних маршрутизаторов, так и между конечными узлами.

---

## Схема сети

-Топология лабораторной работы включает следующие узлы:
-R01.NY — пограничный маршрутизатор сети, к которому подключён узел SGI Prism.
-R01.LND — транзитный маршрутизатор, обеспечивающий передачу трафика внутри магистрали.
-R01.HKI — магистральный маршрутизатор, соединяющий несколько направлений сети.
-R01.SPB — пограничный маршрутизатор сети, к которому подключён узел PC1.
-R01.MSK — транзитный маршрутизатор магистральной сети.
-R01.LBN — транзитный маршрутизатор, обеспечивающий резервный путь передачи данных.
-SGI Prism — конечный узел, используемый для проверки связи через сеть.
-PC1 — конечный узел, используемый для проверки работы EoMPLS-соединения.


Связи между устройствами были организованы следующим образом:

- `SGI Prism:eth1 <-> R01.NY:eth1`
- `R01.NY:eth2 <-> R01.LND:eth1`
- `R01.LND:eth2 <-> R01.HKI:eth1`
- `R01.HKI:eth2 <-> R01.SPB:eth1`
- `R01.NY:eth3 <-> R01.LBN:eth1`
- `R01.LBN:eth2 <-> R01.MSK:eth1`
- `R01.MSK:eth2 <-> R01.SPB:eth2`
- `R01.LBN:eth3 <-> R01.HKI:eth3`
- `R01.SPB:eth3 <-> PC1:eth1`

Сюда необходимо вставить рисунок схемы сети, выполненный в draw.io или Visio.

---

## Файл топологии ContainerLab

Для развертывания лабораторной работы использовался следующий файл `lab3.clab.yaml`:

```yaml
name: lab3

mgmt:
  network: clab-mgmt
  ipv4-subnet: 172.20.20.0/24

topology:
  nodes:

    R01.NY:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.10

    R01.LND:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.11

    R01.HKI:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.12

    R01.SPB:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.13

    R01.MSK:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.14

    R01.LBN:
      kind: vr-mikrotik_ros
      image: vrnetlab/mikrotik_routeros:6.47.9
      mgmt-ipv4: 172.20.20.15

    SGI-Prism:
      kind: linux
      image: alpine:latest

    PC1:
      kind: linux
      image: alpine:latest

  links:
    - endpoints: ["SGI-Prism:eth1", "R01.NY:eth1"]

    - endpoints: ["R01.NY:eth2", "R01.LND:eth1"]
    - endpoints: ["R01.LND:eth2", "R01.HKI:eth1"]
    - endpoints: ["R01.HKI:eth2", "R01.SPB:eth1"]

    - endpoints: ["R01.NY:eth3", "R01.LBN:eth1"]
    - endpoints: ["R01.LBN:eth2", "R01.MSK:eth1"]
    - endpoints: ["R01.MSK:eth2", "R01.SPB:eth2"]

    - endpoints: ["R01.LBN:eth3", "R01.HKI:eth3"]

    - endpoints: ["R01.SPB:eth3", "PC1:eth1"]
````

## Развёртывание лаборатории

Для запуска лабораторной работы использовались следующие команды:

```bash
sudo containerlab destroy -t lab3.clab.yaml --cleanup
sudo containerlab deploy -t lab3.clab.yaml
sudo containerlab inspect -t lab3.clab.yaml
```

После развертывания все linux-контейнеры перешли в состояние `running`, а маршрутизаторы MikroTik RouterOS — в состояние `running (healthy)`.

Сюда рекомендуется вставить скриншот вывода команды:

```bash
sudo containerlab inspect -t lab3.clab.yaml
```

---

## Особенность образа RouterOS в ContainerLab

В процессе выполнения лабораторной работы была подтверждена особенность образа `vrnetlab/mikrotik_routeros:6.47.9`.

Несмотря на то, что в файле `.yaml` использовались интерфейсы `eth1`, `eth2`, `eth3` и т.д., внутри RouterOS реальные интерфейсы сдвигались на один номер из-за использования служебного management-интерфейса.

Например, для маршрутизатора `R01.NY`:

* `eth1` из ContainerLab соответствовал `ether2` в RouterOS;
* `eth2` соответствовал `ether3`;
* `eth3` соответствовал `ether4`.

Таким образом, при настройке всех маршрутизаторов требовалось предварительно проверять реальные интерфейсы командой:

```rsc
/interface ethernet print
```

---

## Принятая схема адресации

### Loopback-адреса маршрутизаторов

* `R01.NY` — `1.1.1.1/32`
* `R01.LND` — `2.2.2.2/32`
* `R01.HKI` — `3.3.3.3/32`
* `R01.SPB` — `4.4.4.4/32`
* `R01.MSK` — `5.5.5.5/32`
* `R01.LBN` — `6.6.6.6/32`

### Межмаршрутизаторные сети

* `R01.NY — R01.LND` → `10.0.12.0/30`
* `R01.LND — R01.HKI` → `10.0.23.0/30`
* `R01.HKI — R01.SPB` → `10.0.34.0/30`
* `R01.NY — R01.LBN` → `10.0.16.0/30`
* `R01.LBN — R01.MSK` → `10.0.65.0/30`
* `R01.MSK — R01.SPB` → `10.0.54.0/30`
* `R01.LBN — R01.HKI` → `10.0.63.0/30`

### Адреса конечных узлов в сегменте EoMPLS

* `SGI Prism` — `192.168.100.10/24`
* `PC1` — `192.168.100.20/24`

---

## Базовая настройка IP-адресации

### Конфигурация `R01.NY`

```rsc
/system identity set name=R01.NY
/user set admin password=StrongPass123

/interface bridge
add name=lo
add name=br-eompls

/interface ethernet
set ether1 comment="mgmt"
set ether2 comment="to SGI-Prism"
set ether3 comment="to R01.LND"
set ether4 comment="to R01.LBN"

/ip address
add address=1.1.1.1/32 interface=lo
add address=10.0.12.1/30 interface=ether3
add address=10.0.16.1/30 interface=ether4

/interface bridge port
add bridge=br-eompls interface=ether2
```

### Конфигурация `R01.LND`

```rsc
/system identity set name=R01.LND
/user set admin password=StrongPass123

/interface bridge
add name=lo

/interface ethernet
set ether1 comment="mgmt"
set ether2 comment="to R01.NY"
set ether3 comment="to R01.HKI"

/ip address
add address=2.2.2.2/32 interface=lo
add address=10.0.12.2/30 interface=ether2
add address=10.0.23.1/30 interface=ether3
```

### Конфигурация `R01.HKI`

```rsc
/system identity set name=R01.HKI
/user set admin password=StrongPass123

/interface bridge
add name=lo

/interface ethernet
set ether1 comment="mgmt"
set ether2 comment="to R01.LND"
set ether3 comment="to R01.SPB"
set ether4 comment="to R01.LBN"

/ip address
add address=3.3.3.3/32 interface=lo
add address=10.0.23.2/30 interface=ether2
add address=10.0.34.1/30 interface=ether3
add address=10.0.63.2/30 interface=ether4
```

### Конфигурация `R01.SPB`

```rsc
/system identity set name=R01.SPB
/user set admin password=StrongPass123

/interface bridge
add name=lo
add name=br-eompls

/interface ethernet
set ether1 comment="mgmt"
set ether2 comment="to R01.HKI"
set ether3 comment="to R01.MSK"
set ether4 comment="to PC1"

/ip address
add address=4.4.4.4/32 interface=lo
add address=10.0.34.2/30 interface=ether2
add address=10.0.54.2/30 interface=ether3

/interface bridge port
add bridge=br-eompls interface=ether4
```

### Конфигурация `R01.MSK`

```rsc
/system identity set name=R01.MSK
/user set admin password=StrongPass123

/interface bridge
add name=lo

/interface ethernet
set ether1 comment="mgmt"
set ether2 comment="to R01.LBN"
set ether3 comment="to R01.SPB"

/ip address
add address=5.5.5.5/32 interface=lo
add address=10.0.65.2/30 interface=ether2
add address=10.0.54.1/30 interface=ether3
```

### Конфигурация `R01.LBN`

```rsc
/system identity set name=R01.LBN
/user set admin password=StrongPass123

/interface bridge
add name=lo

/interface ethernet
set ether1 comment="mgmt"
set ether2 comment="to R01.NY"
set ether3 comment="to R01.MSK"
set ether4 comment="to R01.HKI"

/ip address
add address=6.6.6.6/32 interface=lo
add address=10.0.16.2/30 interface=ether2
add address=10.0.65.1/30 interface=ether3
add address=10.0.63.1/30 interface=ether4
```

---

## Проверка локальной связности

После назначения IP-адресов была проверена связность между соседними маршрутизаторами.

Примеры проверок:

### `R01.NY`

```rsc
ping 10.0.12.2
ping 10.0.16.2
```

### `R01.LND`

```rsc
ping 10.0.12.1
ping 10.0.23.2
```

### `R01.HKI`

```rsc
ping 10.0.23.1
ping 10.0.34.2
ping 10.0.63.1
```

### `R01.SPB`

```rsc
ping 10.0.34.1
ping 10.0.54.1
```

### `R01.MSK`

```rsc
ping 10.0.65.1
ping 10.0.54.2
```

### `R01.LBN`

```rsc
ping 10.0.16.1
ping 10.0.65.2
ping 10.0.63.2
```

Во всех случаях пакеты передавались успешно, потерь не наблюдалось.

Сюда рекомендуется вставить скриншоты результатов ping между соседними маршрутизаторами.

---

## Настройка OSPF

Для обеспечения динамического обмена маршрутной информацией на всех маршрутизаторах был настроен OSPF.

### `R01.NY`

```rsc
/routing ospf instance
set [ find default=yes ] router-id=1.1.1.1

/routing ospf network
add network=1.1.1.1/32 area=backbone
add network=10.0.12.0/30 area=backbone
add network=10.0.16.0/30 area=backbone
```

### `R01.LND`

```rsc
/routing ospf instance
set [ find default=yes ] router-id=2.2.2.2

/routing ospf network
add network=2.2.2.2/32 area=backbone
add network=10.0.12.0/30 area=backbone
add network=10.0.23.0/30 area=backbone
```

### `R01.HKI`

```rsc
/routing ospf instance
set [ find default=yes ] router-id=3.3.3.3

/routing ospf network
add network=3.3.3.3/32 area=backbone
add network=10.0.23.0/30 area=backbone
add network=10.0.34.0/30 area=backbone
add network=10.0.63.0/30 area=backbone
```

### `R01.SPB`

```rsc
/routing ospf instance
set [ find default=yes ] router-id=4.4.4.4

/routing ospf network
add network=4.4.4.4/32 area=backbone
add network=10.0.34.0/30 area=backbone
add network=10.0.54.0/30 area=backbone
```

### `R01.MSK`

```rsc
/routing ospf instance
set [ find default=yes ] router-id=5.5.5.5

/routing ospf network
add network=5.5.5.5/32 area=backbone
add network=10.0.54.0/30 area=backbone
add network=10.0.65.0/30 area=backbone
```

### `R01.LBN`

```rsc
/routing ospf instance
set [ find default=yes ] router-id=6.6.6.6

/routing ospf network
add network=6.6.6.6/32 area=backbone
add network=10.0.16.0/30 area=backbone
add network=10.0.63.0/30 area=backbone
add network=10.0.65.0/30 area=backbone
```

После настройки OSPF соседство между маршрутизаторами успешно установилось.

Примеры проверки:

```rsc
/routing ospf neighbor print
/ip route print
```

Сюда рекомендуется вставить скриншоты:

* `/routing ospf neighbor print`
* `/ip route print`

---

## Настройка MPLS и LDP

После успешного запуска OSPF была выполнена настройка MPLS и LDP на магистральных интерфейсах.

### Пример настройки `R01.NY`

```rsc
/mpls interface
add interface=ether3
add interface=ether4

/mpls ldp
set enabled=yes lsr-id=1.1.1.1 transport-address=1.1.1.1

/mpls ldp interface
add interface=ether3
add interface=ether4
```

Аналогичная настройка была выполнена на остальных маршрутизаторах с использованием собственных loopback-адресов в качестве `lsr-id` и `transport-address`.

В результате LDP-соседство было успешно установлено по всей сети.

Примеры проверки:

```rsc
/mpls ldp neighbor print
/mpls forwarding-table print
```

По результатам проверки:

* `R01.NY` установил LDP-соседство с `R01.LND` и `R01.LBN`;
* `R01.LND` — с `R01.NY` и `R01.HKI`;
* `R01.HKI` — с `R01.LND`, `R01.SPB` и `R01.LBN`;
* `R01.SPB` — с `R01.HKI` и `R01.MSK`;
* `R01.MSK` — с `R01.SPB` и `R01.LBN`;
* `R01.LBN` — с `R01.NY`, `R01.HKI` и `R01.MSK`.

Сюда рекомендуется вставить скриншоты:

* `/mpls ldp neighbor print`
* `/mpls forwarding-table print`

---

## Настройка EoMPLS (VPLS)

Для организации L2-соединения между `SGI Prism` и `PC1` был настроен VPLS между маршрутизаторами `R01.NY` и `R01.SPB`.

### На `R01.NY`

```rsc
/interface vpls
add name=vpls-to-spb remote-peer=4.4.4.4 vpls-id=100:1 disabled=no

/interface bridge port
add bridge=br-eompls interface=vpls-to-spb
```

### На `R01.SPB`

```rsc
/interface vpls
add name=vpls-to-ny remote-peer=1.1.1.1 vpls-id=100:1 disabled=no

/interface bridge port
add bridge=br-eompls interface=vpls-to-ny
```

В результате VPLS-интерфейсы на обоих PE-маршрутизаторах перешли в состояние `running`.

Проверка выполнялась командами:

```rsc
/interface vpls print detail
/interface bridge port print
```

Сюда рекомендуется вставить скриншоты:

* `/interface vpls print detail`
* `/interface bridge port print`

---

## Настройка IP-адресов на конечных хостах

После организации EoMPLS были назначены IP-адреса конечным linux-контейнерам.

### `SGI Prism`

```bash
docker exec -it clab-lab3-SGI-Prism sh
ip addr add 192.168.100.10/24 dev eth1
ip link set eth1 up
```

### `PC1`

```bash
docker exec -it clab-lab3-PC1 sh
ip addr add 192.168.100.20/24 dev eth1
ip link set eth1 up
```

---

## Финальная проверка связности

После завершения настройки была проверена связность между конечными узлами.

### Проверка с `SGI Prism`

```bash
ping 192.168.100.20
```

Результат:

```text
PING 192.168.100.20 (192.168.100.20): 56 data bytes
64 bytes from 192.168.100.20: seq=0 ttl=64 time=10.211 ms
64 bytes from 192.168.100.20: seq=1 ttl=64 time=13.397 ms
64 bytes from 192.168.100.20: seq=2 ttl=64 time=3.788 ms
64 bytes from 192.168.100.20: seq=3 ttl=64 time=16.704 ms
64 bytes from 192.168.100.20: seq=4 ttl=64 time=15.544 ms
64 bytes from 192.168.100.20: seq=5 ttl=64 time=15.616 ms

--- 192.168.100.20 ping statistics ---
6 packets transmitted, 6 packets received, 0% packet loss
round-trip min/avg/max = 3.788/12.543/16.704 ms
```

### Проверка с `PC1`

```bash
ping 192.168.100.10
```

Результат:

```text
PING 192.168.100.10 (192.168.100.10): 56 data bytes
64 bytes from 192.168.100.10: seq=0 ttl=64 time=4.443 ms
64 bytes from 192.168.100.10: seq=1 ttl=64 time=3.360 ms
64 bytes from 192.168.100.10: seq=2 ttl=64 time=9.221 ms
64 bytes from 192.168.100.10: seq=3 ttl=64 time=14.627 ms
64 bytes from 192.168.100.10: seq=4 ttl=64 time=15.585 ms

--- 192.168.100.10 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max = 3.360/9.447/15.585 ms
```

Успешные двусторонние ping подтверждают, что контейнеры `SGI Prism` и `PC1` были объединены в единый Ethernet-сегмент поверх MPLS-сети.

---

## Вывод

В ходе выполнения лабораторной работы была успешно развернута распределённая корпоративная сеть связи в ContainerLab. На маршрутизаторах MikroTik была выполнена настройка IP-адресации, протокола динамической маршрутизации OSPF, MPLS и LDP. Между удалёнными узлами `R01.NY` и `R01.SPB` был организован EoMPLS (VPLS), что позволило объединить контейнеры `SGI Prism` и `PC1` в единый L2-сегмент.

Корректность выполнения лабораторной работы подтверждена:

* успешной локальной связностью между соседними маршрутизаторами;
* установленным OSPF-соседством;
* наличием MPLS/LDP-соседства между магистральными маршрутизаторами;
* работоспособностью VPLS-туннеля;
* успешными двусторонними ping между `SGI Prism` и `PC1`.

Таким образом, цель лабораторной работы была полностью достигнута.
