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

